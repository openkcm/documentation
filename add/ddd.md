# ADD-301: OpenKCM Krypton Unified Cryptographic Topology Engine

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** |2026-03-07 | Architecture Decision Document (ADD) |

## Context and Problem Statement
OpenKCM must support highly diverse enterprise deployment models—ranging from fully self-contained monolithic deployments to highly distributed, zero-latency edge architectures. Hardcoding the relationships between cryptographic key levels (Lx) fundamentally breaks this flexibility.

Furthermore, the engine must support complex compliance and zero-trust security requirements natively:
* **Double Key Encryption (Cascade Wrapping):** Ensuring external KMS providers only ever sign pre-encrypted ciphertexts.
* **Separation of Compute and State:** Allowing edge nodes to process high-speed KMIP requests while deferring cryptographic authority to a stateless central core.
* **Resilient Bootstrapping:** Requiring zero-touch auto-unsealing via distributed Multi-KMS Quorums that survive hyperscaler outages.

The architectural challenge is creating a single, universal execution engine and configuration schema capable of dynamically morphing into any of these roles purely via declarative YAML, without conditional spaghetti code.

## Architectural Decision
The OpenKCM Krypton engine will utilize a dynamic, map-based YAML configuration schema that treats the entire cryptographic hierarchy as a **Directed Acyclic Graph (DAG)**.



This schema strictly decouples four core architectural concepts:
* **Tiers:** The arbitrary cryptographic levels (e.g., MasterKey, IVK, L2, L3, L4).
* **Compute (Provider):** The execution environment that handles entropy generation, wrapping, and unwrapping (`memory_resident`, `local_derived`, `plugins`, `remote_upstream`).
* **State (Storage):** The persistence layer defined via gRPC plugins (`hashicorp_vault_kv`, `postgresql_kv_store`, `redis_kv_store`).
* **Interfaces:** The network exposure for a specific tier (`http`, `grpc_upstream`, `kmip`).

## Core Technical Pillars

### Topological Sort and The Dependency Graph
The engine does not hardcode what "L2" or "L3" means. Instead, every tier defines its parent dependency via the `wrapped_by` directive. At startup, the engine performs a topological sort to validate the graph (ensuring no circular dependencies) and initializes the providers bottom-up.

### Cascade Wrapping (Double Key Encryption)
To achieve zero-trust external integration, the `wrapped_by` directive supports an ordered array. The engine dynamically constructs a `CascadeProvider` that applies wrappers sequentially.
* *Example:* A raw L2 key is wrapped first by the internal `IVK`, producing `Ciphertext_A`. `Ciphertext_A` is then sent to an external `aws_kms` plugin. The external provider is "blinded"—it performs a cryptographic signing operation but mathematically cannot read the raw L2 key.

### Decoupling Compute from State (Asynchronous KMIP)
Cryptographic operations (Wrap/Unwrap) are isolated from data persistence (Save/Load). This allows OpenKCM to natively support standard KMIP asynchronous lifecycles (Create then Get). For example, an edge node can generate an L4 key, delegate the wrapping to a central core over gRPC, and seamlessly persist the returned ciphertext into a high-speed local Redis database.

### Distributed Master Key Sealing (Multi-KMS Quorum)
The root of the DAG (`MasterKey`) uses a `memory_resident` provider equipped with a stateful `seal` mechanism. To survive regional outages and prevent single-vendor lock-in, Krypton implements a `shamir_kms` quorum.
* The system database stores encrypted Shamir shares (referenced by `share_id`).
* At boot, Krypton fetches these ciphertexts and dispatches them in parallel to multiple external KMS plugins (AWS, Azure, GCP, HashiCorp).
* Once the `threshold` of decrypted shares is met, the Master Key is reconstructed purely in volatile memory, unlocking the rest of the DAG.

## Topology A: The Stateful Monolith
In this deployment, a single Krypton binary handles the entire L1-L4 lifecycle. It connects to external plugins for sovereign trust, manages the internal IVK/L2/L3 derivation, handles KMIP L4 data key generation, and persists state to PostgreSQL and HashiCorp Vault.

```yaml
krypton:
  node_name: "eu-central-monolith"

  system:
    MasterKey:
      description: "Krypton Global Master Key"
      #        seal:
      #          mechanism: "remote_kms"
      #          remote_kms:
      #            plugin:
      #              name: "aws_kms"
      #              type: KeyWrapUnwrap
      #              path: ./keystore-plugins/bin/aws
      #              logLevel: debug
      #              yamlConfiguration: |
      #                key_id: "arn:aws:kms:eu-central-1:123456789012:key/abcd-1234"
      #            timeout_seconds: 15
      #            retry_attempts: 3

      #        seal:
      #          mechanism: "shamir_kms"
      #          shamir_kms:
      #            threshold: 2
      #            timeout_seconds: 10
      #            shares:
      #              - plugin:
      #                  name: "aws_kms"
      #                  type: KeyWrapUnwrap
      #                  path: ./keystore-plugins/bin/aws
      #                  logLevel: debug
      #                  yamlConfiguration: |
      #                    key_id: "arn:aws:kms:eu-central-1:123456789012:key/aws-key-1"
      #
      #              - plugin:
      #                  name: "azure_kv"
      #                  type: KeyWrapUnwrap
      #                  path: ./keystore-plugins/bin/azure
      #                  logLevel: debug
      #                  yamlConfiguration: |
      #                    key_id: "https://krypton-vault.vault.azure.net/keys/azure-key-1"
      #
      #              - plugin:
      #                  name: "hashicorp_vault"
      #                  type: KeyWrapUnwrap
      #                  path: ./keystore-plugins/bin/hashicorp_vault
      #                  logLevel: debug
      #                  yamlConfiguration: |
      #                    key_id: "transit/keys/on-prem-key-1"
      seal:
        mechanism: "static_file"
        key:
          source: file
          file:
            path: "./dev/test-keys/master-key.bin"
            format: embedded
      storage:
        type: "none" # Master keys are injected/unsealed, never stored on disk

    IVK:
      description: "Internal Versioned Key"
      provider:
        type: "local_derived"
        wrapped_by: "MasterKey"
      storage:
        type: "plugins" # plugins, none
        plugins:
          - name: "hashicorp_vault_kv"
            type: KeyStorage
            path: ./keystore-plugins/bin/storage/vault
            logLevel: debug
            yamlConfiguration: |
              path: "secret/data/krypton/ivk"

    ExternalRoot:
      description: "Customer BYOK | HYOK | CYOK via plugins"
      provider:
        type: "plugins"
        plugins:
          - name: aws_kms
            type: KeyManagement
            path: ./keystore-plugins/bin/keystoreop/aws
            logLevel: debug
  tiers:
    
    L2:
      description: "Tenant Root Key - Cascade Wrapped"
      provider:
        type: "local_derived"
        wrapped_by:
          - "IVK"             # Step 1: Wrap locally
          - "External_Root"   # Step 2: Wrap externally
      storage:
        type: "plugins" # plugins, none
        plugins:
        - name: "hashicorp_vault_kv"
          type: KeyStorage
          path: ./keystore-plugins/bin/storage/vault
          logLevel: debug
          yamlConfiguration: |
            path: "secret/data/krypton/tenants/l2"

    L3:
      description: "Service Key"
      provider:
        type: "local_derived"
        wrapped_by: "L2"
      storage:
        type: "plugins" # plugins, none
        plugins:
        - name: "hashicorp_vault_kv"
          type: KeyStorage
          path: ./keystore-plugins/bin/storage/vault
          logLevel: debug
          yamlConfiguration: |
            path: "secret/data/krypton/services/l3"

    L4:
      description: "Data Encryption Key (KMIP Stateful)"
      provider:
        type: "local_generated"
        wrapped_by: "L3"
      storage:
        type: "plugins" # plugins, none
        plugins:
        - name: "postgresql_kv_store" # High-volume transactional storage for DEKs
          type: KeyStorage
          path: ./keystore-plugins/bin/storage/postgresql
          logLevel: debug
          yamlConfiguration: |
            path: "krypton.dek_storage"

  interfaces:
    - type: "http"
      bind: "0.0.0.0:443"
      target_tier: "L3" # Allows this monolith to act as a Core for external Gateways
      tls:
        require_mtls: true
    - type: "tcp"
      bind: "0.0.0.0:5696"
      target_tier: "L4" # Exposes direct KMIP access for local applications
      tls:
        require_mtls: true
 ```

 ## Topology B: The Distributed Edge-to-Core
 This architecture splits responsibilities to achieve zero-latency edge encryption while maintaining strict central governance.

 ## The Central Core (Stateless Compute Hub)
 The Core manages the heavy lifting (MasterKey quorum, L2 Cascade Wrapping against AWS KMS, and L3 derivation). It does not store L4 data keys, making it a high-throughput, disposable cryptographic oracle.

 ```yaml
krypton:
  node_name: "eu-central-core-l2&l3"

  system:
    MasterKey:
      description: "Krypton Global Master Key"
      provider:
        type: "memory_resident"
      storage:
        type: "none"

    IVK:
      description: "Internal Versioned Key"
      provider:
        type: "local_derived"
        wrapped_by: "MasterKey"
      storage:
        type: "none"

    ExternalRoot:
      description: "Customer BYOK via plugins"
      provider:
        type: "plugins"
        plugins:
          - name: aws_kms
            type: KeyManagement
            path: ./keystore-plugins/bin/keystoreop/aws
            logLevel: debug
  
  tiers:          
    L2:
      description: "Tenant Root Key"
      provider:
        type: "local_derived"
        wrapped_by:
          - "IVK"
          - "ExternalRoot"
      storage:
        type: "none"

    L3:
      description: "Service Key"
      provider:
        type: "local_derived"
        wrapped_by: "L2"
      storage:
        type: "none"

  interfaces:
    - type: "http"
      bind: "0.0.0.0:443"
      target_tier: "L3"
      tls:
        require_mtls: true
    - type: "tcp"
      bind: "0.0.0.0:5696"
      target_tier: "L3"
      tls:
        require_mtls: true
```

 ## The Edge Gateway (Stateful KMIP Execution)
Deployed close to application workloads. It generates L4 Data Keys locally, isolates them immediately with a local IVK,
and delegates the final L3 wrapping to the central Core over a secure gRPC tunnel. It persists the resulting
KMIP ciphertexts locally in Redis. The L3 raw key material mathematically never leaves the Core.

```yaml
krypton:
  node_name: "edge-gateway-app1"
  system:
    MasterKey:
      description: "Gateway Local Master Key"
      provider:
        type: "memory_resident"
      storage:
        type: "none" # Master keys are unsealed via Shamir, never stored on disk

    IVK:
      description: "Gateway Local Internal Versioned Key"
      provider:
        type: "local_derived"
        wrapped_by: "MasterKey"
      storage:
        type: "none"
        
  tiers:
        
    L3:
      description: "Upstream Service Key (Strict RPC Delegation)"
      provider:
        type: "remote_upstream"
        endpoint: "core.krypton.local:443"
        mode: "rpc_delegation"
        cache:
          enabled: false
      storage:
        type: "none" # L3 is never stored at the edge

    L4:
      description: "Data Encryption Key"
      provider:
        type: "local_generated" # Renamed from ephemeral to reflect persistence
        wrapped_by:
          - "IVK" # Wraps locally
          - "L3"  # Sends payload to Core via RPC
      storage:
        type: "none" 

  interfaces:
    - type: "tcp"
      bind: "0.0.0.0:5696"
      initial_key_tier: "L4"
      delegation_tier: "L3"
      tls:
        require_mtls: true
```

## Strategic Consequences

- Single Binary Simplicity: Development and Operations teams manage exactly one codebase. Behavior is 100% configuration-driven.
- Cryptographic Blast Radius Reduction: A fully compromised Edge Gateway yields only locally-wrapped L4 ciphertexts, which are mathematically useless without the Central Core's L3 keys.

- Seamless Scalability: Central Cores can be scaled horizontally without worrying about state synchronization for L4 keys, while Edge Gateways handle the massive volume of individual KMIP operations.
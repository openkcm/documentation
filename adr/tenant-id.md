# ADR-104: Tenant ID

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-03-05 | Architecture Design Record |

## Context

We define the following qualities for IDs of OpenKCM tenants:
1. they MUST be globally unique
1. they MUST be eternal, i.e. they can never be reused
1. they SHOULD be opaque
1. they MUST remain stable on tenant moves

Besides that we define the following schema for IDs of OpenKCM tenants:
1. Maximum length: 40 characters
1. Allowed characters follow the URI unreserved characters spec:

   `ALPHA / DIGIT / "-" / "." / "_" / "~"`

We recommend to use a UUID in the standard lower-case format
with dashes as specified in RFC 4122. Example:
`658b416d-1f4a-41d9-b70c-59a2442b0724`

Optionally this may be prefixed with a short literal string to easily
grasp the semantic of it. Example:
`kcm-658b416d-1f4a-41d9-b70c-59a2442b0724`

## Decision

We decide that tenant IDs must comply to the above mentioned restrictions.

## Consequences

### Positive (Pros)
* Tenant IDs are globally unique, which prevents collision e.g. when moving a tenant
* Tenant IDs are eternal, which clearly maps information over time
* Tenant IDs are opaque, which decouples the ID from the internal structure
  and prevents exposure of internal information
* Tenant IDs are stable, which for example allowes the move of the whole tenant
  between regions
* Tenant IDs can be technically validated
* Tenant IDs can be used in URIs
* Attacks using very long tenant IDs can be mitigated

### Negative (Cons) & Mitigations
* Tenant IDs are restricted to the above rules, which may prevent other valid formats

## References
* [RFC 4122 - A Universally Unique IDentifier (UUID) URN Namespace](https://www.rfc-editor.org/rfc/rfc4122)

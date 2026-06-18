# Code Style Guide - Protocol Buffers (protobuf)

- [Code Style Guide - Protocol Buffers (protobuf)](#code-style-guide---protocol-buffers-protobuf)
  - [Overview](#overview)
  - [File Structure and Organization](#file-structure-and-organization)
    - [File Naming](#file-naming)
    - [File Header](#file-header)
  - [Package Naming](#package-naming)
    - [Package and Service Naming Alignment](#package-and-service-naming-alignment)
  - [Syntax and Format](#syntax-and-format)
    - [Syntax Version](#syntax-version)
  - [Message Definitions](#message-definitions)
    - [Message Naming](#message-naming)
    - [Field Naming](#field-naming)
    - [Field Numbering](#field-numbering)
    - [Field Structure Example](#field-structure-example)
    - [Required vs. Optional](#required-vs-optional)
    - [Comments and Documentation](#comments-and-documentation)
    - [Oneofs](#oneofs)
  - [Enums](#enums)
    - [Enum Naming](#enum-naming)
    - [Enum Values](#enum-values)
  - [Services](#services)
    - [Service Naming](#service-naming)
    - [RPC Method Naming](#rpc-method-naming)
    - [Example Service](#example-service)
  - [Request and Response Messages](#request-and-response-messages)
    - [Naming Pattern](#naming-pattern)
    - [Common Fields](#common-fields)
  - [Imports](#imports)
    - [Import Organization](#import-organization)
  - [Well-Known Types](#well-known-types)
    - [Well-Known Types Overview](#well-known-types-overview)
    - [Common Well-Known Types](#common-well-known-types)
    - [Example Usage](#example-usage)
    - [Guidelines](#guidelines)
  - [Related Resources](#related-resources)

## Overview

This guide defines the mandatory code style and conventions that **must** be followed when writing Protocol Buffer (protobuf) definitions. All guidelines in this document are requirements. Consistent adherence to these standards improves readability, maintainability, and reduces merge conflicts.

## File Structure and Organization

### File Naming

- Use lowercase with underscores for file names: `my_service.proto`, `user_request.proto`
- Organize files by logical domain or service
- Group related messages in the same file

### File Header

- Start every proto file with the syntax declaration
- Follow with the package declaration
- Add a blank line between major sections (syntax, package, imports, messages)

```protobuf
syntax = "proto3";

package com.example.service.v1;

```

## Package Naming

- Use fully qualified, reverse domain notation: e.g. `plugin.notification.v1`
- Include version in package name for API stability: `v1`, `v2`, etc.
- Use consistent casing: lowercase with dots as separators
- Explicitly define Go package option for Go code generation
- Avoid using technology-specific terms in package names (e.g., `proto`, `grpc`)

### Package and Service Naming Alignment

- Derive the service name from the package name
- Extract the primary domain/service name from the package and capitalize it (PascalCase)
- Append the "Service" suffix
- This creates a clear relationship between package organization and API surface

**Examples:**

| Package                     | Service Name            | Rationale                                    |
|-----------------------------|-------------------------|----------------------------------------------|
| `plugin.notification.v1`    | `NotificationService`   | Primary service extracted from package       |
| `plugin.user.management.v1` | `UserManagementService` | Combined domain terms for clarity            |
| `plugin.auth.v1`            | `AuthService`           | Simple one-word domain                       |
| `plugin.order.v2`           | `OrderService`          | Service version in package, not service name |

**Good Practice:**

```protobuf
// File: notification.proto
syntax = "proto3";

package plugin.notification.v1;

option go_package = "github.com/example/plugin/proto/notification/v1;notification";

service Notification {
  rpc SendNotification(SendNotificationRequest) returns (SendNotificationResponse) {}
}
```

**Anti-pattern (avoid):**

```protobuf
// File: notification.proto
syntax = "proto3";

package plugin.proto.notification.v1;

option go_package = "github.com/example/plugin/proto/notification/v1;notification";

// Service name doesn't correspond to package name
service AlertingService {
  rpc SendAlert(SendAlertRequest) returns (SendAlertResponse) {}
}
```

## Syntax and Format

### Syntax Version

- Always specify `syntax = "proto3";` at the top of files
- Use proto3 syntax for all files and **do not** use proto2 syntax

## Message Definitions

### Message Naming

- Use PascalCase: `UserRequest`, `CreateOrderResponse`, `ProductDetails`
- Use nouns or noun phrases
- Plural names indicate collections

### Field Naming

- Use snake_case: `user_id`, `created_at`, `email_address`
- Use meaningful, self-documenting names
- Avoid abbreviations (e.g. use `user_identifier` instead of `usr_id`)

### Field Numbering

- Start numbering from 1
- Use a clear numbering scheme for fields (e.g., 1,2,3,... or 10,20,30,...)
- Use a consecutive numbering based on the chosen scheme
- Reserve field numbers for removed fields to prevent reuse
- Never reuse field numbers: use `reserved` statement instead

### Field Structure Example

```protobuf
message User {
  // Reserved fields to prevent accidental reuse
  reserved 5, 7, 9;
  reserved "status", "legacy_id";

  // Unique identifier for the user
  int64 id = 1;
  
  // User's email address
  string email = 2;
  
  // User's full name
  string name = 3;
  
  // Timestamps
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 6;
  
  // Collection of tags
  repeated string tags = 8;
}
```

### Required vs. Optional

- In proto3, all fields are implicitly optional (except `oneof` and `repeated`)
- Use `optional` keyword explicitly for fields that might not always be present
- Use `repeated` for zero or more occurrences of a field
- Do not use `required` in proto3 (breaks backward compatibility)
- Document the expected presence of fields in comments related to the use-case (e.g., "This field is required for CreateUserRequest")
- Use wrapper types (e.g., `google.protobuf.StringValue`) if you need to distinguish between "not set" and "set to default value"

### Comments and Documentation

- Add comments for all public fields, messages, and services
- Use `//` for single-line comments
- Start comments with a capital letter
- Line comments must be above the field
- Field comments can be in line with the field if short, but prefer above for longer comments
- Include purpose, constraints, and examples where helpful
- **Document use cases and context**: Explain when and how fields/messages should be used
- **Include constraints and validation rules**: Document expected formats, ranges, or business rules
- **Provide context for services**: Explain what the service does and when to use it
- **Provide examples for complex fields**: Help developers understand the expected data structure

**Good Practice:**

```protobuf
// User represents a user account in the system.
// Used in authentication and authorization flows.
message User {
  // Unique identifier for the user.
  // This value is immutable and assigned by the system on creation.
  int64 id = 1;

  // Email address for user notifications and login.
  // Must be a valid email format (e.g., user@example.com).
  // Required for CreateUserRequest, optional in UpdateUserRequest.
  string email = 2;

  // User role for access control.
  // Determines which operations the user can perform.
  // Example values: ADMIN, USER, GUEST
  UserRole role = 3;
}
```

### Oneofs

- Use for mutually exclusive choices
- Use snake_case for oneof name
- Place oneof definitions at the end of the message
- Example:

```protobuf
message PaymentMethod {
  oneof payment_option {
    CreditCard credit_card = 1;
    BankAccount bank_account = 2;
    PayPal paypal = 3;
  }
}
```

## Enums

### Enum Naming

Use PascalCase: `UserStatus`, `OrderState`, `PaymentMethod`

### Enum Values

- Add `<ENUM_NAME>_UNSPECIFIED` as the first value with number 0

- Use SCREAMING_SNAKE_CASE: `USER_STATUS_UNSPECIFIED`, `ORDER_STATE_UNSPECIFIED`
- **No Prefix values** with the enum name (for name spacing)
- Example:

```protobuf
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  ACTIVE = 1;
  INACTIVE = 2;
  BANNED = 3;
}
```

## Services

### Service Naming

- Use PascalCase: `User`, `Order`, `Payment`
- The "Service" suffix is optional but can be used for clarity
- Check also [Package and Service Naming Alignment](#package-and-service-naming-alignment) for guidelines on aligning service names with package names

### RPC Method Naming

- Use PascalCase: `GetUser`, `CreateOrder`, `ListProducts`
- Use standard verb prefixes:
  - `Get` - retrieve a single resource
  - `Create` - create a new resource
  - `Update` - update an existing resource
  - `Delete` - delete a resource
  - `List` - retrieve multiple resources
  - `Search` - search for resources

### Example Service

```protobuf
service User {
  // Get a user by ID
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {}

  // List all users with pagination
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {}

  // Create a new user
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {}

  // Update an existing user
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse) {}

  // Delete a user
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse) {}
}
```

## Request and Response Messages

### Naming Pattern

- Request: `[Method]Request` (e.g., `GetUserRequest`, `CreateOrderRequest`)
- Response: `[Method]Response` (e.g., `GetUserResponse`, `CreateOrderResponse`)

### Common Fields

Include these standard fields in responses:

```protobuf
message GetUserResponse {
  // The requested user
  User user = 1;
  
  // Operation timestamp
  google.protobuf.Timestamp timestamp = 2;
  
  // Error message (if any)
  string error_message = 3;
}
```

## Imports

### Import Organization

- Group imports by category
- Use relative or absolute paths consistently
- Common order:
  1. Syntax declaration
  2. Package declaration
  3. Standard protobuf imports
  4. Third-party imports
  5. Local/internal imports

```protobuf
syntax = "proto3";
package com.example.service.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

import "github.com/example/common/v1/error.proto";

import "internal/models/user.proto";
```

## Well-Known Types

### Well-Known Types Overview

- Use Google's [Well-Known Types (WKT)](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf) instead of custom implementations where applicable
- WKT are pre-defined messages that provide common functionality across different languages

### Common Well-Known Types

| Type                          | Use Case        | Example                              |
|-------------------------------|-----------------|--------------------------------------|
| `google.protobuf.Timestamp`   | Dates and times | `created_at`, `updated_at`           |
| `google.protobuf.Duration`    | Time intervals  | `timeout`, `ttl`                     |
| `google.protobuf.StringValue` | Optional string | Wrapper for nullable strings         |
| `google.protobuf.Int32Value`  | Optional int32  | Wrapper for nullable integers        |
| `google.protobuf.BoolValue`   | Optional bool   | Wrapper for nullable booleans        |
| `google.protobuf.Empty`       | No data         | Used in RPC responses with no data   |
| `google.protobuf.Struct`      | Dynamic data    | JSON-like structures (use sparingly) |
| `google.protobuf.ListValue`   | List of values  | Dynamic lists (use sparingly)        |

### Example Usage

```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";

message Event {
  // Event identifier
  int64 id = 1;
  
  // When the event occurred
  google.protobuf.Timestamp timestamp = 2;
  
  // How long the event took
  google.protobuf.Duration duration = 3;
  
  // Event name
  string name = 4;
}

service EventService {
  // Record an event
  rpc RecordEvent(Event) returns (google.protobuf.Empty) {}
}
```

### Guidelines

- Prefer `google.protobuf.Timestamp` over Unix timestamps (int64, int32)
- Use `google.protobuf.Duration` for time intervals instead of custom duration messages
- Use wrapper types (`StringValue`, `Int32Value`, etc.) sparingly - prefer `optional` keyword in proto3
- Use `google.protobuf.Empty` for RPC methods that don't return data
- Avoid `google.protobuf.Any` and `google.protobuf.Struct` unless absolutely necessary

## Related Resources

- [Official Protocol Buffers Style Guide](https://protobuf.dev/programming-guides/style/)
- [buf documentation](https://buf.build/docs/lint/rules/#rules-and-categories)
- [Protocol Buffers Style Guide](https://developers.google.com/protocol-buffers/docs/style)
- [Protocol Buffers Best Practices](best-practices-protobuf.md)

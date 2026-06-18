# Best Practices for Protocol Buffer (protobuf) Files

## Syntax and Versioning

### Syntax Version Migration

- **Strongly recommended**: Migrate to Edition 2024 (`edition = "2024";`) for new proto files
- Edition 2024 provides better defaults and more flexible field presence semantics
- For existing files using `syntax = "proto3";`, plan a migration to Edition 2024
- Edition 2024 unifies proto2 and proto3 features and provides a clearer path forward
- See [Protobuf Editions](https://protobuf.dev/editions/overview/) for migration guidance

### Version Stability

- Never change field numbers or remove fields without marking as `reserved`
- When removing a message or field, add it to the `reserved` section
- Maintain backward compatibility within a major version
- Use semantic versioning in package names

## Text Formatting

### Whitespace

- No blank lines between related message fields
- Use consistent spacing around `=`, `:`, and parentheses

### Indentation and Line Length

- Use 2 spaces for indentation (not tabs)
- Keep lines to a maximum of 100 characters where possible
- Break long lines at logical points

### Deprecations

#### Deprecating Fields

- Use the `deprecated = true;` option to mark fields that should no longer be used
- Keep deprecated fields in the message for backward compatibility
- Always add the field to the `reserved` section when it's safe to remove (usually after 2+ major versions)
- Document why the field is deprecated and what should be used instead

```protobuf
message User {
  // Unique identifier for the user
  int64 id = 1;
  
  // Unique identifier for the user
  string uid = 2 [deprecated = true];
  
  // Email address (replaces legacy_email)
  string email = 3;
  
  // DEPRECATED: Use email field instead
  string legacy_email = 4 [deprecated = true];
}
```

#### Deprecating Services and Methods

- Use the `deprecated = true;` option for RPC methods or entire services
- Provide alternative methods or services in comments
- Maintain deprecated methods for backward compatibility during deprecation period

```protobuf
service UserService {
  // Get a user by ID
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {}
  
  // DEPRECATED: Use GetUser instead
  rpc GetUserInfo(GetUserInfoRequest) returns (GetUserInfoResponse) {
    option deprecated = true;
  }
}
```

## Related Resources

- [Protocol Buffers Style Guide](style-guide-protobuf.md)
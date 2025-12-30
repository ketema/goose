# Implementation Plan: MCP Protocol Compliance for Mack

**Date**: 2025-12-30
**Branch**: feature/mcp-protocol-compliance
**Status**: M3 PLAN - REVISED after rmcp investigation
**Revision**: 2.0 - Upgrade approach (vs custom transport)

---

## 1. Problem Statement

Mack's MCP client (via rmcp 0.9.1) fails to properly negotiate with MCP-compliant servers (AI Panel).

**Observed failures**:
- `streamable_http` mode: Hangs - doesn't follow 202 Accepted pattern
- `sse` mode: Sends GET instead of POST, receives 405 Method Not Allowed

**Root cause**: rmcp 0.9.1 transport implementations don't follow MCP spec 2025-11-25.

---

## 2. Investigation Findings (Phase 0)

### 2.1 rmcp Version Analysis

| Version | Release Date | Key Changes |
|---------|-------------|-------------|
| 0.9.1 | Nov 24, 2025 | Current Mack version |
| 0.10.0 | Dec 1, 2025 | Custom client notifications |
| 0.11.0 | Dec 8, 2025 | **SSE transport REMOVED** (breaking) |
| 0.12.0 | Dec 18, 2025 | Custom requests/notifications, OAuth improvements |

**Critical Finding**: rmcp is the **official MCP Rust SDK** from modelcontextprotocol. Version 0.12.0 is MCP spec 2025-11-25 compliant.

### 2.2 rmcp 0.12.0 Streamable HTTP Compliance

Verified in source (`streamable_http_client.rs`):
- ✅ Handles 202 Accepted responses properly (`StreamableHttpPostResponse::Accepted`)
- ✅ Session ID extraction and propagation
- ✅ SSE stream parsing from HTTP responses
- ✅ POST for messages, GET for streams

### 2.3 Feature Migration

| Current 0.9.1 Feature | 0.12.0 Status |
|----------------------|---------------|
| `client` | ✅ Available |
| `reqwest` | ✅ Available |
| `transport-child-process` | ✅ Available |
| `transport-sse-client` | ❌ REMOVED |
| `transport-sse-client-reqwest` | ❌ REMOVED |
| `transport-streamable-http-client` | ✅ Available |
| `transport-streamable-http-client-reqwest` | ✅ Available |

---

## 3. Revised Implementation Strategy

### Original Plan (Option C - Custom Transport)
- Effort: 9-14 hours
- Risk: rmcp trait compatibility uncertain
- Maintenance: Ongoing custom code

### Revised Plan (Option A - Upgrade rmcp)
- Effort: 2-4 hours
- Risk: Low (official SDK)
- Maintenance: None (upstream maintained)

**Selected**: **Option A** - Upgrade rmcp 0.9.1 → 0.12.0

**Rationale**:
1. rmcp 0.12.0 is MCP spec 2025-11-25 compliant (verified in source)
2. Official SDK from modelcontextprotocol - guaranteed compatibility
3. 3 releases in 3 weeks shows active maintenance
4. SSE removal aligns with MCP spec direction (Streamable HTTP is preferred)
5. Minimal code changes vs custom implementation

---

## 4. Implementation Plan (Revised)

### Phase 1: Dependency Upgrade (30 min)

**File**: `Cargo.toml` (workspace root, line 18)

```toml
# Before:
rmcp = { version = "0.9.1", features = ["schemars", "auth"] }

# After:
rmcp = { version = "0.12.0", features = ["schemars", "auth"] }
```

**File**: `crates/goose/Cargo.toml` (lines 19-27)

```toml
# Before:
rmcp = { workspace = true, features = [
    "client",
    "reqwest",
    "transport-child-process",
    "transport-sse-client",              # REMOVE
    "transport-sse-client-reqwest",      # REMOVE
    "transport-streamable-http-client",
    "transport-streamable-http-client-reqwest",
] }

# After:
rmcp = { workspace = true, features = [
    "client",
    "reqwest",
    "transport-child-process",
    "transport-streamable-http-client",
    "transport-streamable-http-client-reqwest",
] }
```

### Phase 2: Code Migration (1-2 hours)

**File**: `crates/goose/src/agents/extension_manager.rs`

#### 2.1 Remove SSE Transport Handling (lines 382-401)

Replace:
```rust
ExtensionConfig::Sse { uri, timeout, .. } => {
    let transport = SseClientTransport::start(uri.to_string()).await...
    // ... existing SSE code
}
```

With deprecation error:
```rust
ExtensionConfig::Sse { name, .. } => {
    return Err(ExtensionError::ConfigError(format!(
        "SSE transport deprecated in MCP spec 2025-11-25. \
         Extension '{}': Change 'type: sse' to 'type: streamable_http' in config.",
        name
    )));
}
```

#### 2.2 Update Imports (lines 9-12)

Remove unused SSE imports:
```rust
// Remove:
use rmcp::transport::SseClientTransport;
```

#### 2.3 Verify Streamable HTTP Code (lines 467-504)

The existing Streamable HTTP code should work with rmcp 0.12.0:
```rust
let transport = StreamableHttpClientTransport::with_client(
    client,
    StreamableHttpClientTransportConfig {
        uri: uri.clone().into(),
        ..Default::default()
    },
);
```

**May need adjustment** if API changed - verify after upgrade.

### Phase 3: Build Verification (30 min)

```bash
# Clean build
cargo clean
cargo build --all

# Run existing tests
cargo test --all

# Check for deprecation warnings
cargo clippy --all
```

### Phase 4: Integration Testing (1 hour)

**Prerequisite**: AI Panel running on localhost:30097

1. Update user config to use streamable_http:
```yaml
aipanel:
  enabled: true
  type: streamable_http  # Changed from sse
  uri: http://localhost:30097/mcp
```

2. Run Mack and test AI Panel tools:
```bash
# Start Mack
cargo run --bin goose-cli

# In Mack, invoke AI Panel tool
> Use the AI Panel to critique this code...
```

3. Verify in tcpdump:
   - POST requests with proper Accept header
   - 202 responses handled correctly
   - Tool responses received

---

## 5. Test Strategy

### Unit Tests (Cargo test)

Existing rmcp transport tests should pass after upgrade.

### Integration Tests

```rust
#[test]
#[ignore] // Requires AI Panel running
async fn test_ai_panel_streamable_http() {
    let config = ExtensionConfig::StreamableHttp {
        uri: "http://localhost:30097/mcp".to_string(),
        timeout: Some(300),
        ..Default::default()
    };

    let manager = ExtensionManager::new();
    let client = manager.instantiate_client(&config).await;

    assert!(client.is_ok(), "AI Panel connection failed: {:?}", client.err());
}
```

### Manual Verification

1. List AI Panel tools
2. Execute critique_code tool
3. Verify SSE streaming responses work
4. Test session persistence across calls

---

## 6. Risk Assessment (Revised)

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| API breaking changes | Low | Medium | Review rmcp changelog |
| Compile errors | Medium | Low | Fix as they appear |
| Runtime behavior change | Low | Medium | Integration testing |
| Config migration | Low | Low | Clear deprecation message |

---

## 7. Success Criteria

1. **Build**: `cargo build --all` succeeds
2. **Tests**: `cargo test --all` passes
3. **Functional**: AI Panel tools callable from Mack
4. **Config**: Clear deprecation message for SSE type

---

## 8. Timeline Estimate (Revised)

| Phase | Effort | Dependencies |
|-------|--------|--------------|
| Phase 1: Upgrade | 30 min | None |
| Phase 2: Code Migration | 1-2 hours | Phase 1 |
| Phase 3: Build Verification | 30 min | Phase 2 |
| Phase 4: Integration Testing | 1 hour | Phase 3, AI Panel running |

**Total**: 3-4 hours (down from 9-14 hours)

---

## 9. References

- [rmcp crates.io](https://crates.io/crates/rmcp) - Version 0.12.0
- [rmcp GitHub](https://github.com/modelcontextprotocol/rust-sdk) - Official MCP SDK
- [MCP Spec 2025-11-25 Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [Why MCP Deprecated SSE](https://blog.fka.dev/blog/2025-06-06-why-mcp-deprecated-sse-and-go-with-streamable-http/)

---

## 10. Approval Required

- [ ] AI Panel critique reviewed
- [ ] Human approval received
- [ ] Feature branch created

**I will NOT code until this plan is approved.**

---

## Appendix: Evidence

### A.1 rmcp 0.12.0 202 Handling (Source Verification)

From `streamable_http_client.rs`:
```rust
Ok(StreamableHttpPostResponse::Accepted) => {
    tracing::trace!("client message accepted");
    Ok(())
}
```

### A.2 Session ID Handling (Source Verification)

```rust
let (message, session_id) = match self.client.post_message(
    config.uri.clone(),
    initialize_request,
    None,  // No session on initial request
    self.config.auth_header,
)
```

### A.3 SSE Stream Parsing (Source Verification)

```rust
let payload = event.data.unwrap_or_default();
if payload.trim().is_empty() {
    continue;
}
let message: ServerJsonRpcMessage = serde_json::from_str(&payload)?;
```

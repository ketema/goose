# MCP Streamable HTTP Transport Contract

**Protocol Version**: MCP 2025-11-25
**Source**: https://modelcontextprotocol.io/specification/2025-11-25/basic/transports
**Last Verified**: 2025-12-30
**Contract ID**: MCP-TRANSPORT-001

---

## Purpose

This contract defines the expected behavior of MCP Streamable HTTP transport implementations
as used by Mack (Goose CLI) to communicate with MCP servers like AI Panel.

---

## Overview

MCP 2025-11-25 defines Streamable HTTP as the standard transport, replacing the older HTTP+SSE
transport from 2024-11-05. Key characteristics:

1. Single HTTP endpoint supporting both POST and GET methods
2. POST for client→server messages, GET for server→client SSE streams
3. Session management via Mcp-Session-Id header
4. Protocol version negotiation via Mcp-Protocol-Version header

**Note on SSE**: SSE (Server-Sent Events) is actively used within Streamable HTTP for
server→client responses. The deprecation applies to the standalone "HTTP+SSE transport"
from the 2024-11-05 spec, not SSE as a response format.

**Note on rmcp**: rmcp 0.11.0 removed `SseClientTransport` (standalone SSE) in favor of
`StreamableHttpClientTransport` which handles SSE responses internally.

---

## Contract Clauses

### CLAUSE-001: Standalone SSE Transport Removal

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "This [Streamable HTTP] replaces the HTTP+SSE transport from protocol version 2024-11-05."
> - MCP Spec 2025-11-25, Transports section

**rmcp Implementation**:
> "SSE transport REMOVED" - rmcp 0.11.0 changelog (Dec 8, 2025)

**Derived Test Assertions**:
- Client MUST reject ExtensionConfig::Sse with configuration error
- Error message SHOULD indicate transport is deprecated/removed
- Client MUST NOT attempt to establish standalone SSE connection
- Client SHOULD direct users to use Streamable HTTP instead

**Applies To**: ExtensionConfig::Sse variant handling

---

### CLAUSE-002: HTTP Methods

**Requirement Level**: MUST (Server), MUST (Client)

**Spec Quote**:
> "The server MUST provide a single HTTP endpoint path that supports both POST and GET methods."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- Client MUST send POST for JSON-RPC messages
- Client MAY use GET to open SSE stream for receiving server messages

**Applies To**: All HTTP requests to MCP endpoint

---

### CLAUSE-003: Accept Header (POST)

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "The client MUST include an Accept header, listing both application/json and
> text/event-stream as supported content types."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- POST requests MUST include Accept header
- Accept header MUST list application/json
- Accept header MUST list text/event-stream

**Applies To**: POST requests to MCP endpoint

---

### CLAUSE-004: Accept Header (GET)

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "The client MUST include an Accept header, listing text/event-stream as a supported content type."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- GET requests MUST include Accept header
- Accept header MUST list text/event-stream

**Applies To**: GET requests to MCP endpoint

---

### CLAUSE-005: 202 Accepted Response Handling

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "If the input is a JSON-RPC response or notification: If the server accepts the input,
> the server MUST return HTTP status code 202 Accepted with no body."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- Client MUST treat 202 response as success (not error)
- Client MUST NOT retry on 202 response
- Client MUST NOT expect response body for 202

**Applies To**: POST response handling for notifications/responses

---

### CLAUSE-006: Session ID Handling

**Requirement Level**: SHOULD (Server), MUST (Client)

**Spec Quote**:
> "If an Mcp-Session-Id is returned by the server during initialization, clients using
> the Streamable HTTP transport MUST include it in the Mcp-Session-Id header on all
> of their subsequent HTTP requests."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- Client SHOULD extract Mcp-Session-Id from initialize response
- Client MUST include Mcp-Session-Id in subsequent requests if received
- Client MUST handle responses without session ID (initial request)

**Applies To**: Session management

---

### CLAUSE-007: Protocol Version Header

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "If using HTTP, the client MUST include the Mcp-Protocol-Version: <protocol-version>
> HTTP header on all subsequent requests to the MCP server"
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- Client MUST include Mcp-Protocol-Version header after initialization
- Header value SHOULD be negotiated protocol version

**Applies To**: All HTTP requests after initialization

---

### CLAUSE-008: SSE Response Parsing

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "The server MUST either return Content-Type: text/event-stream, to initiate an SSE stream,
> or Content-Type: application/json, to return one JSON object. The client MUST support
> both these cases."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- Client MUST parse Content-Type: application/json as single JSON-RPC response
- Client MUST parse Content-Type: text/event-stream as SSE stream
- Client MUST extract JSON-RPC messages from SSE event data fields

**Applies To**: Response parsing

---

### CLAUSE-009: Session Expiration

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "When a client receives HTTP 404 in response to a request containing an Mcp-Session-Id,
> it MUST start a new session by sending a new InitializeRequest without a session ID attached."
> - MCP Spec 2025-11-25, Streamable HTTP section

**Derived Test Assertions**:
- Client MUST handle 404 as session expiration signal
- Client MUST re-initialize without session ID on 404

**Applies To**: Session error handling

---

### CLAUSE-010: JSON-RPC Message Format

**Requirement Level**: MUST (Client)

**Spec Quote**:
> "All messages MUST be valid JSON-RPC 2.0 messages."
> - MCP Spec 2025-11-25, Protocol section

**Derived Test Assertions**:
- Outgoing messages MUST include jsonrpc: "2.0"
- Requests MUST include method and id
- Notifications MUST include method but NOT id

**Applies To**: All MCP messages

---

## Error Handling Summary

| HTTP Status | Client Behavior | Contract Clause |
|-------------|-----------------|-----------------|
| 200 OK | Parse response body per Content-Type | CLAUSE-008 |
| 202 Accepted | Treat as success, no body expected | CLAUSE-005 |
| 400 Bad Request | Report client error | N/A (standard HTTP) |
| 404 Not Found | Session expired, re-initialize | CLAUSE-009 |
| 405 Method Not Allowed | Wrong HTTP method used | CLAUSE-002 |
| 500+ | Report server error | N/A (standard HTTP) |

---

## Mock Derivation Rules

Any mock for MCP Streamable HTTP transport MUST:
1. Reference a specific CLAUSE from this contract
2. Implement ONLY the behavior specified in that clause
3. Use the exact requirement level (MUST/SHOULD/MAY)
4. Not invent behaviors not in the spec

**Example mock comment**:
```rust
// Mock per CLAUSE-005: Server returns 202 Accepted for notification
// Contract: contracts/mcp_streamable_http_transport.contract.md
let mock_response = Response::builder()
    .status(202)
    .body(Body::empty())
    .unwrap();
```

---

## Verification Tests

Contract verification tests MUST:
1. Run against REAL MCP server (AI Panel on localhost:30097)
2. Be marked with `#[ignore]` for CI without live server
3. Verify each CLAUSE independently
4. Document which CLAUSEs are verified

---

## Changelog

| Date | Version | Change |
|------|---------|--------|
| 2025-12-30 | 1.1.0 | Updated to MCP spec 2025-11-25, clarified SSE deprecation scope |
| 2025-12-30 | 1.0.0 | Initial contract for rmcp 0.12.0 upgrade |

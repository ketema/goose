# Goose Architecture Research for Constitutional Framework Migration

**Date**: 2025-12-30
**Purpose**: Evaluate Goose as Claude Code alternative with custom constitutional framework
**Researcher**: Claude (Anthropic) via ametek_chess orchestration

---

## Executive Summary

Goose is a **highly viable** alternative to Claude Code for implementing a custom constitutional framework. Key findings:

1. **Native CLAUDE.md support** - Goose already reads CLAUDE.md as a hints file
2. **Full system prompt control** - No hidden token taxation
3. **MCP protocol compliance** - Supports stdio, SSE, and Streamable HTTP transports
4. **AI Panel partially working** - Config exists at `streamable_http` on port 30097
5. **Apache 2.0 license** - Fully open, modifiable

---

## 1. System Prompt Architecture

### Location
```
crates/goose/src/prompts/
├── system.md              # Main system prompt (~100 lines)
├── system_gpt_4.1.md      # GPT-4.1 specific prompt
├── desktop_prompt.md      # Desktop app additions
├── subagent_system.md     # Sub-agent prompts
├── plan.md                # Planning mode prompt
└── recipe.md              # Recipe execution prompt
```

### System Prompt Content (system.md)
The base system prompt is **minimal and transparent**:
- Agent identity ("you are goose, created by Block")
- Extension instructions (MCP server documentation)
- Response formatting guidelines (Markdown)
- **NO hidden reminders**
- **NO TodoWrite nudges**
- **NO token-taxing injections**

### How to Inject Constitutional Framework

**Option 1: Global Hints File**
```
~/.config/goose/.goosehints
```
- Loaded automatically for all sessions
- Supports `@file.md` imports for modular constitution

**Option 2: Project-Level CLAUDE.md**
```
~/projects/ametek_chess/CLAUDE.md
```
- Goose natively recognizes CLAUDE.md as hints file
- Inherits full hints hierarchy (git root → cwd)

**Option 3: Modify system.md directly**
```rust
// crates/goose/src/prompts/system.md
// Add your constitutional framework here
```
- Requires recompilation
- Most direct integration
- Can use Jinja2 templating for dynamic sections

---

## 2. Hints System (GOOSE.md Equivalent)

### File Naming
```rust
pub const GOOSE_HINTS_FILENAME: &str = ".goosehints";
pub const AGENTS_MD_FILENAME: &str = "AGENTS.md";
```

### Configurable Hints Files
From `load_hints.rs` tests:
```rust
// Goose accepts multiple hint filenames including CLAUDE.md
let hints = load_hint_files(
    dir.path(),
    &["CLAUDE.md".to_string(), GOOSE_HINTS_FILENAME.to_string()],
    &gitignore
);
```

### Hints Hierarchy
1. **Global hints**: `~/.config/goose/.goosehints`
2. **Git root hints**: Walk up from cwd to git root
3. **Directory hints**: Each directory in path can have hints
4. **Merged output**: All hints concatenated with section headers

### File Import Syntax
```markdown
# In .goosehints
@README.md           # Import relative file
@../docs/api.md      # Import from parent
@config/setup.md     # Import from subdirectory
```

### Import Boundary
- **With git**: Can import any file within git repo
- **Without git**: Can only import from current directory

---

## 3. MCP Server Integration

### Supported Transport Types

| Type | Config Key | Use Case |
|------|------------|----------|
| **stdio** | `type: stdio` | Command-line MCP servers (npx, uvx) |
| **sse** | `type: sse` | Server-Sent Events (legacy HTTP) |
| **streamable_http** | `type: streamable_http` | MCP Streamable HTTP (newest) |
| **builtin** | `type: builtin` | Bundled with Goose |
| **platform** | `type: platform` | Direct agent access |
| **frontend** | `type: frontend` | UI-provided tools |

### Current AI Panel Config
From `~/.config/goose/config.yaml`:
```yaml
aipanel:
  enabled: true
  type: streamable_http
  name: AI Panel
  description: Multi model AI Consultation Board
  uri: http://localhost:30097/mcp
  envs: {}
  env_keys: []
  headers: {}
  timeout: 300
  bundled: null
```

### MCP Library
Goose uses **rmcp 0.9.1** with features:
- `transport-streamable-http-server`
- `transport-io` (stdio)
- `client`, `server`, `macros`
- `schemars`, `auth`

### Environment Variable Substitution
Headers support `${VAR}` and `$VAR` syntax:
```yaml
headers:
  Authorization: Bearer ${AI_PANEL_TOKEN}
```

---

## 4. AI Panel Integration Issues

### Potential Issues to Investigate

1. **Port Mismatch**
   - Config shows port `30097`
   - AI Panel typically runs on port `30097` (verify)

2. **Streamable HTTP vs SSE**
   - AI Panel may use SSE, not Streamable HTTP
   - Try changing `type: sse` instead of `streamable_http`

3. **Authentication**
   - Headers section is empty
   - May need `Authorization: Bearer ${AI_PANEL_TOKEN}`

4. **Timeout**
   - 300 seconds may be too short for parallel multi-model queries
   - Consider increasing to 600

### Recommended Config Changes
```yaml
aipanel:
  enabled: true
  type: sse  # Try SSE if streamable_http fails
  name: AI Panel
  description: Multi model AI Consultation Board
  uri: http://localhost:30097/mcp
  envs: {}
  env_keys:
    - AI_PANEL_TOKEN  # If auth needed
  headers:
    Authorization: Bearer ${AI_PANEL_TOKEN}
  timeout: 600  # Increase for multi-model
  bundled: null
```

---

## 5. Constitutional Framework Migration Plan

### Phase 1: Hints-Based Constitution
1. Copy `~/.claude/CLAUDE.md` → `~/.config/goose/.goosehints`
2. Adapt Claude Code-specific references
3. Test with existing Goose installation

### Phase 2: Project-Level Integration
1. Use existing `~/projects/ametek_chess/CLAUDE.md`
2. Goose will automatically load it as project hints
3. Verify hints appear in system prompt

### Phase 3: Hook System Equivalent
Goose doesn't have a hook system equivalent to Claude Code's:
- **PreToolUse** → No direct equivalent
- **PostToolUse** → No direct equivalent
- **UserPromptSubmit** → No direct equivalent
- **PreCompact** → No compaction (you control context)

**Alternative**: Use MCP server to implement hook-like behavior:
1. Create custom MCP server for fact injection
2. Tool calls trigger your logic
3. Returns injected context

### Phase 4: Full System Prompt Modification
For deepest integration:
1. Fork Goose repo
2. Modify `crates/goose/src/prompts/system.md`
3. Add constitutional sections with Jinja2 templating
4. Build custom binary: `cargo build --release`

---

## 6. Key Advantages Over Claude Code

| Aspect | Claude Code | Goose |
|--------|-------------|-------|
| System prompts | Hidden, token-taxing | Visible, customizable |
| TodoWrite reminders | Forced, ~80 tokens each | Optional, transparent |
| Token visibility | Hidden for subscription | Full visibility |
| LLM choice | Claude only | Any LLM |
| License | Proprietary | Apache 2.0 |
| Hooks | Built-in but limited | Build your own via MCP |
| MCP support | Native | Native (reference impl) |
| Context control | Auto-compaction | You decide |

---

## 7. Key Disadvantages / Trade-offs

1. **No built-in fact injection** - Must build custom MCP server
2. **No pre-tool hooks** - Cannot intercept before tool execution
3. **Different tool ecosystem** - Extensions vs Claude Code tools
4. **Learning curve** - Different config format (YAML vs JSON)
5. **Less mature** - Younger project, more rough edges

---

## 8. File References

### Core Architecture
- `crates/goose/src/agents/agent.rs` - Main agent logic
- `crates/goose/src/hints/load_hints.rs` - Hints loading
- `crates/goose/src/prompts/system.md` - System prompt
- `crates/goose/src/config/extensions.rs` - Extension management

### MCP Implementation
- `crates/goose/src/agents/extension_manager.rs` - MCP client handling
- `crates/goose/src/agents/mcp_client.rs` - MCP client trait
- `crates/goose/src/agents/extension.rs` - Extension config types

### Documentation
- `documentation/docs/getting-started/using-extensions.md`
- `documentation/docs/mcp/` - Individual MCP server docs

---

## 9. Existing Config Analysis

Your current `~/.config/goose/config.yaml`:
- **Provider**: Google Gemini 2.5 Pro
- **Extensions enabled**: developer, extensionmanager, skills, todo, chatrecall, sequential-thinking, aipanel
- **AI Panel**: streamable_http on port 30097

### Sequential Thinking MCP
```yaml
sequential-thinking:
  enabled: true
  type: stdio
  cmd: /opt/homebrew/bin/node
  args:
    - /Users/ketema/projects/ametek_chess/submodules/mcp-servers/src/sequentialthinking/dist/index.js
```
This is shared with your Claude Code setup.

---

## 10. Next Steps (Research Only)

1. **Test AI Panel with SSE transport** - Change type from `streamable_http` to `sse`
2. **Verify AI Panel MCP endpoint** - Confirm `/mcp` endpoint responds correctly
3. **Create GOOSE.md** - Adapt CLAUDE.md for Goose-specific features
4. **Design hook alternative** - Plan MCP server for fact injection
5. **Benchmark token usage** - Compare same task in Claude Code vs Goose

---

## Appendix A: rmcp Crate Version

From `Cargo.toml`:
```toml
rmcp = { version = "0.9.1", features = ["schemars", "auth"] }
```

Transport features available:
- `transport-io` (stdio)
- `transport-streamable-http-server`
- `transport-streamable-http-client`
- SSE via `SseClientTransport`

---

## Appendix B: Extension Config Schema

```rust
pub enum ExtensionConfig {
    Sse { name, description, uri, envs, env_keys, timeout, bundled, available_tools },
    Stdio { name, description, cmd, args, envs, env_keys, timeout, bundled, available_tools },
    Builtin { name, description, display_name, timeout, bundled, available_tools },
    Platform { name, description, bundled, available_tools },
    StreamableHttp { name, description, uri, envs, env_keys, headers, timeout, bundled, available_tools },
    Frontend { name, ... },
}
```

---

*Research compiled from Goose source code analysis. No code modifications made.*

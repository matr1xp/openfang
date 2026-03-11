# OpenFang — Gemini Agent Instructions

## Project Overview
OpenFang is an open-source Agent Operating System written in Rust (14 crates).
- Config: `~/.openfang/config.toml`
- Default API: `http://127.0.0.1:4200`
- CLI binary: `target/release/openfang.exe` (or `target/debug/openfang.exe`)

## Build & Verify Workflow
After every feature implementation, run ALL THREE checks:
```bash
cargo build --workspace --lib          # Must compile (use --lib if exe is locked)
cargo test --workspace                 # All tests must pass (currently 2075+ tests)
cargo clippy --workspace --all-targets -- -D warnings  # Zero warnings
```

### Running Specific Tests
```bash
# Unit tests for a specific crate
cargo test -p openfang-kernel

# Integration tests
cargo test -p openfang-api --test api_integration_test -- --nocapture

# LLM integration tests (requires GROQ_API_KEY)
GROQ_API_KEY=<key> cargo test -p openfang-api --test api_integration_test -- --nocapture
```

## Architecture

### Crate Dependency Graph
```text
openfang-types      ← Core types, no deps on other crates
    ↓
openfang-memory     ← SQLite persistence, vector embeddings, sessions
    ↓
openfang-runtime    ← Agent loop, LLM drivers (Anthropic/Gemini/OpenAI), 53 tools, WASM sandbox
    ↓
openfang-kernel     ← Kernel orchestrator, registry, scheduler, workflows
    ↓
openfang-api        ← 140+ REST/WS/SSE endpoints, OpenAI-compatible API
    ↓
openfang-cli        ← CLI with daemon management, TUI dashboard
openfang-desktop    ← Tauri 2.0 native app
```

### Key Modules by Crate

**openfang-types**: Core domain types
- `agent.rs`: AgentId, AgentManifest, AgentState, ModelRoutingConfig
- `config.rs`: KernelConfig, ChannelConfig, McpServerConfigEntry
- `tool.rs`: ToolDefinition, ToolInputSchema
- `taint.rs`: Information flow taint tracking for security

**openfang-runtime**: Agent execution engine
- `agent_loop.rs`: Main agent loop (`run_agent_loop`, `run_agent_loop_streaming`)
- `llm_driver.rs`: LLM driver trait + CompletionRequest/Response
- `drivers/`: Anthropic, Gemini, OpenAI, Groq, etc. (27 providers)
- `tool_runner.rs`: Tool execution engine with 53 built-in tools
- `sandbox.rs`: WASM dual-metered sandbox for untrusted code
- `mcp.rs`: MCP (Model Context Protocol) server connections

**openfang-kernel**: Core orchestrator
- `kernel.rs`: OpenFangKernel struct — the main assembly point
- `registry.rs`: AgentRegistry — in-memory agent management
- `scheduler.rs`: AgentScheduler — cron-based scheduling
- `workflow.rs`: WorkflowEngine — multi-agent orchestration
- `metering.rs`: Cost tracking and budget management

**openfang-api**: HTTP/WebSocket server
- `server.rs`: Route registration, CORS, auth middleware, daemon startup
- `routes.rs`: All 140+ API endpoint handlers (40KB+ file)
- `ws.rs`: WebSocket real-time agent communication
- `openai_compat.rs`: OpenAI-compatible `/v1/chat/completions` endpoint

**openfang-memory**: Persistence layer
- `substrate.rs`: SQLite database manager
- `session.rs`: Canonical session management
- `knowledge.rs`: Knowledge graph storage

### KernelHandle Trait Pattern
The `KernelHandle` trait in `openfang-runtime/kernel_handle.rs` breaks circular dependencies:
- `openfang-runtime` defines the trait
- `openfang-kernel` implements it
- This allows agents to call `spawn_agent()`, `send_to_agent()`, `kill_agent()` without kernel coupling

### AppState Bridge
`AppState` in `server.rs` bridges kernel to API routes:
```rust
pub struct AppState {
    pub kernel: Arc<OpenFangKernel>,
    pub peer_registry: Option<Arc<PeerRegistry>>,  // Note: kernel has Option<PeerRegistry>
    pub bridge_manager: tokio::sync::Mutex<Option<ChannelBridge>>,
    // ...
}
```

### Dashboard (Alpine.js SPA)
Static files in `crates/openfang-api/static/`:
- `index_body.html`: Main SPA with 15 panels (chat, agents, workflows, hands, etc.)
- `js/app.js`: Alpine.js app initialization
- `js/pages/*.js`: Panel-specific logic
- Embedded at compile-time via `include_str!()` for single-binary deployment

## MANDATORY: Live Integration Testing

**After implementing any new endpoint, feature, or wiring change, you MUST run live integration tests.** Unit tests alone are not enough — they can pass while the feature is actually dead code. Live tests catch:
- Missing route registrations in `server.rs`
- Config fields not being deserialized from TOML
- Type mismatches between kernel and API layers
- Endpoints that compile but return wrong/empty data

### How to Run Live Integration Tests

#### Step 1: Stop any running daemon
```bash
# macOS/Linux
pkill -f openfang || true
sleep 3

# Windows (Git Bash/MSYS2)
tasklist | grep -i openfang
taskkill //PID <pid> //F
sleep 3
```

#### Step 2: Build fresh release binary
```bash
cargo build --release -p openfang-cli
```

#### Step 3: Start daemon with required API keys
```bash
GROQ_API_KEY=<key> target/release/openfang start &
sleep 6  # Wait for full boot
curl -s http://127.0.0.1:4200/api/health  # Verify it's up
```

#### Step 4: Test every new endpoint
```bash
# GET endpoints — verify they return real data, not empty/null
curl -s http://127.0.0.1:4200/api/<new-endpoint>

# POST/PUT endpoints — send real payloads
curl -s -X POST http://127.0.0.1:4200/api/<endpoint> \
  -H "Content-Type: application/json" \
  -d '{"field": "value"}'

# Verify write endpoints persist — read back after writing
curl -s -X PUT http://127.0.0.1:4200/api/<endpoint> -d '...'
curl -s http://127.0.0.1:4200/api/<endpoint>  # Should reflect the update
```

#### Step 5: Test real LLM integration
```bash
# Get an agent ID
curl -s http://127.0.0.1:4200/api/agents | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])"

# Send a real message (triggers actual LLM call to Groq/OpenAI)
curl -s -X POST "http://127.0.0.1:4200/api/agents/<id>/message" \
  -H "Content-Type: application/json" \
  -d '{"message": "Say hello in 5 words."}'
```

#### Step 6: Verify side effects
After an LLM call, verify metering/cost/usage tracking:
```bash
curl -s http://127.0.0.1:4200/api/budget       # Cost should have increased
curl -s http://127.0.0.1:4200/api/budget/agents  # Per-agent spend should show
```

#### Step 7: Verify dashboard HTML
```bash
curl -s http://127.0.0.1:4200/ | grep -c "newComponentName"
# Should return > 0 if new UI components exist
```

### Key API Endpoints for Testing
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Basic health check |
| `/api/agents` | GET/POST | List/spawn agents |
| `/api/agents/{id}/message` | POST | Send message (triggers LLM) |
| `/api/budget` | GET/PUT | Global budget status/update |
| `/api/budget/agents` | GET | Per-agent cost ranking |
| `/api/network/status` | GET | OFP network status |
| `/api/peers` | GET | Connected OFP peers |
| `/api/a2a/agents` | GET | External A2A agents |
| `/api/hands` | GET | List autonomous Hands |
| `/api/hands/{id}/activate` | POST | Activate a Hand |

## Development Patterns

### Adding a New API Endpoint
1. Add route in `server.rs` → `build_router()` function
2. Implement handler in `routes.rs`
3. If it needs kernel data, access via `state.kernel.*`
4. Test with `curl` against running daemon

### Adding a New Config Field
1. Add to `KernelConfig` struct in `openfang-types/src/config.rs`
2. Add `#[serde(default)]` attribute
3. Add default value in `Default` impl
4. Add `#[derive(Serialize, Deserialize)]` if needed

### Adding a New Channel Adapter
1. Create `crates/openfang-channels/src/<channel>.rs`
2. Implement `ChannelAdapter` trait
3. Register in `crates/openfang-channels/src/bridge.rs`
4. Add config in `openfang-types/src/config.rs`

### Adding a New Tool
1. Add definition in `openfang-runtime/src/tool_runner.rs` → `builtin_tool_definitions()`
2. Implement execution in the same file (match on tool name)
3. Add tests in the test module at bottom of file

## Common Gotchas
- `openfang.exe` may be locked if daemon is running — use `--lib` flag or kill daemon first
- `PeerRegistry` is `Option<PeerRegistry>` on kernel but `Option<Arc<PeerRegistry>>` on `AppState` — wrap with `.as_ref().map(|r| Arc::new(r.clone()))`
- Config fields added to `KernelConfig` MUST also be added to the `Default` impl or build fails
- `AgentLoopResult` field is `.response` not `.response_text`
- CLI command to start daemon is `start` not `daemon`
- On Windows: use `taskkill //PID <pid> //F` (double slashes in MSYS2/Git Bash)
- **Don't touch `openfang-cli`** — user is actively building the interactive CLI

## Security Notes
- Never log API keys or credentials
- Use `Zeroizing<String>` for sensitive data (auto-wipes on drop)
- All external HTTP requests go through SSRF protection
- WASM sandbox with dual-metering (fuel + epoch) for untrusted code
- Merkle hash-chain audit trail for all agent actions

## Gemini-Specific Notes
- When executing tasks, systematically adhere to these documented paths and verification steps.
- Prioritize using exact internal tool endpoints when possible.
- Use `rust-toolchain.toml` and standard cargo invocations exactly as listed above.

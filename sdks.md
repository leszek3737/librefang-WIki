# SDKs

# LibreFang SDKs

Official language clients for the LibreFang Agent OS REST API. Four implementations provide identical functionality across Go, JavaScript/TypeScript, Python, and Rust.

## Overview

The SDKs enable remote control of a LibreFang server through its REST API. Each client manages HTTP communication, request/response serialization, error handling, and SSE streaming. The client architecture follows a **resource pattern**: related operations are grouped into resource objects attached to the main client.

```
┌─────────────────────────────────────────────────────────────┐
│                    LibreFang Client                          │
├─────────────────────────────────────────────────────────────┤
│  agents    sessions   workflows   skills   channels         │
│  tools     models     providers   memory   triggers         │
│  schedules                                                  │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTP / SSE
                     ▼
        ┌────────────────────────────┐
        │   LibreFang Server API     │
        │   (localhost:4545)         │
        └────────────────────────────┘
```

## Client Initialization

All SDKs follow the same initialization pattern: create a client pointing at the server URL, then access resources through named properties.

**Go**
```go
client := librefang.New("http://localhost:4545")
```

**JavaScript**
```javascript
const { LibreFang } = require("@librefang/sdk");
const client = new LibreFang("http://localhost:4545");
```

**Python**
```python
from librefang import Client
client = Client("http://localhost:4545")
```

**Rust**
```rust
use librefang::LibreFang;
let client = LibreFang::new("http://localhost:4545");
```

## Resources and Operations

### AgentResource

Full agent lifecycle management and messaging. Agents are the primary abstraction in LibreFang—each represents an autonomous agent that can receive messages, execute tools, and maintain conversation state.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/agents | Retrieve all agents |
| `Get` / `get(id)` | GET /api/agents/{id} | Fetch single agent by ID |
| `Create` / `create()` | POST /api/agents | Spawn a new agent |
| `Delete` / `delete(id)` | DELETE /api/agents/{id} | Terminate an agent |
| `Stop` / `stop(id)` | POST /api/agents/{id}/stop | Halt agent execution |
| `Clone` / `clone(id)` | POST /api/agents/{id}/clone | Duplicate an agent |
| `Update` / `update(id, data)` | PUT /api/agents/{id}/update | Modify agent configuration |
| `SetMode` / `set_mode(id, mode)` | PUT /api/agents/{id}/mode | Change agent operating mode |
| `SetModel` / `set_model(id, model)` | PUT /api/agents/{id}/model | Switch the underlying LLM |
| `Message` / `message(id, text)` | POST /api/agents/{id}/message | Send message, wait for full response |
| `Stream` / `stream(id, text)` | POST /api/agents/{id}/message/stream | Send message, receive SSE stream |
| `Session` / `session(id)` | GET /api/agents/{id}/session | Get current session state |
| `ResetSession` / `reset_session(id)` | POST /api/agents/{id}/session/reset | Clear conversation history |
| `ListSessions` / `list_sessions(id)` | GET /api/agents/{id}/sessions | Enumerate agent's sessions |
| `CreateSession` / `create_session(id, label)` | POST /api/agents/{id}/sessions | Create named session |
| `SwitchSession` / `switch_session(id, sessionId)` | POST /api/agents/{id}/sessions/{id}/switch | Change active session |
| `GetSkills` / `get_skills(id)` | GET /api/agents/{id}/skills | List agent's enabled skills |
| `SetSkills` / `set_skills(id, skills)` | PUT /api/agents/{id}/skills | Assign skills to agent |
| `SetIdentity` / `set_identity(id, **identity)` | PATCH /api/agents/{id}/identity | Update agent persona |
| `PatchConfig` / `patch_config(id, **config)` | PATCH /api/agents/{id}/config | Modify runtime config |

**Go example:**
```go
agent, _ := client.Agents.Create(map[string]interface{}{
    "template": "assistant",
})
reply, _ := client.Agents.Message(agent["id"].(string), "Hello!")
```

**JavaScript example:**
```javascript
const agent = await client.agents.create({ template: "assistant" });
const reply = await client.agents.message(agent.id, "Hello!");
```

### SessionResource

Manage conversation sessions globally. Sessions track dialogue history and context for agents.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/sessions | All sessions across agents |
| `Delete` / `delete(id)` | DELETE /api/sessions/{id} | Remove a session |
| `SetLabel` / `set_label(id, label)` | PUT /api/sessions/{id}/label | Name a session |

### WorkflowResource

Execute and manage workflows—composed sequences of agent operations.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/workflows | Available workflows |
| `Create` / `create(**workflow)` | POST /api/workflows | Register a workflow |
| `Run` / `run(id, input)` | POST /api/workflows/{id}/run | Execute workflow |
| `Runs` / `runs(id)` | GET /api/workflows/{id}/runs | History of executions |

### SkillResource

Install and manage capabilities that extend agent functionality.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/skills | Installed skills |
| `Install` / `install(**skill)` | POST /api/skills/install | Add a skill |
| `Uninstall` / `uninstall(**skill)` | POST /api/skills/uninstall | Remove a skill |
| `Search` / `search(query)` | GET /api/marketplace/search | Find marketplace skills |

### ChannelResource

Configure communication channels through which agents interact with external systems.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/channels | Configured channels |
| `Configure` / `configure(name, **config)` | POST /api/channels/{name}/configure | Set channel credentials |
| `Remove` / `remove(name)` | DELETE /api/channels/{name}/configure | Disable a channel |
| `Test` / `test(name)` | POST /api/channels/{name}/test | Verify channel connectivity |

### ProviderResource

Manage LLM provider credentials and test connectivity.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/providers | Configured providers |
| `SetKey` / `set_key(name, key)` | POST /api/providers/{name}/key | Store API key |
| `DeleteKey` / `delete_key(name)` | DELETE /api/providers/{name}/key | Remove API key |
| `Test` / `test(name)` | POST /api/providers/{name}/test | Validate provider access |

### ModelResource

Query available models and their capabilities.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/models | Available models |
| `Get` / `get(id)` | GET /api/models/{id} | Single model details |
| `Aliases` / `aliases()` | GET /api/models/aliases | Model name aliases |

### MemoryResource

Key-value store associated with specific agents. Enables persistent state between sessions.

| Operation | Method | Description |
|-----------|--------|-------------|
| `GetAll` / `get_all(agentId)` | GET /api/memory/agents/{id}/kv | All keys for agent |
| `Get` / `get(agentId, key)` | GET /api/memory/agents/{id}/kv/{key} | Single value |
| `Set` / `set(agentId, key, value)` | PUT /api/memory/agents/{id}/kv/{key} | Store value |
| `Delete` / `delete(agentId, key)` | DELETE /api/memory/agents/{id}/kv/{key} | Remove key |

### TriggerResource

Event-driven automation triggers.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/triggers | All triggers |
| `Create` / `create(**trigger)` | POST /api/triggers | Register trigger |
| `Update` / `update(id, **trigger)` | PUT /api/triggers/{id} | Modify trigger |
| `Delete` / `delete(id)` | DELETE /api/triggers/{id} | Remove trigger |

### ScheduleResource

Time-based task scheduling.

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/schedules | All schedules |
| `Create` / `create(**schedule)` | POST /api/schedules | Register schedule |
| `Update` / `update(id, **schedule)` | PUT /api/schedules/{id} | Modify schedule |
| `Delete` / `delete(id)` | DELETE /api/schedules/{id} | Remove schedule |
| `Run` / `run(id)` | POST /api/schedules/{id}/run | Execute immediately |

### ToolResource

List available tools (read-only).

| Operation | Method | Description |
|-----------|--------|-------------|
| `List` / `list()` | GET /api/tools | Available tools |

## Server Methods

Top-level client methods for system information:

| Operation | Method | Endpoint | Returns |
|-----------|--------|----------|---------|
| `Health` / `health()` | GET | /api/health | Basic health status |
| `HealthDetail` / `health_detail()` | GET | /api/health/detail | Detailed diagnostics |
| `Status` / `status()` | GET | /api/status | Runtime status |
| `Version` / `version()` | GET | /api/version | Server version |
| `Metrics` / `metrics()` | GET | /api/metrics | Prometheus-format metrics |
| `Usage` / `usage()` | GET | /api/usage | Usage statistics |
| `Config` / `config()` | GET | /api/config | Server configuration |

## Streaming Responses

All SDKs support Server-Sent Events (SSE) for streaming agent responses. Streaming yields individual events as they arrive rather than waiting for the complete response.

**Go channel-based approach:**
```go
for event := range client.Agents.Stream(agentID, "Tell me a story") {
    if delta, ok := event["delta"].(string); ok {
        fmt.Print(delta)
    } else if event["type"] == "done" {
        break
    }
}
```

**JavaScript async iterator:**
```javascript
for await (const event of client.agents.stream(id, "Tell me a story")) {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta);
  } else if (event.type === "done") {
    console.log("\n--- Done ---");
  }
}
```

**Python generator:**
```python
for event in client.agents.stream(agent_id, "Tell me a story"):
    if event.get("type") == "text_delta":
        print(event["delta"], end="", flush=True)
```

**Rust future-based:**
```rust
use futures::stream::StreamExt;
let response = client.agents().stream(&agent_id, "Hello").await?;
let mut stream = response.bytes_stream();
while let Some(chunk) = stream.next().await {
    print!("{}", String::from_utf8_lossy(&chunk?));
}
```

Common SSE event types include `text_delta` (incremental text), `tool_call` (function invocation), and `done` (completion signal).

## Error Handling

All SDKs define an error type capturing HTTP status and response body.

**Go:**
```go
resp, err := client.Agents.Message(id, "Hello")
if err != nil {
    if lfErr, ok := err.(*LibreFangError); ok {
        fmt.Println(lfErr.Status, lfErr.Message)
    }
}
```

**JavaScript:**
```javascript
try {
  await client.agents.create({ template: "assistant" });
} catch (e) {
  if (e.name === "LibreFangError") {
    console.error(e.status, e.body);
  }
}
```

**Python:**
```python
try:
    client.agents.create(template="assistant")
except LibreFangError as e:
    print(e.status, e.body)
```

**Rust:**
```rust
match client.agents().list().await {
    Ok(response) => println!("Agents: {}", response.agents.len()),
    Err(Error::Api(msg)) => eprintln!("API error: {}", msg),
    Err(e) => eprintln!("Request failed: {}", e),
}
```

## Python Agent SDK

The Python package includes a distinct component for **writing agents that run inside LibreFang**, separate from the REST client. This is the `librefang_sdk` module.

```
┌─────────────────┐     ┌─────────────────┐
│  LibreFang      │     │  Your Python    │
│  Kernel         │────▶│  Agent Code     │
│  (orchestrator)  │◀────│  (librefang_sdk)│
└─────────────────┘     └─────────────────┘
     stdin/stdout JSON protocol
```

Agents receive messages via stdin and respond via stdout using a simple JSON protocol:

```python
from librefang_sdk import Agent, read_input, respond, log

# Decorator-based approach
agent = Agent()

@agent.on_message
def handle(message: str, context: dict) -> str:
    return f"You said: {message}"

@agent.on_setup
def init():
    log("Agent initializing...")

@agent.on_teardown
def cleanup():
    log("Agent shutting down...")

agent.run()

# Or procedural approach
data = read_input()        # Read from stdin
result = f"Echo: {data['message']}"
respond(result)            # Write to stdout
```

The `read_input()` function parses the JSON input from the LibreFang kernel, containing `message`, `agent_id`, and `context` fields. The `respond()` function sends the response back. Logging writes to stderr, visible in daemon logs.

## SDK Feature Matrix

| Feature | Go | JavaScript | Python | Rust |
|---------|----|------------|--------|------|
| REST client | ✓ | ✓ | ✓ | ✓ |
| SSE streaming | ✓ | ✓ | ✓ | ✓ |
| All resources | ✓ | ✓ | ✓ | Partial* |
| TypeScript types | — | ✓ | — | — |
| Zero deps (stdlib) | — | — | ✓ | — |
| Python agent SDK | — | — | ✓ | — |

*Rust SDK currently implements Agents, Skills, Models, and Providers resources.

## Internal Usage

These SDKs are consumed throughout the LibreFang codebase:

- **Dashboard** (`dashboard/src/`) uses the JavaScript SDK for all API calls, including channel configuration, agent management, and approval workflows
- **TUI** (`src/tui/event.rs`) uses the Go SDK for spawning daemon streams and terminal connections
- **MCP Server** (`librefang-runtime/src/mcp.rs`) uses the Python client to configure agents
- **Telegram Channel** (`librefang-channels/src/telegram.rs`) uses the Python streaming client
- **WhatsApp Gateway** (`packages/whatsapp-gateway/`) uses the JavaScript SDK for agent communication

## Installation

**Go:**
```bash
go get github.com/librefang/librefang/sdk/go
```

**JavaScript:**
```bash
npm install @librefang/sdk
```

**Python:**
```bash
pip install librefang
```

**Rust:** Add to `Cargo.toml`:
```toml
[dependencies]
librefang = "2026.4"
tokio = { version = "1", features = ["full"] }
```
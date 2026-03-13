# Learning Path: Build a Simplified Code Agent

This document summarizes a practical learning path for understanding this repository and building a simplified code agent yourself.

## Goal

The target is not to reproduce the full Codex project. The target is to understand the core architecture well enough to implement a smaller version with:

- a user prompt
- a model call
- a tool loop
- basic file reading/writing
- shell execution
- minimal safety boundaries

## How To Read This Repository

The most important idea in this codebase is that Codex is an event-driven agent runtime, not just a CLI wrapper around model calls.

The most useful crates and files for learning are:

- [`codex-rs/docs/protocol_v1.md`](../codex-rs/docs/protocol_v1.md)
- [`codex-rs/protocol/src/protocol.rs`](../codex-rs/protocol/src/protocol.rs)
- [`codex-rs/core/src/lib.rs`](../codex-rs/core/src/lib.rs)
- [`codex-rs/core/src/codex.rs`](../codex-rs/core/src/codex.rs)
- [`codex-rs/core/src/client_common.rs`](../codex-rs/core/src/client_common.rs)
- [`codex-rs/core/src/tools/registry.rs`](../codex-rs/core/src/tools/registry.rs)
- [`codex-rs/core/src/tools/handlers/mod.rs`](../codex-rs/core/src/tools/handlers/mod.rs)
- [`codex-rs/core/src/tools/handlers/shell.rs`](../codex-rs/core/src/tools/handlers/shell.rs)
- [`codex-rs/exec/src/main.rs`](../codex-rs/exec/src/main.rs)

## A Simple Mental Model

You can treat the current project as five layers:

1. `protocol`
Defines how the UI and agent communicate through `Submission(Op)` and `Event(EventMsg)`.

2. `core`
Implements the actual agent runtime: session state, prompt building, model interaction, tool execution, turn loop, and event emission.

3. `tools`
Provides the tool abstraction, registration, dispatch, execution, and permission logic.

4. `exec`
Provides a headless entrypoint that is much easier to study than the TUI.

5. `tui` / `cli`
Provide user-facing shells around the core engine.

For learning, focus on `protocol -> core -> tools -> exec` first.

## Core Runtime Flow

The core runtime loop can be reduced to this:

```text
1. User submits a task
2. Agent builds prompt and available tool definitions
3. Model returns either:
   - final text
   - tool call(s)
4. Agent executes the tool
5. Tool output is appended back into conversation history
6. Agent calls the model again
7. Loop ends when the model returns a final answer
```

This is the essential loop you should reproduce in a simplified implementation.

## Minimal Architecture For Your Own Project

Use this as the first version of your simplified code agent:

```text
User/CLI
  -> Agent Runner
    -> Session State
    -> Prompt Builder
    -> Model Client
    -> Tool Registry
      -> read_file
      -> shell
      -> write_file or apply_patch
    -> Event Stream / Logger
```

## Minimal Module Breakdown

A small implementation only needs these modules:

### `main.rs`

Responsibilities:

- read user input
- initialize the agent
- print progress and final output

### `agent.rs`

Responsibilities:

- run the main agent loop
- handle model responses
- dispatch tool calls
- feed tool results back into history

Suggested pseudocode:

```rust
loop {
    let response = model_client.complete(history, tools)?;
    match response {
        FinalMessage(msg) => return Ok(msg),
        ToolCall(call) => {
            let output = tool_registry.execute(call)?;
            history.push(tool_output_as_message(output));
        }
    }
}
```

### `model_client.rs`

Responsibilities:

- wrap the LLM API
- translate provider output into your internal response type

Suggested shape:

```rust
enum ModelResponse {
    FinalMessage(String),
    ToolCall(ToolCall),
}
```

### `prompt.rs`

Responsibilities:

- build the system prompt
- include history
- include available tools

Do not implement advanced context compaction in the first version.

### `tools/mod.rs`

Responsibilities:

- define a common tool interface
- register tools
- dispatch by tool name

Suggested trait:

```rust
trait Tool {
    fn name(&self) -> &str;
    fn schema(&self) -> serde_json::Value;
    fn execute(&self, input: serde_json::Value) -> anyhow::Result<String>;
}
```

### `session.rs`

Responsibilities:

- store cwd
- store message history
- store run configuration

Suggested shape:

```rust
struct Session {
    cwd: PathBuf,
    history: Vec<Message>,
}
```

## Minimal Data Structures

Keep the first version simple:

```rust
enum Message {
    System(String),
    User(String),
    Assistant(String),
    ToolCall {
        id: String,
        tool_name: String,
        args: serde_json::Value,
    },
    ToolResult {
        id: String,
        tool_name: String,
        content: String,
    },
}
```

```rust
struct ToolCall {
    id: String,
    name: String,
    arguments: serde_json::Value,
}
```

## The First Three Tools To Implement

These are enough to make the agent useful:

### `read_file`

Inputs:

- `path`

Output:

- file content

### `shell`

Inputs:

- `command`
- optional `workdir`

Output:

- stdout
- stderr
- exit code

### `write_file` or `apply_patch`

Inputs:

- `path`
- `content`

Or:

- patch text

Output:

- success or failure

If you want the simplest route, implement `write_file` before `apply_patch`.

## Recommended Development Order

Build your simplified agent in these stages:

### Stage 1: Plain Chat

Goal:

- user enters text
- model returns text
- no tools

### Stage 2: Tool Calling

Goal:

- define tool schemas
- detect tool calls from model output
- dispatch tools
- continue the loop

### Stage 3: Read Existing Code

Goal:

- add `read_file`
- allow the agent to inspect local source files

### Stage 4: Execute Commands

Goal:

- add `shell`
- allow the agent to run safe local commands such as `ls`, `rg`, or test commands

### Stage 5: Modify Code

Goal:

- add `write_file` or `apply_patch`
- allow the agent to produce local code changes

### Stage 6: Add Guardrails

Goal:

- limit execution to a workspace directory
- add command timeout
- block obviously dangerous commands
- add loop limits

## What To Learn From This Repository

The most important patterns to extract from this codebase are:

### 1. Protocol-first design

Study:

- [`codex-rs/docs/protocol_v1.md`](../codex-rs/docs/protocol_v1.md)
- [`codex-rs/protocol/src/protocol.rs`](../codex-rs/protocol/src/protocol.rs)

Why it matters:

- the agent runtime becomes reusable
- the UI becomes a thin shell
- event streaming stays explicit

### 2. Core/runtime separated from UI

Study:

- [`codex-rs/core/src/lib.rs`](../codex-rs/core/src/lib.rs)
- [`codex-rs/exec/src/main.rs`](../codex-rs/exec/src/main.rs)

Why it matters:

- your simplified version should also keep the agent engine separate from the CLI

### 3. A unified tool registry

Study:

- [`codex-rs/core/src/tools/registry.rs`](../codex-rs/core/src/tools/registry.rs)
- [`codex-rs/core/src/tools/handlers/mod.rs`](../codex-rs/core/src/tools/handlers/mod.rs)

Why it matters:

- you need one place to register, validate, and dispatch tool calls

### 4. Shell as a normal tool

Study:

- [`codex-rs/core/src/tools/handlers/shell.rs`](../codex-rs/core/src/tools/handlers/shell.rs)

Why it matters:

- shell execution should be isolated behind a tool boundary, not spread across the codebase

### 5. Prompt-building as a first-class concern

Study:

- [`codex-rs/core/src/client_common.rs`](../codex-rs/core/src/client_common.rs)

Why it matters:

- tools, history, and output formatting all affect model behavior

## Reading Order Inside This Repository

If you want a practical reading sequence, use this:

1. [`codex-rs/docs/protocol_v1.md`](../codex-rs/docs/protocol_v1.md)
2. [`codex-rs/protocol/src/protocol.rs`](../codex-rs/protocol/src/protocol.rs)
3. [`codex-rs/core/src/lib.rs`](../codex-rs/core/src/lib.rs)
4. [`codex-rs/core/src/codex.rs`](../codex-rs/core/src/codex.rs)
5. [`codex-rs/core/src/client_common.rs`](../codex-rs/core/src/client_common.rs)
6. [`codex-rs/core/src/tools/registry.rs`](../codex-rs/core/src/tools/registry.rs)
7. [`codex-rs/core/src/tools/handlers/shell.rs`](../codex-rs/core/src/tools/handlers/shell.rs)
8. [`codex-rs/exec/src/main.rs`](../codex-rs/exec/src/main.rs)

## What To Skip At First

Do not start with these areas if your goal is a simplified code agent:

- `tui`
- `memories`
- `mcp`
- `plugins`
- `realtime_conversation`
- advanced sandbox internals
- platform-specific hardening details

These are product-scale concerns, not minimum viable agent concerns.

## A Realistic Six-Week Learning Path

If you already have basic development experience and can spend regular time each week:

### Week 1

- read the protocol docs
- trace one full request through `exec -> core -> tools`
- draw your own sequence diagram

### Week 2

- implement a plain chat-only prototype

### Week 3

- add a tool registry
- implement `read_file`

### Week 4

- implement `shell`
- support iterative tool/model loops

### Week 5

- implement `write_file` or `apply_patch`
- finish a simple code-edit task end to end

### Week 6

- add guardrails
- revisit this repository and compare your design against the production architecture

## The Four Questions To Keep Asking

Whenever you study a module in this repository, ask:

1. Where is the runtime state stored?
2. Who decides whether the model output is final text or a tool call?
3. How is tool output fed back into the next model turn?
4. Where are the safety boundaries enforced?

## Final Recommendation

Use this repository as an architecture reference, not as a template to clone line-by-line.

The highest-value subset to understand is:

`protocol -> core loop -> tool registry -> shell handler -> exec entry`

Once that mental model is solid, implement a version that is roughly ten times smaller than this codebase. That is the fastest path to actually building a working simplified code agent yourself.

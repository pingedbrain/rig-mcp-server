> **Moved.** `rig-mcp-server` now lives in [pingedbrain/rig-contrib](https://github.com/pingedbrain/rig-contrib/tree/main/crates/rig-mcp-server), together with the other Rig companion crates. This repository is archived.

# rig-mcp-server

Serve [Rig](https://github.com/0xPlaygrounds/rig) agents and tools over the
[Model Context Protocol](https://modelcontextprotocol.io), so any MCP client can use them:
Claude Code, Cursor, Zed, Claude Desktop, or another agent. Built on the official MCP
Rust SDK, [`rmcp`](https://crates.io/crates/rmcp).

Rig ships an MCP *client* (`rig-rmcp`, Rig agents using MCP tools). This crate is the
other direction: Rig agents *as* MCP tools.

```rust
use rig_mcp_server::McpServer;

McpServer::new("my-agents")
    .agent(research_agent)            // a Rig agent, as a tool
    .typed_agent::<Ticket>(triager)   // answers with structured content
    .tool(Calculate)                  // a plain Rig tool, no model
    .serve_stdio()                    // for a client that launches this binary
    .await?;
```

## What a client sees

| Exposed | As an MCP tool |
|---|---|
| `agent(agent)` | Named after the agent (made provider-safe), described by its description. Takes `message` and an optional `session_id`. Answers with its text and any images it made, plus `{answer, session_id}` as structured content; pass the `session_id` back to continue the conversation. |
| `typed_agent::<T>(agent)` | Same input; runs as `prompt_typed::<T>` and answers with the `T` as structured content, declared by the tool's `outputSchema` (the JSON schema of `T`). |
| `tool(t)` / `dynamic_tool(t)` | The Rig tool as itself: name, description, parameters and annotations (`readOnlyHint`, `destructiveHint`, ...). Runs without a model; its failure is an error result the model reads. A lone JSON object it returns is also structured content. |

While an agent writes, its text streams to the client as **progress notifications**
(when the client asks for progress). When the client **cancels** a call, the agent run
stops.

## Transports

```rust
// stdio: a local client launches the binary. Stdout carries the protocol,
// so log to stderr only.
server.serve_stdio().await?;

// Streamable HTTP at /mcp, as an Axum router.
let app = server.bearer_token(token).into_router();
axum::serve(listener, app).await?;

// Any rmcp transport (an in-memory duplex in tests, a socket...).
server.serve(transport).await?;
```

Over HTTP the endpoint answers only to `localhost`, `127.0.0.1` and `::1` by default,
which keeps a local server safe from DNS rebinding; set `allowed_hosts` to the public
host name when deploying. With `bearer_token`, calls without the token get HTTP 401.

### In Claude Code

```sh
claude mcp add my-agents -- /path/to/your/binary
```

## Sessions and limits

Agent sessions live in memory by default (at most 1000, oldest forgotten first), or in any
[`rig-memory`](https://crates.io/crates/rig-memory) store with `history_store` (`DirStore`
on disk, `rig-sqlite`, `rig-postgres`), so they survive restarts. Each call sends the
model only the newest messages that fit `history_window` (32k tokens by default). A
`session_id` must be 1-128 of `[A-Za-z0-9_-]`.

Every call to an agent spends model tokens, so a server bounds them:

| Knob | Default | Beyond it |
|---|---|---|
| `max_concurrent_calls` | 32 | an error result saying the server is busy, before any model call |
| `max_run_duration` | 10 min | the run is stopped, an error result |
| `bearer_token` (HTTP) | none | HTTP 401 |

Each agent call runs in an `mcp.agent_call` tracing span (with `mcp.session_id`); Rig's
spans for the model calls and tools nest under it.

## Testing

`cargo test` drives the server with the official `rmcp` client in memory: listing,
schemas and annotations, plain tools, agent sessions, streamed progress, typed answers,
cancellation, time limits, HTTP with and without the token, and a Rig agent using the
served tools through `rig-rmcp` (Rig to MCP to Rig).

`interop/run.sh` checks the server against an independent implementation, the official
MCP Python SDK, over stdio and Streamable HTTP, keyless (`cargo run --example
echo_server`). CI runs both, plus MSRV and `cargo-deny`.

## Not supported

MCP resources, prompts, sampling and elicitation; the SEP-2663 tasks extension
(long calls run to completion, reporting progress); OAuth (bearer only).

# SSE Had a Good Run. It's Time to Let Go.

*Why long-lived connections are the wrong abstraction for agent-to-tool communication*

---

## Wait — Isn't MCP Just a Fancy CLI?

Before we talk transport, let's kill this confusion. MCP gets compared to CLI tools all the time: "Why not just shell out to a script?" Here's the difference:

| | CLI Tool | MCP Server |
|---|---------|-----------|
| Discovery | You read the `--help` flag | Client calls `tools/list` at runtime — gets names, schemas, descriptions. No docs needed. |
| Schema | Freeform strings in, freeform strings out | JSON Schema for every parameter. The LLM knows exactly what to send. |
| State | Stateless. Every call starts from zero. | Server holds session state — purchase history, learned preferences, context across calls. |
| Lifecycle | Process starts, runs, exits | Server stays up, serves many clients, maintains connections to databases and APIs. |
| Composability | Pipe `stdout` to the next tool and pray | Structured JSON-RPC. Client can call 8 tools on the same server in sequence, each building on the last. |

A CLI tool is a hammer. An MCP server is a workshop with a toolbelt, a workbench, and a memory of every project you've done. The LLM doesn't need to parse `--help` output or guess at argument formats — it gets a typed contract. And the server remembers what happened last time.

Could you wrap every CLI tool in a shell script and make it work? Sure — the same way you could serve a website with `netcat`. The question isn't "can it work" but "does it scale to 8 tools, 3 agents, and a conversation that spans 15 minutes?"

## The Setup

MCP (Model Context Protocol) started with two transport options: `stdio` for local processes and `SSE` (Server-Sent Events) for network communication. The idea was simple — keep a persistent HTTP connection open, and the server pushes events down the stream as they happen.

In a demo, this looks elegant. In production, it falls apart.

## What Goes Wrong with Persistent SSE

The original MCP transport used a dual-endpoint model: one long-lived SSE pipe for the session, separate POSTs for client messages. That persistent connection is where the trouble starts. Every piece of infrastructure between your client and server has opinions about how long it should live:

**Proxy timeouts.** Nginx, AWS ALB, Cloudflare, GCP LB, Vercel — all enforce idle timeouts, typically 60–120 seconds (sometimes 30s). A quiet MCP session gets killed silently. Your client doesn't know until the next read fails. No amount of "just add heartbeats" fixes the fundamental mismatch.

**TCP/IP betrayal.** The application layer thinks the connection is fine. But a middlebox has already reaped the underlying TCP socket. The connection is a ghost — alive in software, dead on the wire.

**No resumability.** When the connection drops (and it will), you lose the event stream. SSE has `Last-Event-ID` in the spec, but most MCP implementations don't implement it properly. You reconnect blind.

**Scaling pressure.** Each SSE connection pins a server thread or process. A thousand connected agents means a thousand held connections. That's the opposite of how HTTP was designed to work — and it fights serverless, edge compute, and autoscaling.

## The Fix: Stateless HTTP (With a Nuance)

The pattern that actually works at scale is the one the rest of the internet already uses for async operations:

```
Client → POST /task
Server → 202 Accepted
         Location: /task/abc-123
         Retry-After: 5              ← server tells you when to come back

Client → GET /task/abc-123           ← not before 5 seconds
Server → 200 OK  { "status": "complete", "result": {...} }
```

No guessing. No wasteful 2-second polling loops. The server knows how long the work takes — it tells you via `Retry-After`. The client sleeps, wakes up, asks once. Connection drops don't matter — you just call back. Load balancers route each request independently. No pinned threads, no ghost sockets, no silent timeouts.

**But pure polling isn't the whole answer.** For a 30-second tool call, `Retry-After` is perfect. For a multi-turn conversation where a remote agent is streaming thoughts in real time, you don't want to wait. The winning pattern is what both updated MCP and A2A actually ship: **POST + optional upgrade to SSE stream for that specific response**, plus `Retry-After`-driven polling or webhooks for truly long-running work (>1 min).

The key distinction: **scoped, per-request SSE = good. Session-long persistent connection = bad.** SSE isn't dead — it's demoted to the right role. And when you do poll, let the server drive the cadence with `Retry-After` — don't guess.

## MCP Caught Up

The MCP spec authors reached the same conclusion. The transport evolution tells the story:

| Version | Transports | What Changed |
|---------|-----------|--------------|
| Nov 2024 | stdio + SSE | Original spec. Dual-endpoint: persistent SSE pipe + separate POSTs. |
| Mar 2025 | stdio + Streamable HTTP | **Persistent SSE deprecated.** Single `/mcp` POST endpoint. Server can respond with JSON or upgrade to per-request SSE stream, then close. |

Streamable HTTP is the HTTP 202 pattern with an optional streaming upgrade per-request. No persistent connection required. Each call is independent, stateless, and proxy-friendly. The server still *uses* `text/event-stream` when it needs to stream — but scoped to one request, not one session.

## When to Use What

| Transport | Use Case | Trade-off |
|-----------|----------|-----------|
| `stdio` | Local dev, notebooks, CLI tools | No network. Fast. Single-machine only. Still king. |
| `SSE` (legacy) | Quick demos over a network | Simple, but fragile at scale. Deprecated. |
| **`Streamable HTTP`** | **Production deployment** | **Robust, proxy-friendly, recoverable. MCP's current standard.** |
| `HTTP 202 + Retry-After` | Long-running agent tasks (>60s) | Server-driven cadence. Best for A2A where conversations take minutes. |

## The A2A Connection

This matters even more for A2A (Agent-to-Agent Protocol). When one agent delegates to another, the conversation can take minutes, not milliseconds. You cannot hold a connection open for that. Google's A2A spec was born with this understanding — it uses `POST /task` → `202 Accepted` + `Retry-After` → poll (`tasks/get`) on the server's schedule, subscribe to a per-task SSE stream (`message/stream`), or register a webhook from day one. The server drives the timing, not the client.

MCP started with the old pattern and evolved. A2A started with the right one.

## If You're Building This Today

- **Local/dev/notebooks/CLI** → `stdio` (still king)
- **Remote/production** → **Streamable HTTP** (MCP's current standard). Use the single `/mcp` POST endpoint. Let the server upgrade to SSE only when it needs to stream.
- **Long-running agent delegations (>60s)** → task ID + `Retry-After`-driven polling or webhook (A2A style). Let the server tell you when to come back. Don't hold the connection.
- **Maintaining legacy MCP servers?** → Keep the backwards-compat layer for now, but default new deploys to Streamable HTTP and plan to drop old SSE.

---

## The Bottom Line

Long-lived connections are a 90s pattern that fights against modern infrastructure — load balancers, CDNs, serverless, edge compute. The fix isn't "ban SSE" — it's scope it properly. Per-request streaming is efficient and low-overhead. Session-long persistent pipes are technical debt.

MCP got there. A2A was born there. The evidence is in the spec changelog. If you're still shipping new agent tooling on persistent SSE as the primary transport in 2026, you're building technical debt on purpose.

Start with Streamable HTTP. Don't look back.

`#MCP` `#A2A` `#AgenticAI` `#ProtocolDesign` `#SSE` `#HTTP202` `#GoogleADK`

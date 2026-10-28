# How Claude Selects Tools — The Role of Descriptions
- schema you provided: the name, the description, the parameters.
- Claude selects tools by semantically matching the task against the tool description
#  Writing Tool Descriptions That Route Correctly
- Describe what it does, what it takes, and what it returns — not how it works
- the description is a routing contract written for a semantic matcher.
- Problem : If two tools could both plausibly match the same task, the model has no reliable basis to choose.

# Designing Tool Input Schemas for Precision and Reliability
- Once Claude picks a tool, it has to fill in the arguments — and it generates those arguments from the schema you defined
- A loose schema invites loose inputs
- input schema is your primary lever for making tool calls precise and reliable
# Required vs. Optional Parameters, Enums, and Constraints
- A required parameter must be present for the call to be valid — the model cannot omit it, which forces it to gather that data before calling
- An optional parameter can be left out, and the tool applies a default or adjusts behavior.
- Enums. When a field accepts only a fixed set of values, enumerate them.
- Constraints. Beyond type and enum, constraints narrow a field further: numeric ranges (min/max), string patterns/formats, length limits, and so on
- Over-requiring vs. under-requiring. A wrong-answer pattern is "make every field required to be safe." That causes hallucinated values when data isn't available. The correct principle: require only what's truly essential, make the rest optional. Both extremes are failures — the exam rewards matching required/optional to the tool's actual contract.
- Enum vs. free-string for fixed sets. If a scenario shows a tool receiving inconsistent or invented category/status values, the fix is an enum, not a better field description. Descriptions guide; enums enforce. When the value set is known and closed, enumerate it — that's the deterministic fix, echoing the "code enforces, prompts guide" theme from Domain 1.
# Strict Tool Use and Valid Inputs
- Strict tool use is the mechanism that constrains the model's output so tool calls are guaranteed to conform to the schema — valid types, required fields present, enum values respected —
- It guarantees structural validity — the arguments match the schema's types, required fields are present, enum values are from the allowed set, formats are respected.
- It does not guarantee semantic correctness — that the right values were chosen.
- gurantee call to tool is success with proper schema

# Returning Useful Errors from Tools
- It's an observation that goes straight back into the model's context, and the model plans its next step from it. If the error is opaque — a bare 500, a stack trace, a generic "Error: failed" — the model has nothing actionable to reason with
- It can't tell whether to retry, fix an argument, try a different tool, or escalat
- A useful tool error is one the model can act on
- kind of falire 
  - a transient failure (timeout, service unavailable) → signals retry may work;
  - input failure (bad parameter, validation error) → signals fix the argument and retry differently, not repeat identically;
  - a not-found / state failure (no such record) → signals the premise was wrong; replan or escalate, retrying is futile;
  - a permission failure → signals escalate, retrying won't grant access.
- : an error is a communication channel to the model, and its job is to enable correct recovery.

# Tool Retry Logic and Partial Success Patterns
- the agent may retry the whole operation — re-processing the 8 that already worked, re-committing side effects, double-charging, duplicating records.
- Partial success handled as total failure is how retries cause corruption, not just wast
- Some retry decisions belong inside the tool, not in the agent loop.
- A tool that wraps a flaky network call can retry transient failures internally — with bounded attempts and backoff — and only surface an error to the agent if those internal retries are exhausted.
- the tool handles low-level transient retries (network blips, brief unavailability) transparently; the agent handles higher-level recovery (wrong approach, replanning, escalation).
- Push transient-failure retries down into the tool so the agent only sees failures that genuinely need its judgmen
- the tool must report partial success explicitly rather than collapsing it to success-or-failure.
- A partial-success response tells the agent three things: what succeeded, what failed, and — critically — what is safe to retr
- idempotent operation can be repeated without changing the result beyond the first application — retrying it is harmless.
- retry must be scoped to what's actually safe and necessary to redo.
- Partial-success + idempotency (the good design): the tool returns:
# Tool Scoping per Agent — Why Less Is More
- selection reliability degrades as the toolset grows: more tools means more descriptions to disambiguate, more chances for overlap, more opportunities for the model to pick the wrong one.
- Tool scoping is the practice of giving each agent only the tools its role actually requires — the least-privilege principle
# Selection Heuristics and Tool-Selection Failures
- It gives you (a) the heuristics Claude uses to select, and (b) a taxonomy of how selection fails, so you can map symptom → cause → fix.
- map the symptom to the lever.  
  - Skipped tool → description didn't match (or missing context). 
  - Wrong sibling → descriptions overlap. 
  - Right tool, bad arguments → schema (not a selection problem at all). 
  - Selection degrades at scale → too many tools.

  # Controlling Tool Selection
  - You might need to force a tool call when the model keeps answering from its own knowledge instead of using the tool. You might need to forbid tool use and force a text answer. You might need to require one specific tool at a decision point where only that action is valid.
  - tool_choice forces an outcome on a specific turn (deterministic, situational). If a scenario needs a guaranteed action on a critical step

  # Tool Search for Large Catalogs
  - . Tool search (tool discovery) decouples how many tools exist from how many the model sees at once. Instead of loading the entire catalog into context up front, the relevant tools are retrieved on demand based on the current task, and only that small, relevant subset is presented to the model for selection. The full catalog can be huge; the model's working set stays small.
  - : tool search is the answer to catalog scale, not to per-tool quality
  - Tool search vs. tool scoping.
     - Scoping is static and role-based: this agent only ever gets these tools (a permission/least-privilege decision). Tool search is dynamic and task-based: from the available tools, retrieve the ones relevant to this request right now (a scale/retrieval decision)

# What MCP Is and Why It Exists
- the Model Context Protocol — is an open standard for connecting AI applications to external tools, data, and prompts
- If you wanted Claude to reach your database, you wrote custom glue for your app. If a colleague wanted the same database in their app, they wrote their own glue again. Every model-to-tool connection was a one-off, built for one client, non-reusable and non-portable.
- Standardization and reusability. Because the protocol is uniform, servers are portable and shareable.
- Separation of concerns. The team that owns a data source can build and maintain its MCP server, exposing exactly what it wants to expose, independent of whoever consumes it.
- Instead of every application building custom integrations to every tool, each tool is exposed once as an MCP server, and each application implements the MCP client once. Then any client can use any server.
- An MCP server exposes capabilities — tools, resources, and prompts (the three primitives, next topic) — from some system (a database, an API, a filesystem)
- An MCP client lives inside the AI application (the "host") and connects to servers to consume those capabilities.
- MCP servers at once. Communication flows over a defined transport (STDIO or StreamableHTTP — a later topic), us

# Tools, Resources, and Prompts — The Three MCP Primitives
- MCP exposes tool, resources, prompt 
- Resources — application-controlled. Data/content the server exposes for the model to read as context — a file's contents, a database record, a document, a log.
- Prompts — user-controlled. Pre-defined prompt templates the server exposes, typically user-controlled — surfaced for a user to invoke deliberately (e.g., as a slash command or menu item in the host UI).
- "The model decides the tool call, and the MCP server executes the call." ✅

# Building an MCP Server — Defining and Exposing Tools, Resources, and Prompt Templates
- It gives you the anatomy of an MCP server and the declaration pattern for each primitive.

Server anatomy. An MCP server, at minimum:

- Instantiates a server with an identity (a name, so clients know what they've connected to). Your notes use the Python SDK, where a high-level server object (FastMCP-style) handles most of the protocol plumbing for you.
- Declares its capabilities — which primitives it offers (tools, resources, prompts). This declaration is part of the MCP handshake: when a client connects, the server advertises what it exposes so the client knows what's available. Capability advertisement is how the client discovers the server's offerings.
- Connects a transport — STDIO or StreamableHTTP (next topic) — over which it will receive requests and send responses.
- Runs, waiting for client requests and responding. Remember from the last exchange: the server responds, it never initiates.

# Implementing MCP Clients and Connecting to Servers
- The client is the component inside the host that manages the connection to a server and mediates between the model and that server. Its responsibilities:
- Establish the connection. The client connects to a server over a chosen transport (STDIO or StreamableHTTP). For a local STDIO server, connecting means the client launches the server process and talks to it over stdin/stdout. For a remote HTTP server, it connects to a URL. Either way the client initiates and owns the session.
- Perform the handshake and discover capabilities. On connect, client and server negotiate (protocol version, capabilities), and the client discovers what the server exposes — it asks the server to list its tools, resources, and prompts. This discovery step is how the host learns what's available.
- The client takes the discovered tools and surfaces them into the model's available toolset — so from the model's point of view, an MCP tool looks like any other tool it can call (Cluster A).
- It makes resources available for the host to load as context, and prompts available for the user to invoke. The client is the bridge that turns "the server has these" into "the model/app/user can use these
- Relay calls and return results. This is the round trip from earlier. When the model emits a tool call, the client routes it to the correct server, waits for the server to execute, and returns the result to the host, which injects it into context. The client is the message broker in both directions.
- Client vs. host vs. server (again, because it's tested relentlessly). The host is the AI application. The client is the connector object inside the host that owns one server connection. The server exposes primitives. One host → many clients → one server each

# Transports — STDIO vs. StreamableHTTP
- MCP standardizes two transports, and each maps to a distinct deployment model.
- STDIO (standard input/output). The client and server run on the same machine, communicating over the process's stdin/stdout streams.
  - Local only — server and client are co-located on one machine. No network involved.
  - Simple and low-latency — it's inter-process communication on the same host, no network hop, no HTTP overhead.
  - Launched by the client — the server's lifecycle is tied to the client; the client starts the subprocess and shuts it down.
  - Security profile is inherently narrower — since there's no network exposure, the attack surface is the local machine, not the internet. But it also means no remote access.
  - Best for: local tools, desktop applications, developer tooling, personal integrations — anything where the server lives on the same box as the host (e.g., a filesystem server for Claude Desktop, a local dev tool).

- StreamableHTTP. The client and server communicate over HTTP, so the server can be remote — running on a different machine, a server, a cloud deployment. The defining characteristic: it's a network transport built for remote, production servers. Properties:

  - Remote-capable — the server runs independently, anywhere reachable over HTTP; many clients can connect to one deployed server.
    - Production deployment model — this is the transport for a server you host as a service: independently deployed, scalable, shareable across users/apps.
Supports streaming — as the name says, it can stream responses (relevant for long-running operations and progress).
  - Requires the full network security stack — because it's exposed over HTTP, it needs authentication, authorization, transport security (TLS) — all the concerns of any networked service. This is where the auth/versioning/reliability material (a later topic) lands.

- Who launches the server. STDIO servers are launched by the client as subprocesses (lifecycle tied to client). StreamableHTTP servers run independently and are connected to. If a scenario says the client starts/stops the server process, that's STDIO; if the server is a standing service clients dial into, that's StreamableHTTP.
Best for: remote/hosted MCP servers, multi-user services, anything deployed as production infrastructure.
- STDIO vs. StreamableHTTP by deployment. The single most common transport question: given a scenario, pick the transport. Tells: "local," "desktop," "same machine," "developer's laptop," "personal" → STDIO. "Remote," "hosted," "production," "multiple users," "deployed service," "cloud" → StreamableHTTP. Match on locality and production-ness, not on features.

# Sampling — Letting Servers Make LLM Calls Through the Client
- Sampling is the MCP capability that lets a server request an LLM completion from the client — the server asks the host's model to generate something on its behalf, rather than the server calling a model itself. The direction is inverted: instead of host→server (a tool call), this is server→host (a request for inference), routed through the client.
- Why route it through the client instead of letting the server call a model directly
  - No credentials in the server.
  - The host stays in control. Because the completion request passes through the client, the host controls it — it can inspect, approve, modify, or reject the sampling request, choose which model handles it, and apply its own rules
  - Centralized model choice and cost. The host decides which model, at what cost, under what limits — uniformly — rather than each server making its own independent model decisions.
  - Capability without coupling. Servers can incorporate LLM-powered behavior without being coupled to any particular model provider or holding provider credentials. They just express "I need a completion for this," and the host fulfills it.
  - The MCP-correct answer is sampling, because it keeps credentials out of the server and control at the host (inspect/approve, model choice, centralized cost). If a scenario stresses governance, security, or control over LLM usage, sampling-through-the-client is the point.

# Notifications, Roots, and Progress Reporting
- Progress reporting. For long-running operations, the server can send progress updates to the client during execution, rather than going silent until it's done. This lets the host surface "40% complete…" to the user and confirms the operation is alive, not stalled. The mechanism: the client includes a progress token with a request, and the server sends progress notifications referencing that token as the work advances. The purpose to remember: visibility into long-running work — feedback during execution, not just a final result. This pairs naturally with StreamableHTTP and long operations (and with sampling, which can be slow).

- Notifications. MCP supports asynchronous, one-way messages that don't expect a response — distinct from the request/response round trip. The most important kind for the exam is list-changed notifications: when a server's available tools (or resources, or prompts) change while connected, it notifies the client so the client can re-discover the updated list. This keeps the client's picture of the server's capabilities current dynamically, without tearing down and rebuilding the connection.
- Roots. Roots are how the client tells the server which parts of a resource space (typically the filesystem) it's allowed to operate within — a set of boundary URIs/paths defining the server's permitted scope.
- The principle tying them together: these three features handle the dynamics and boundaries of a live MCP session that the basic request/response can't. Progress = staying informed during long work. Notifications = staying in sync as capabilities change. Roots = staying inside sanctioned boundaries. Each fills a gap the simple call/response model leaves open
- Roots vs. tool scoping / least privilege. Both limit what a server can do, but at different layers. Tool scoping (Cluster A) limits which tools an agent has. Roots limit the resource/filesystem boundary a server may operate within. If a scenario is about confining a filesystem server to certain directories, that's roots. If it's about which tools an agent is granted, that's scoping. (They're complementary least-privilege controls.)
- Notifications vs. request/response. A notification is one-way, no response expected. A tool call is request/response. If a scenario describes the server pushing an update (capabilities changed, progress) without the client having asked in the usual paired way, that's a notification. The list-changed case specifically is about the toolset/resource set changing during a session.

# Tool Boundaries, Access Scoping, and Least Privilege
- It applies least privilege — grant only the minimum capability required — at the MCP layer, across three related boundaries:
- Tool boundaries — what a server exposes. A server should expose only the operations that are actually needed, and each tool should be narrowly scoped to its purpose.
- Access scoping — what the server can reach and on whose authority. The server's own access to underlying systems should be scoped down: it should hold the minimum permissions on the backend needed for its legitimate tools, not broad admin credentials. If a server only needs to read a table, it gets read-only access to that table — not read/write to the whole database.
- Least privilege as the governing rule. Tie both together: every capability the server has that it doesn't strictly need is pure downside — attack surface with no benefit. The most reliable way to prevent a dangerous operation is for the server not to expose it and for the backing credentials not to permit it.
- An MCP server should be a minimal, purpose-scoped capability surface — narrow tools, minimal backend permissions, sanctioned boundaries — so that its worst-case behavior is tightly bounded.


# MCP in Production — Authentication, Versioning, and Reliability
- Authentication and authorization. A remote MCP server is exposed over the network, so it must establish who is calling (authentication) and what they're allowed to do (authorization) — the concern flagged when we covered StreamableHTTP. Your notes tie MCP's production auth to standard web practice: token-based auth, commonly OAuth-style flows, so the server can verify a caller's identity and issue/accept scoped credentials.
- A local STDIO server for one developer needs almost no operational rigor. But the moment you deploy a StreamableHTTP server as a shared, remote service (last cluster's transport lesson), you've built production infrastructure — and it inherits every hard problem production services have. Who's allowed to call it?
- Versioning. Servers evolve — tools get added, parameters change, behavior is refined. But clients depend on the current contract, so uncoordinated changes break consumers. Versioning is how you evolve a server without breaking the clients relying on it. The principles: maintain backward compatibility where possible, communicate breaking changes explicitly, and version the interface so clients can rely on a stable contract. T
- Reliability. A production server must behave well under real-world conditions: handle errors gracefully, manage load, deal with failures of its own dependencies, and not become a single point of failure that takes down every agent connected to it.
- a production MCP server is real infrastructure and must be treated as such — authenticated and authorized access, a versioned/stable contract, and reliable behavior under failure and load.


# The Built-in Toolset — What Ships with Claude
- It's the inventory: the categories of tools Anthropic provides out of the box, so you know what's available before deciding to build. Your notes describe the built-in toolset by capability category.
- Web search / web access — a server tool (Anthropic-executed, per the Domain 1 client-vs-server-tool distinction).
- Code execution — a sandboxed environment where Claude can run code (e.g., Python) to compute, analyze data, and produce results, without you standing up your own sandbox. Anthropic manages the execution environment.
- Computer use — the capability for Claude to interact with a computer interface (screen, clicking, typing) to operate software as a user would. A distinct, higher-capability built-in for automating GUI-driven tasks.
- File / document handling capabilities — built-in support for working with files and documents in supported forms.

- Server tools run on Anthropic's side. For built-ins like web search, Anthropic executes the tool and returns the result (vs. custom/MCP tools your side executes). Less infrastructure for you; less control for you.

- Built-in (server) tools vs. custom (client/MCP) tools — who executes. Built-ins like web search are server tools: Anthropic runs them. Custom and MCP tools are executed on your side (your app / your MCP server).

# Choosing Built-in vs. Custom — Boundaries and Tradeoffs
- It gives you the decision framework — when a built-in is right, when custom is required, and the tradeoffs on each axis.

Reach for a built-in when:
- The need is a common, general capability Anthropic already provides — web search, code execution, computer use, file handling.
- You want zero infrastructure and zero maintenance — Anthropic hosts, secures, updates, and optimizes it. Server-side built-ins mean nothing to deploy on your side.
- You want out-of-the-box integration and reliability — it's tuned to work well with Claude, and its upkeep isn't your problem.
- The tradeoff you accept: less control and less customization. You take the built-in's behavior as given; you can't reshape it to bespoke needs, and execution happens on Anthropic's side (less visibility/control).

Build a custom tool / MCP server when:
- The capability is specific to your systems or domain — your proprietary database, your internal API, your business logic. No built-in can know your private schema, so custom is required, not optional.
- You need control over behavior, execution, data handling, or security — custom/MCP executes on your side, so you own how it runs, what it touches, and how it's scoped (all the Cluster A–C design levers).
- You need specific scoping, boundaries, or least-privilege controls that only your own tool/server can enforce.
- The tradeoff you accept: you own the build, the hosting, the security, the versioning, and the reliability — everything from Cluster C. More control, more responsibility.
# ChatBot
- text based reply
- statless text gnerator 
- no ability to call API, web search, open file or remember
- no decision making

# determinsitic flow 
- steps are fixed at design time
- llm is still called for processing or generating output not decision

# agentic system 
- current context, decides which tool to call
- observes the result, and then decides what to do next
- runtime decision-making

# action-environment gap.
- LLMs is token generator 
- no native ability to read a file, execute code, query a database, or observe anything outside the current inference call
- No persistent memory, no side effects, no feedback loop
- boundary — between what the model describes doing and what actually happens — is the action-environment gap

# Tool Use as the Foundation of Agency
- fundamentally for: closing the action-environment gap
- Tool use is the bridge that turns model text output into real-world action.

# tool definition
- schema — name, natural language description, parameter specification

# tool call
- tool call is the structured output the model emits, containing the tool name and a JSON argument object. then application execute the tool

# four-pillar agentic loop
- There has to be goal for the task
- Perception — the model reads the request plus available tool schemas.
- Selection — it matches the tool description with schemantic mathcing and generates tool argumeters. E.g it picks check_room_availability with Thursday as the argument
- Execution — the runtime calls that, gets back "all rooms booked."
- Iteration — that observation enters context, and the model plans differently: it now selects check_room_availability for Friday instead.
- Second pass returns a free room, it selects book_room, that succeeds, and the goal condition is met, so the loop terminates

# Loop Control
- Problem : loop that doesn't know how to stop 
   - agent can keep calling tools forever
   - failing API a thousand times,
   - ping-ponging between two tools
   - burning tokens
- Defn : Loop control is the discipline of defining, in advance, the conditions under which the agent stops
- Various Conditions to agent to stop
  - Goal: achieved the success condition.
  - Limit hit: the guardrail condition
    - a maximum number of iterations, a token budget, a wall-clock timeout, or a spend cap
    - limit fires whether or not the agent thinks it's making progress
    - backstop that turns "loops forever" into "fails after N tries
  - Error requires escalation: 
     - when the failure is not something more attempts will fix
     - e.g agent lacks the authority, the information, or the permission to proceed safely.
     - loop which need a human is the core judgment of loop control
  
  # Task Decomposition & Planning
- breaking a complex goal into smaller, actionable subtasks the agent can tackle one at a time
- Planning is deciding the order and dependencies among those subtasks before executing
- ended goal into a sequence (or structure) of concrete steps, each small enough to execute and verify
- Problem it solves 
  - each subtask is checkable —
  - failures are contained - if a subtask fails, you know exactly where, and you can retry or escalate
  - the work becomes auditable
     - a plan you can read is a plan you can review,


# Decomposition Patterns — Hierarchical, Sequential, and Parallel
- Sequential : subtasks run in a fixed order, each depending on the output of the one before.
- Parallel: independent subtasks run at the same time, then their results are combined. This is the right choice when subtasks don't depend on each other. The pattern your notes name is fan-out / fan-in: fan-out dispatches the independent pieces of work concurrently, fan-in gathers and synchronizes the results once they're all back.
- Hierarchical: subtasks are decomposed into further subtasks, forming a tree. A high-level goal breaks into major subtasks, and each of those breaks down again until you reach steps small enough to execute directly

# Multi-Agent Orchestration
- A single agent has one context window and one loop. Push a large, multi-domain job through it and two things break.
- the context fills with everything at once — research notes, tool outputs, intermediate reasoning for every subtask
- problem it solves :multi-agent orchestration structures how several agents divide work and coordinate.
- one agent that owns the top-level goal, decomposes it, delegates subtasks to subagents,
- Directing, delegating, and aggregating — that's the orchestrator's three-part job
- Context isolation : ach subagent gets its own clean context window
- Scoped authority — each subagent gets only the tools and permissions its job requires


# Subagent design
- Scope — a subagent should own one well-defined piece of work, not a vague mandate. Narrow scope makes it reliable and testable.
- Authority — it gets only the tools and permissions that scope requires. A research subagent reads; it doesn't get write access to production.
- Isolation — it runs in its own context, unaware of the orchestrator's full state and its sibling agents' internals. Isolation is a feature: it keeps contexts clean and contains failures, but it also means everything a subagent needs must be handed to it explicitly, because it can't see the broader picture.
- message schemas — a defined structure for agent-to-agent communication, so a subagent's output is something the orchestrator can reliably parse and act on, not free-form pros
- handoff protocols — when work passes from one agent to another, the handoff payload must carry everything the next actor needs to continue, because the receiving agent has none of the sender's context

# Topologies
- hub-and-spoke (orchestrator at the center, subagents as spokes, no spoke-to-spoke talk)
- pipeline (agents in a line, each feeding the next)
- peer-to-peer (agents talk directly to each other)


# handoff vs delegation
- Orchestrator → each research subagent: delegation (down-and-back). The orchestrator assigns competitor A, expects findings returned.
- Research phase → synthesis: handoff (forward). Responsibility moves on; the synthesis actor continues the pipeline.
- Subagent → orchestrator, returning results: this is the return side of delegation, and it also needs a well-formed payload (via the message schema) — which is why it feels handoff-like too.

# Topology Selection — Hub-and-Spoke vs. Pipeline vs. Peer-to-Peer
- It determines how control flows, where failures propagate, how much coordination overhead you pay, and how auditable the system is. Pick a topology that's more connected than the task needs
- Hub-and-spoke: 
   - Pros
    - One central orchestrator (the hub) connects to subagents (the spokes)
    - communication flows through the hub — spokes don't talk to each other.
    - orchestrator-led designs. I
    - clear central control, easy to audit (every message passes through one place), and contained failures (a spoke failing is isolated; the hub decides what to do)
  - weakness
     - single point of failure and a potential bottleneck
     - every message and decision routes through it, so it can get overloaded
     - when you need centralized coordination and clear authority, which is most orchestrated tasks
- Pipeline
   - Agents arranged in a line, each one's output feeding the next
   - Pros: simple, clear data flow, naturally suited to staged work where each phase transforms the previous phase's output.
   - Weaknesses: latency accumulates down the line, and each stage is a single point of failure — if stage three breaks, everything downstream stalls, and the work in stages one and two may be stranded.
   - Best when the task is a genuine sequence of transformations with real stage-to-stage dependencies.
- Peer-to-peer
  - Agents communicate directly with each other, no central hub.
  - Strengths: flexible, no single bottleneck, agents can coordinate dynamically
  - Weaknesses — and this is the big one — it's the hardest to control, audit, and debug.

  # Reliability & Human Oversight
  - Issue: An agent takes actions in the world, across many steps, based on runtime observations. Things go wrong constantly — a tool times out, the model reasons its way to a wrong conclusion, the environment returns something unexpected.
  - Classify failures into three categories, and the category
     - Tool errors. The failure is in the tool execution itself, not the model's thinking.The signal is usually explicit: an error code, an exception, a failed status. Relatively easy to detect because the tool tells you it failed.
     - Reasoning errors. The failure is in the model's logic. The tool worked perfectly, but the model chose the wrong tool, passed wrong-but-valid arguments, misread an observation, or drew a bad conclusion. nothing throws an error — the system reports success while doing the wrong thing.etection is hard precisely because there's no error signal; you need validation to catch them.
     - Environment errors. The failure is in the world the agent operates in, outside both the model and the tool's own logic.
- Fix 
   - Transient tool errors → retry
   - Structural tool errors → adjust parameters and retry, or escalate.
   - Reasoning errors → you can't retry your way out. so you need validation to catch them and often a human or a different approach to fix them
   - Environment errors → often need escalation or a fallback, because the agent can't change the environment itself.
   - Error detection — catching failures before they cascade.
      - Your notes make this its own lesson, and the framing is: in an agentic loop, an undetected error doesn't stay put — it feeds into the next step as a (bad) observation, and the model plans on top of it

# Retry vs. Abort, and Fallback Chains / Graceful Degradation
Retry logic — when to retry, when to abort. The core judgment: retry only when the failure is plausibly recoverable and a repeat attempt has a different chance of succeeding. Your notes tie this straight back to the error taxonomy.

- Transient tool errors (timeout, 500, dropped connection) — retry. These genuinely may succeed on a second attempt.
- Structural errors (malformed parameters, wrong input) — don't retry identically; that just reproduces the failure. Either adjust and retry, or abort/escalate.
- Reasoning errors — retrying is usually pointless. The model will make the same mistake again. This needs validation or a different approach, not a repeat.
- Permission / authorization errors — never retry blindly; the credentials won't change on attempt two. Escalate.

- Bounded retries — a hard cap (e.g., three attempts), never infinite
- Backoff — space attempts out (often exponentially) rather than firing them back-to-back
- And abort as a first-class decision — aborting isn't failure, it's the correct move when retries are exhausted or the error is unrecoverable
- Fallback chains. When the primary approach fails and retrying won't fix it, a fallback chain provides an ordered sequence of alternatives:
- Graceful degradation. The principle that when the full result is unavailable, the system should deliver a reduced-but-useful result rather than failing outright
- retry (same approach again, for transient failures) 
- fallback (different approach, when the primary path is dead)
- graceful degradation (reduced result, when no full path succeeds).

# Reliability & Human Oversight — Part 3: Why Prompts Aren't Enough, and Programmatic Enforcement
Why the prompt alone fails, stated the way the exam wants it:

A prompt is probabilistic guidance. Compliance is likely, never certain. Across enough runs, the rare violation will happen.
Prompts are vulnerable to adversarial input. A user (or injected content from a tool result) can craft input that steers the model past the instruction. Prompt injection is a real attack surface, and you cannot prompt your way to being immune to prompt attacks.
Prompts have no hard guarantee. There's no mechanism forcing compliance — only the model's tendency to comply.

# Reliability & Human Oversight — Part 4: Escalation Triggers & Human-in-the-Loop Review
- Put a human checkpoint on everything and you've destroyed the point of an agent — it's now a slow
- Escalation trigger design — valid vs. invalid patterns. An escalation trigger is a condition that causes the agent to stop and hand off to a human (or higher authority) instead of proceeding. The whole art is defining good triggers. Your notes frame this as valid vs. invalid patterns:
- Valid escalation triggers are tied to concrete, meaningful conditions:

   - High-stakes / irreversible actions — the action crosses a consequence threshold (large money movement, data deletion, external communication). Escalate before it executes.
   - Low confidence — the model or a validation check signals it isn't sure. Uncertainty on a consequential path is a reason to get a human, not to guess.
   - Unrecoverable errors — retries and fallbacks are exhausted, and the failure needs authority or information the agent doesn't have (the permission-error case from loop control).
   - Explicit policy conditions — a business rule says this case type always needs review (e.g., refunds over a threshold, contracts above a value).
Invalid escalation patterns are the anti-patterns that make human oversight worthless:
  - Escalating everything — if the agent kicks every action to a human, you've built a bottleneck, not an agent. The human tunes out (alert fatigue) and the value of autonomy is gone.
  - Escalating nothing — no triggers on genuinely high-stakes paths; the agent runs unsupervised where it shouldn't.
  - Vague or unactionable triggers — escalating without giving the human the context to act, so the "escalation" is just a shrug passed upward.
  - Triggering on the wrong signal — e.g., escalating on transient tool errors (which should just retry) instead of on consequence or confidence.
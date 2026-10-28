# CCAF — Claude Certified Architect (Foundations)
## Study Index: Domains & Topics

**Exam facts:** 60 questions · 2 hours · fully proctored (no Claude, docs, or browser) · passing score **720/1000** · ~6-month recertification cycle. Domain weights shift over time. Anthropic recommends ~6 months hands-on; this course assumes 2+ months.

---

## Domain 1 — Agentic Architecture  ✅ *(complete — taught + diagrammed)*

1. **Agents vs Workflows vs Chatbots** — who controls the control flow, and *when*
2. **Tool Use Lifecycle** — the roundtrip; model *requests*, app *executes*
3. **The Agentic Loop** — Perceive → Reason → Act → Observe (feedback loop, not a pipeline)
4. **Loop Control** — termination condition, turn budget (in code), escalation; 4 stop reasons
5. **The Orchestrator Role** — decompose → assign → aggregate (validate!)
6. **Subagent Design** — context isolation, least privilege, single responsibility
7. **Topologies** — hub-and-spoke vs pipeline vs peer-to-peer
8. **Multi-Agent Cost & Fit** — 1× / 4× / 15× token multipliers; default to one agent
9. **Handoffs & Message Schemas** — 4 schema components; silent failure is the key trap

*Deliverables: 9 per-topic `.puml` activity diagrams + combined files.*

---

## Domain 2 — Tool Design & MCP  🔵 *(current)*

**Part A — Tool Design**
1. **How Claude Selects Tools** — the description is the primary routing signal *(current topic)*
2. **Writing Tool Descriptions That Route Correctly**
3. **Designing Tool Input Schemas for Precision & Reliability**
4. **Required vs Optional Parameters, Enums & Constraints**
5. **Strict Tool Use and Valid Inputs**
6. **Returning Useful Errors from Tools**
7. **Tool Retry Logic and Partial Success Patterns**
8. **Tool Scoping per Agent — Why Less Is More**
9. **Selection Heuristics and Tool-Selection Failures**
10. **Controlling Tool Selection**
11. **Tool Search for Large Catalogs**

**Part B — MCP (Model Context Protocol)**
12. **What MCP Is and Why It Exists**
13. **The Three MCP Primitives — Tools, Resources, Prompts**
14. **Building Your First MCP Server in Python**
15. **Defining and Exposing Tools via MCP**
16. **Defining and Exposing Resources**
17. **MCP Prompt Templates — Design and Use Cases**
18. **Implementing MCP Clients and Connecting to Servers**
19. **STDIO Transport — Local Servers**
20. **StreamableHTTP Transport — Remote/Production**
21. **Sampling — Letting Servers Make LLM Calls Through the Client**
22. **Notifications, Roots, and Progress Reporting**
23. **Tool Boundaries, Access Scoping, and Least Privilege**
24. **MCP in Production — Authentication, Versioning, Reliability**
25. **The Built-in Toolset — What Ships with Claude**
26. **Choosing Built-in vs Custom — Boundaries and Tradeoffs**

---

## Domain 3 — Claude Code  ⚪ *(pending)*

1. What Claude Code Is and How It Works
2. The Tool System — Reads, Bash, Edits, Web Search
3. Starting a Project — Setup & Onboarding
4. Adding and Managing Context Effectively
5. Permission Modes
6. The Three-Level CLAUDE.md Hierarchy
7. Precedence Rules and Conflict Resolution
8. Subdirectory CLAUDE.md Files and Scoped Instructions
9. Excluding Files and Content
10. Instruction Design — What Claude Code Actually Reads
11. Common CLAUDE.md Patterns for Real Teams
12. Creating and Structuring Custom Slash Commands
13. Command Design Patterns and Team Distribution
14. What Skills Are and When to Build Them
15. Authoring Agent Skills
16. Creating and Configuring Custom Subagents
17. Designing Subagents for Delegation
18. Hook Lifecycle Events
19. Plan Mode — When and Why to Plan First
20. Iterating on a Plan — Approve, Edit, Re-Plan
21. Four Common Hook Events
22. Implementing Hooks — Shell Scripts, Exit Codes, Gotchas
23. Real-World Hook Applications — Logging, Linting, Safety Nets
24. The Claude Agent SDK — Session Control
25. Claude Code in CI/CD — Non-Interactive Mode and the -p Flag

---

## Domain 4 — Prompt Engineering  ⚪ *(pending)*

1. Clarity, Specificity, and the Cost of Ambiguity
2. Instruction Decomposition and Constraint Design
3. The System Prompt — Purpose, Structure, Best Practices
4. Role Assignment, Persona Design, Context Setting
5. Using XML Tags to Structure Complex Prompts
6. Document Structure Patterns for Multi-Part Tasks
7. Why Examples Work — Pattern Matching & Format Anchoring
8. Designing High-Quality Few-Shot Examples
9. Chain-of-Thought & Extended Thinking
10. Prefilling and Steering Output
11. Temperature, Top-p, Top-k
12. Multi-Turn / State
13. JSON & Schema-Constrained Output
14. Extraction Schema Design
15. Validation Loops & Hallucination Prevention
16. Validation Gates & Retry
17. Multi-Pass Review
18. Prompt Eval Workflow
19. Test Dataset Generation & Model-Based Grading

*(Exact lesson list to be confirmed from transcript when we reach this domain.)*

---

## Domain 5 — Context Management  ⚪ *(pending)*

1. In-Context State vs External Memory
2. Session Continuity Across Turns and Failures
3. Context Window Mechanics & Economics
4. Why Summaries Lose Facts / Persistent Fact Block
5. The Memory Tool
6. Tool-Output Bloat & Trim Strategies (filter / summarize / reference)
7. Escalation Trigger Design
8. Handoff Payload & Propagation Contract
9. Scratchpad Pattern
10. Delegating Discovery to a Subagent
11. Long-Conversation Coherence / Compaction / Session Memory
12. Server-Side Compaction
13. Per-Field Confidence & Calibration with a Validation Set
14. Stratified Sampling / Routing Low-Confidence to Review
15. Claim-to-Source Mappings & Conflict Annotation

*(Exact lesson list to be confirmed from transcript when we reach this domain.)*

---

## Legend
- ✅ complete   🔵 in progress   ⚪ pending
- **Pace:** one domain per session, interactive recall format.

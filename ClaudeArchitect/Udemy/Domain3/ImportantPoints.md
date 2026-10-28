# What Claude Code Is and How It Works
- Claude Code is Anthropic's agentic coding tool — Claude operating as an agent in your terminal/development environment, with direct access to your filesystem and shell.
- Claude Code is a terminal-based agent that can autonomously read files, run commands, edit code, and iterate, rather than just generating text you paste elsewhere. It is the Domain 1 agentic loop (perceive → select → execute → observe → iterate)
- Claude perceives your codebase, selects tools (read, edit, bash), executes them against your real environment, observes results (test output, errors), and iterates toward the goal.
- Claude Code vs. Claude in a chat window. Both are Claude, but Claude Code is agentic and environment-integrated — it reads/runs/edits directly. Chat is conversational — it generates text you act on. If a scenario involves Claude autonomously operating on a real codebase (running tests, making multi-file edits), that's Claude Code, and its architecture is the Domain 1 agentic loop.
- Claude Code vs. the Agent SDK. Foreshadow: Claude Code is the interactive terminal product; the Claude Agent SDK (Cluster D) is the programmatic framework for building your own agents with the same underlying machinery. Same foundation, different surface — don't conflate the CLI tool with the SDK. We'll separate them fully later.

# The Tool System — Reads, Bash, Edits, and Web Search
- It enumerates Claude Code's built-in tools by function, and — importantly — sorts them by risk, which is what the permission system keys off.
- Read tools — Claude Code reads files from your codebase to understand it: opening files, viewing contents, listing directories, searching across the code
- Bash / command execution — Claude Code runs shell commands: running tests, building, installing dependencies, executing scripts, git operations, anything the terminal can do. This is enormously powerful
- Edit / write tools — Claude Code modifies files: making targeted edits, writing new files, applying multi-file changes.
- Web search / web access — Claude Code can retrieve current information from the web (e.g., looking up current library docs, error messages, APIs).
- bash is the highest-risk tool (arbitrary command execution) and reads are lowest (non-mutating). Permission modes and hooks are designed around this gradient. If a scenario asks where to concentrate oversight, the answer tracks the risk: bash and edits, not reads.

# Starting a Project — Setup and Onboarding Claude to a Codebase
- It's the process of getting Claude Code productively oriented in a codebase — both the initial exploration and, crucially, the persistent project knowledge that survives across sessions.
- Initial exploration / codebase understanding. When Claude Code starts on a project, it can explore to build a working model — reading key files, mapping the directory structure, identifying the stack and conventions. This is Claude perceiving the codebase (the read tools from the last topic). It's how it answers "what am I working with." But exploration alone is ephemeral — it lives in the current context and is lost when the session ends, so relying on it every time is wasteful and inconsistent.
- Persistent project memory via CLAUDE.md. The durable solution — and the hinge into all of Cluster B — is the CLAUDE.md file: a project file that Claude Code automatically reads and loads into context at the start, containing the project-specific knowledge you want Claude to always have. Instead of re-explaining or re-discovering the project each session, you write it down once in CLAUDE.md, and Claude Code picks it up automatically. This is the Domain 5 "persistent fact block" idea (which you'll see later) made concrete for coding: externalize the stable, important context so it's reliably present rather than rediscovered.

What belongs in that onboarding memory — the project knowledge that makes Claude act like it knows the codebase:

- Architecture and structure — how the project is organized, where key things live.
- Conventions and standards — coding style, naming, patterns the team follows.
- Commands — how to build, test, run, lint (so Claude uses your commands, not guesses).
- Constraints and gotchas — things to avoid, project-specific rules, "don't touch X."
- Tech stack and key dependencies — what the project is built on.

# Adding and Managing Context Effectively
- It's the set of practices for curating what's in Claude Code's context so the signal-to-noise stays high. Your notes frame this around a few ideas:
- Context is a scarce, shared resource — spend it deliberately. Everything in context competes for the same finite budget: your CLAUDE.md, the files Claude has read, tool outputs, the conversation so far. More context isn't better; relevant context is better.
- Add context intentionally. You steer what Claude focuses on by directing it to the relevant files/areas rather than letting it wander the whole codebase. Pointing Claude at the specific files, directories, or modules that matter for the task gets the right material in fast and keeps it focused.
- Keep noise out. Just as important as what you add is what you exclude. Irrelevant files, large generated artifacts, dependencies, build outputs — pulling these in wastes budget and dilutes attention. 
- context is finite and quality-sensitive, so curate it — add the relevant deliberately, exclude the irrelevant, and manage accumulation over long sessions.


# Claude Code Permission Modes
- Permission modes govern how much Claude Code can do without asking you first — the autonomy-versus-oversight dial. Your notes describe a spectrum of modes; the exam-durable thing is understanding the spectrum and the tradeoff, since exact mode names can shift:
- A default / prompting mode — Claude Code asks for approval before taking consequential actions (running commands, making edits), so you review and approve them as they come. This is the safe default: you stay in the loop on mutating/risky operations. Higher oversight, lower speed.
- An auto-accept / autonomous mode — Claude Code executes actions without stopping for per-action approval, running through the work autonomously. This is fast and powerful for when you trust the task and want Claude to just get it done — but it removes the per-action human checkpoint, so the oversight burden shifts to before (choosing to enable it) and after (reviewing results), plus whatever guardrails (hooks, scoping) you've set. Higher speed, lower oversight.
- A plan / read-only-leaning mode — Plan Mode (its own Cluster D topic) is the low-risk end: Claude analyzes and proposes a plan without making changes, so you review the intended approach before any mutation happens. Effectively "look, don't touch yet." (Full treatment later; note here that it sits at the safe end of the permission spectrum.)

The judgment the exam wants — match the permission mode to the stakes and your trust in the task:

- Unfamiliar codebase, high-stakes changes, production-adjacent work, or anything you can't easily undo → more oversight (prompting/default, or Plan Mode first).
- Well-understood, low-risk, repetitive, easily-reversible work you want done fast → more autonomy (auto-accept), with appropriate guardrails.

- Modes vs. hooks/scoping (defense in depth). Permission modes are one control; hooks (Cluster D) can programmatically enforce rules on tool use, and scoping limits capability. On genuinely high-stakes work you combine them — an auto-accept session still governed by hooks that block dangerous commands, for instance.

# The Three-Level CLAUDE.md Hierarchy
- Claude Code reads CLAUDE.md files at multiple levels, and it combines them — they're not mutually exclusive; all applicable levels load together into context
- Project level — a CLAUDE.md in the project/repository. This holds project-specific memory that applies to anyone working in this codebase: architecture, conventions, build/test commands, constraints. Because it typically lives in the repo, it's shared with the team via version control — commit it, and every teammate's Claude Code picks up the same project instructions. This is the "team standards" level.
- User / global level — a CLAUDE.md at the user level (your home/config directory), holding your personal preferences that apply across all projects you work on. This is where your individual working style lives ("always show me the plan before editing," "prefer functional style") — it follows you, not the project, and isn't shared with the team. This is the "personal, everywhere" level.
- Local / project-local (personal-to-this-project) level — instructions that are yours but scoped to this specific project, and not shared with the team (kept out of version control / git-ignored). This covers personal overrides or notes for a particular repo that you don't want to impose on teammates — your own scratch preferences for this project only.

# Precedence Rules and Conflict Resolution
- It defines the ordering among the CLAUDE.md levels so that conflicts resolve deterministically. - The governing principle your notes establish is that more specific 
- The precedence ordering, from highest (wins) to lowest (yields), follows specificity/locality:
    - Subdirectory-level CLAUDE.md (most specific — a particular part of the project; full topic next) — highest.
    - Project-level / project-local (specific to this project).
    - User / global (least specific — applies to all projects) — lowest.
- precedence only matters on genuine conflict. When instructions don't conflict, they simply combine — this is the default relationship from the hierarchy topic.

# Subdirectory CLAUDE.md Files and Scoped Instructions
- Claude Code reads CLAUDE.md files placed in subdirectories, not just at the project root, and those instructions are scoped to that subdirectory — they apply when Claude is working on files within that part of the tree. This is the most specific level of the hierarchy (highest precedence, from the last topic), and it's how you attach the right rules to the right region of a codebase without polluting the global project config.
- Scope by location. A CLAUDE.md in src/frontend/ governs work in the frontend; one in src/backend/ governs the backend. The instruction's reach is defined by where the file sits in the directory tree. Claude applies the subdirectory's instructions when operating on files under that directory, in addition to the broader project and global instructions that still apply.
- Layering, not replacement. Subdirectory files add to and refine the higher levels — they don't wipe them out. Within src/frontend/, Claude follows the project-wide rules plus the frontend-specific ones, with the subdirectory rules winning only where they genuinely conflict (precedence by specificity). Everywhere the subdirectory file is silent, the broader project/global instructions still fill in. So it's the general baseline, locally specialized.
- Keeps instructions relevant and uncluttered. Instead of one giant root file full of location-conditionals, each area carries its own concise, relevant rules. This mirrors the context-management principle: the right instructions in the right place, without noise. It also keeps each file maintainable and lets different teams own the config for their part of a monorepo.
- When a scenario has a non-uniform codebase (monorepo, distinct modules with different rules), the right architecture is scoped subdirectory CLAUDE.md files, not cramming everything into the root with conditionals. If instructions differ by area, put t

# Excluding Files and Content
- Claude Code provides ways to exclude files and content so they're kept out of what Claude reads and loads. The purposes fall into two buckets your notes emphasize:
- Context efficiency / noise reduction. Excluding irrelevant, bulky, or generated content — dependencies (node_modules, vendor/), build outputs (dist/, build/), large generated files, logs — keeps them from consuming the context window and burying the relevant code. This is the exclusion side of the "curate context, keep noise out" principle from the context-management topic: you protect signal-to-noise by preventing junk from entering context in the first place.
Security / sensitivity. Excluding sensitive files — .env, credentials, secret keys, private configuration, sensitive data — so Claude doesn't read them into context or risk surfacing them. This is a least-privilege boundary at the content level: if Claude never needs to see your secrets to do its job, it shouldn't have access to them, and exclusion enforces that structurally. (The strongest way to keep a secret out of the model is to make it unreadable, not to instruct the model to ignore it.)
- Exclusion keeps the wrong content out of Claude Code — noise, for context efficiency, and sensitive material, for security.
- The mechanism operates on paths/patterns — you specify what to exclude (commonly following gitignore-style patterns, and Claude Code respects standard ignore conventions), and those files are kept out of Claude's reads and context. The effect is a boundary: excluded content is not loaded, not read, effectively invisible to the agent.
- Security exclusion: .env, .env.local, config/secrets.yml, and credentials.json are excluded. Now when Claude explores the project to make a change, it cannot read your database passwords, API keys, or signing secrets — they're structurally outside its

1. .gitignore — inherited automatically. Claude Code respects your existing .gitignore. Anything already ignored by git (like node_modules/, dist/, .env) is generally kept out of Claude's view without you doing anything extra. This is why standard noise and already-git-ignored secrets are typically excluded by default — you already listed them for git.

2. .claudeignore — Claude-Code-specific exclusions. For content you want excluded from Claude but not necessarily from git (or to be explicit and independent of your gitignore), Claude Code supports a dedicated ignore file using the same gitignore-style patterns. This is the place for "I want git to track this, but I don't want Claude reading it," or to add exclusions beyond what .gitignore covers.

3. Settings-level permission/deny configuration. Claude Code's settings (its configuration files) can also define permission rules that deny access to specific paths — a settings-based way to block reads of particular files/directories. This overlaps with the permission-modes/controls layer: it's not just "don't load into context," it's "deny the tool from touching this path." This is the stronger, access-control flavor of exclusion.


# Instruction Design — What Claude Code Actually Reads and Follows
- It's the set of principles for writing CLAUDE.md instructions that translate into reliable behavior. Your notes emphasize:
- Be specific and actionable, not vague. The same principle as tool descriptions (Domain 2) and prompts (Domain 4): "write clean code" is unactionable — Claude can't reliably act on it. "Use AppError for all thrown errors; put routes in src/api/routes/, one per resource" is concrete and followable. Instructions should tell Claude exactly what to do, in terms it can act on, not express aspiration
- Keep it concise / avoid bloat. A CLAUDE.md competes for the same context budget as everything else (context economics), and an overloaded instruction file dilutes the important rules among noise — Claude is less likely to reliably follow any single rule when it's buried in a hundred. Prioritize the high-value, load-bearing instructions; cut the rest. A focused, concise file produces better adherence than an exhaustive one. Quality and priority over volume.
- Structure for clarity. Organize instructions so they're easy to parse — grouped by concern, clearly delineated (the document-structure principles from Domain 4 apply). A well-structured file is more reliably followed than a stream-of-consciousness dump.
- Avoid contradictions. Conflicting instructions within or across files create unpredictable behavior. Keep instructions consistent (and rely on the precedence rules only for genuine cross-level conflicts, not as a substitute for writing coherent instructions in the first place).
- Put the most important things where they carry weight. Load-bearing constraints (safety rules, hard "never do X" boundaries) should be prominent and unambiguous, not buried among stylistic preferences. Emphasis and placement matter for what Claude actually prioritizes.
- : an instruction only helps if Claude reliably follows it, and reliable adherence comes from specific, actionable, concise, well-structured, non-contradictory instructions with the important rules given prominence.

# Common CLAUDE.md Patterns for Real Teams
- It's the set of proven CLAUDE.md content patterns — the high-value things worth capturing so Claude Code behaves like an informed team member. Your notes describe the recurring categories:
- Build, test, and run commands. How to build, run tests, lint, and start the project — using the team's actual commands (make test, not a guessed npm test). This is one of the highest-value entries: it stops Claude from guessing at how to run the project and ensures it verifies its work the way the team does.
- Architecture and project structure. Where things live, how the codebase is organized, the major components and how they relate. Gives Claude the map so it puts new code in the right place and understands what it's touching.
- Coding conventions and style. Naming, patterns, formatting expectations, preferred idioms, the error-handling approach — so Claude's output matches the existing codebase rather than introducing a foreign style.
- Tech stack and key dependencies. Languages, frameworks, major libraries, and how they're used (e.g., "DB access via Prisma," "state via Redux") — so Claude works with the stack instead of against it.
- Constraints, boundaries, and "do not" rules. Hard prohibitions and gotchas: "never modify legacy/," "don't run migrations against prod," "don't add new dependencies without approval." These are the load-bearing safety/scope rules that (per instruction design) deserve prominence.
- Workflow and process expectations. Team practices Claude should follow — branching model, commit conventions, PR expectations, "run tests before finishing," "update the changelog."


# Creating and Structuring Custom Slash Commands
- A custom slash command is a reusable, named prompt/workflow you invoke in Claude Code with /commandname. Instead of retyping a complex instruction, you define it once and trigger it by name. Your notes establish the mechanics and structure:
- Defined as files (typically Markdown). A slash command is created by placing a Markdown file in a designated commands directory; the filename becomes the command name (review.md → /review),
- They take arguments. Commands can accept input you pass at invocation, injected into the prompt via a placeholder (e.g., an $ARGUMENTS-style token). This makes a command parameterized and reusable across cases — /review src/auth.ts feeds the file path into the command's prompt. Without arguments, a command is a fixed prompt; with them, it's a template.
- Project vs. personal scope (mirrors the CLAUDE.md hierarchy). Commands can live at the project level — in the repo, committed, shared with the team so everyone gets the same commands — or at the personal/user level — your own commands across projects, not shared. Same sharing/scope logic as CLAUDE.md: team workflows go in the project (committed); personal shortcuts go in your user config.
- Structure for a good command. The command's content is a prompt, so instruction-design discipline applies: make it specific and actionable, structure it clearly, and use arguments for the variable parts. A well-structured command encodes a known-good way of doing a recurring task so it runs consistently every time.
- Slash command vs. CLAUDE.md. Both are reusable prompt content, but they differ in invocation. CLAUDE.md is always-on — automatically loaded into every session as persistent context. A slash command is on-demand — it runs only when you invoke it


# Command Design Patterns and Team Distribution
- Command design patterns.
    - One command, one clear purpose
    - Parameterize the variable parts with arguments.
    - Write the prompt with instruction-design discipline.
    - Encode the team's actual best practice. 
- Team distribution
   - Distribute via the project level / version control.
   - Commands placed at the project level live in the repo and are committed, so they're shared with everyone through git
- design commands to be narrow, parameterized, well-written, and encode real best practice; distribute them at the project level (committed to the repo) so the whole team runs the same standardized workflows.


# What Skills Are and When to Build Them
- A Skill (Agent Skill) is a packaged capability — a set of instructions (and optionally supporting files/scripts/resources) that teaches Claude how to do a particular kind of task, which Claude can invoke automatically when relevant, rather than only when explicitly triggered. Your notes frame skills around a few defining properties:
- Model-invoked / automatically triggered by relevance. This is the headline distinction from slash commands. A slash command is user-invoked — it runs only when you type /name. A skill is model-invoked — Claude decides to use it when the current task matches what the skill is for.
- A packaged bundle, not just a prompt. A skill can contain more than instructions — it can bundle supporting files, scripts, reference documents, templates that the capability needs. This makes skills suited to richer, self-contained capabilities that carry their own resources, where a single-prompt slash command wouldn't suffice.
- Discovered/selected by its description. Because Claude decides when to use a skill, the skill must describe what it's for clearly enough that Claude can recognize when it applies


# Authoring Agent Skills
- It's the authoring principles for effective skills. Your notes center on a few things:
- Structure — the skill file and its metadata. (Markdown-based, SKILL.md-style) containing the instructions, with metadata — critically a name and a description — plus the body of instruction
- The description is the trigger — write it for discovery.
- Claude selects a skill by matching the task against the skill's description

# Creating and Configuring Custom Subagents
-A subagent in Claude Code is a separate agent instance, with its own context window and its own configuration, that the main agent delegates work to. The main Claude Code agent (the orchestrator) hands a scoped task to a subagent, the subagent works on it in isolation — its own clean context — and returns a result to the main agent, without its intermediate work polluting the main conversation. This is exactly the orchestrator/subagent pattern from Domain 1, instantiated: context isolation + scoped delegation, in Claude Code.
- SubAgent Defining propertie
    - Separate, isolated context.
    - Its own configuration.
    - Defined as files, with metadata including a description.
    - Invoked by delegation.
- subagent is a separately-configured, context-isolated agent the main agent delegates scoped work to
- Subagent vs. skill. The big one. A skill is packaged know-how the main agent applies within its own context — instructions/resources that teach Claude how to do something. A subagent is a separate agent with its own context that work is delegated to. Key differentiators: a subagent gives you context isolation (its own window) and its own toolset/config; a skill does not — it runs in the main context. If a scenario needs isolation (keep intermediate work out of the main context) or a separate scoped agent, that's a subagent. If it needs specialized instructions/resources applied in-context, that's a skill. Isolation + own tools = subagent; in-context know-how = skill.


# Designing Subagents for Delegation
- . It's the principles for designing subagents that delegate cleanly, and they map directly onto the Domain 1 triad plus the return contract:
- Scope — one clear, well-defined job. A
- Authority — only the tools its job needs (least privilege).
- Isolation — leverage it, but respect what it implies.
- The delegation + return contract — self-contained in both directions. This is the handoff discipline from Domain 1 applied here. Returning out: the subagent must return a clear, self-contained, usable result — because the main agent
- Description — the delegation routing signal. subagent's purpose and when to use it, so delegation happens at the right times and not the wrong ones

# Hook Lifecycle Events
- The problem it solves. Hooks are user-defined scripts that Claude Code runs automatically at specific points ("lifecycle events") in its operation. They let you inject deterministic, programmatic behavior into the agent loop —
- Before a tool runs (pre-tool-use). block it, or modify the flow
- After a tool runs (post-tool-use). A
- Session / prompt lifecycle events.logging, notifications, or final checks.
- Deterministic and automatic. A hook is not a request to the model — it's code that runs at its event, every time, guaranteed
- hooks are deterministic, automatically-executed scripts at defined lifecycle events (notably before/after tool use) that let you enforce rules and react to the agent's actions with guaranteed reliability

# Plan Mode — When and Why to Plan First; Iterating on a Plan
- Plan Mode is a Claude Code mode where Claude analyzes the task and produces a plan without making any changes — it reads and reasons and proposes how it would do the work, but holds off on edits and mutating actions until you approve. It separates the thinking/proposing phase from the executing phase, putting a human review checkpoint between them.


When to use it: for complex, high-stakes, ambiguous, or risky tasks
- Iterating on a plan — Approve, Edit, Re-Plan
- Approve — the plan looks right; Claude proceeds to execute it.
- Edit — you adjust the plan (correct a step, change an approach, add a constraint) before execution.
- Re-plan — you send Claude back to revise the plan based on your feedback, producing an updated plan you review again.


# Implementing Hooks — Shell Scripts, Exit Codes, and Gotchas
- It's the concrete mechanics of implementing hooks. Your notes center on a few things:
- A hook is implemented as a shell command/script that Claude Code runs at the configured lifecycle event.
- When Claude Code fires a hook, it passes the hook information about what's happening — details of the tool call, the event, relevant data — typically via stdin (as JSON) and/or environment. This is how the hook knows what Claude is about to do (e.g., which bash command, which file is being edited), so it can decide whether to allow it. A pre-tool-use hook inspects this input to make its gating deci
- Exit codes control the flow — the crux. This is the most important mechanic. A hook signals its decision back to Claude Code through its exit code:
- Exit code 0 — success/allow. The hook approves; Claude Code proceeds.
- A non-zero exit code (specifically exit code 2) — block

# Real-World Hook Applications — Logging, Linting, Safety Nets
- It's the set of proven, real-world hook use cases. Your notes group them into a few recurring categories, each mapping to a lifecycle event:
- Safety nets (guardrails) — pre-tool-use, blocking. The highest-value pattern: hooks that prevent dangerous or forbidden actions b
- Linting / formatting / testing — post-tool-use, reactive. Hooks that automatically run quality checks after Claude changes code
- Logging / auditing — on tool-use (and other) events.
- Notifications / workflow integration. Hooks that react to events by notifying or integrating with other systems

# The Claude Agent SDK — Session Control
- The Claude Agent SDK is the programmatic framework for building custom agentic applications on the same foundation that powers Claude Code. Instead of driving Claude interactively through the terminal, you drive it in code — constructing agents, controlling their execution, and integrating them into your own systems. Your notes frame the SDK around session control and the programmatic capabilities it exposes
- It's the framework beneath / alongside Claude Code. Claude Code is the interactive product; the Agent SDK is the toolkit for building your own agents with the same agentic loop, tool use, and capabilities.
- Session control. The SDK gives you programmatic control over agent sessions — creating and managing the lifecycle of an agent's run: starting a session, managing its state and context across turns, controlling its execution, and continuing/resuming as needed. "Session control" means you, in code, govern how the agent runs rather than a human steering it live. This connects to the Domain 5 session-continuity material (managing state across turns/sessions) — the SDK is where you programmatically handle that.
- Programmatic configuration of the agent. T
- Integration into applications and automation
- the Claude Agent SDK is the programmatic framework for building custom agents on the same foundation as Claude Code, with session control giving you code-level command over an agent's lifecycle, state, and execution

# Claude Code in CI/CD — Non-Interactive Mode and the -p Flag
- Claude Code supports a non-interactive / headless mode for automation, invoked with the -p flag (the "print" / prompt flag). Your notes frame it around:
- -p runs Claude Code non-interactively with a given prompt. You pass the prompt directly (claude -p "your instruction"), and Claude Code executes that task autonomously and returns the result — without entering the interactive conversational session. I
- Designed for automation / CI/CD. This is the mode for running Claude Code inside pipelines, scripts, and automated workflows — anywhere there's no human to interact. It turns Claude Code into a command you can call from a CI step. The output can be captured and used by the pipeline (piped, parsed, acted on).
- Autonomous execution — which raises the stakes on guardrails. In non-interactive mode there's no human approving actions in real time, so the per-action human checkpoint is gone. That makes the other controls essential: permissions/scoping (what tools it's allowed), hooks (deterministic enforcement of safety rules), and careful prompt scoping become the guardrails, since you can't rely on a human clicking approve
- Relationship to the SDK. Non-interactive -p is the lightweight, command-line way to run Claude Code in automation; the Agent SDK (last topic) is the fuller programmatic framework for building richer custom agents. Both serve automated/non-interactive use — -p is the CLI-invocation route for scripting Claude Code; the SDK is for building agents into applications in code. (For simple "run this task in a pipeline step," -p; for building a full custom agentic application, the SDK.)
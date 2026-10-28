# Task Statement 2.4: Revision Sheet

## 1. MCP scoping hierarchy

**Purpose:** decide *who* gets an MCP server.

| | **Project level** | **User level** |
|---|---|---|
| **File** | `.mcp.json` | `~/.claude.json` |
| **Location** | Project repository root | User's home directory |
| **Version-controlled?** | Yes | No |
| **Shared?** | Yes, with everyone who clones the repository | No, personal only |
| **Use for** | Servers the whole team needs (Jira, GitHub, internal APIs) | Experimental servers, personal integrations |

**Rules**
- Whole team needs it → `.mcp.json`
- Only I need it, or I'm testing it → `~/.claude.json`
- Tools from **all** configured servers (both levels) are discovered at connection time and are available **simultaneously**. There is **no manual activation step**.

**Example:** eight developers need Jira, so it goes in `.mcp.json`. Priya is trialling a new server, so it goes in her `~/.claude.json`.

⚠️ **Anti-patterns:**
- Team-wide servers in `~/.claude.json`, which forces everyone to configure them by hand.
- Experimental servers in `.mcp.json`, which pushes untested tools onto the whole team.

---

## 2. Environment variable expansion

**Purpose:** keep credentials out of version control.

**Rule:** `.mcp.json` uses `${VARIABLE_NAME}` syntax. The file holds the variable's **name**, never its **value**.

```json
{
  "mcpServers": {
    "jira": {
      "env": {
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    }
  }
}
```

Each developer sets the value locally, in a **shell profile**, a **`.env` file**, or a **secrets manager**.

**Four benefits:**
1. **Safe to commit:** no secrets in the file.
2. **Personal credentials:** each developer uses their own token.
3. **Token rotation without config changes:** update the environment, with no commit needed.
4. **No secrets in repository history:** nothing to leak from old commits.

**Example:** tokens are rotated after six months. Developers update `JIRA_API_TOKEN` locally, and `.mcp.json` is untouched.

⚠️ **Anti-pattern:** pasting tokens directly into `.mcp.json`. "It's a private repo" is **not** a defence, because the token stays in history, is shared, and is visible to everyone with access.

---

## 3. MCP resources

**Purpose:** give agents **visibility** into what content exists, without exploratory tool calls.

**Key terms**
- **Resource:** a content catalogue the agent can see straight away (read-only visibility).
- **Tool:** an action the agent takes on that data.

> 🧠 **Resource = the table of contents. Tool = turning to the page.**

**Typical resources:**
- Issue summaries
- Documentation hierarchies
- Database schemas

**Example:**
- ❌ Without a resource: `list_tables` → `describe_table` × 40 = **41 tool calls** before any real work.
- ✅ With a schema resource: the agent reads the schema once, then runs only the query it needs.

**Working pattern:** the resource lists open Jira issues (ID, title, status) → the agent picks the relevant IDs → the `get_issue` tool fetches the full details.

⚠️ **Anti-pattern:** not exposing content catalogues as resources, which leaves agents wasting calls on discovery or guessing.

---

## 4. Build-vs-use decision

**Purpose:** avoid unnecessary engineering effort.

**Golden rule:**
> **Evaluate community MCP servers first.**

**Use community servers** for standard integrations such as **Jira, GitHub, Slack, Notion**. They are maintained, tested, and cover standard use cases.

**Build custom ONLY when:**
1. A **proprietary internal system** has no community server, or
2. **Team-specific workflows** can't be handled by community servers.

**Example:**
- GitHub integration → use the community server.
- Your in-house deployment tracker → build a custom server.

⚠️ **Anti-pattern:** building custom servers for standard integrations.
🚩 **Exam distractor:** "build it ourselves for full control". Wanting control alone is **not** a valid reason.

---

## 5. Enhancing MCP tool descriptions

**Purpose:** help the agent choose the right tool.

**Why it matters:** agents choose tools by reading their descriptions. If an MCP tool's description is **sparse**, the agent will prefer **built-in tools (like Grep)**, whose descriptions are richer.

**A good description includes four things:**
1. **What it does**
2. **What it returns**
3. **When to use it**
4. **How it compares to built-in alternatives**

**Example:**
- ❌ *"Searches code."*
- ✅ *"Semantic search across repositories that finds code by meaning, not exact text. Returns ranked snippets with file paths and relevance scores. Use for conceptual queries such as 'where do we handle payment retries?'. Prefer Grep for exact strings or regex patterns."*

⚠️ **Anti-pattern:** leaving MCP tool descriptions sparse, so the agent ignores the tool.

---

## Quick-fire memory list

| Scenario in the question | Answer |
|---|---|
| Whole team needs a server | `.mcp.json` |
| Personal or experimental server | `~/.claude.json` |
| Need to activate a server's tools? | No. They're auto-discovered at connection time. |
| Credentials in config | `${VARIABLE_NAME}` syntax |
| Token rotation | Change the environment, not the config |
| Agent wastes calls discovering data | Expose a **resource** |
| Agent needs to fetch or modify data | Use a **tool** |
| Standard integration (GitHub, Jira, Slack, Notion) | Community server first |
| Proprietary system with no community server | Build custom |
| Agent ignores an MCP tool in favour of Grep | Enhance the tool description |

## Five anti-patterns to spot in answer options
1. Team-wide servers in `~/.claude.json`
2. Credentials committed in `.mcp.json`
3. Custom servers for standard integrations
4. Sparse MCP tool descriptions
5. Content catalogues not exposed as resources

---

Would you like a short mixed quiz on these points to test your recall before the exam?
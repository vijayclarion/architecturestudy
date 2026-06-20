# Solution Architect Interview Prep

## Folder Structure

```
ArchitectPrep/
├── 01-Caching/
│   ├── cache-aside.md
│   ├── cache-stampede.md
│   ├── cdn-strategies.md
│   ├── cache-invalidation.md
│   └── diagrams/
│       ├── cache-layers.mermaid
│       ├── cache-aside-flow.mermaid
│       ├── cache-stampede.mermaid
│       └── event-driven-invalidation.mermaid
│
├── 02-Databases/
│   ├── sql-vs-nosql.md
│   ├── cassandra-internals.md
│   ├── sql-vs-cassandra-partitioning.md
│   ├── consistent-hashing.md
│   └── diagrams/
│       ├── btree-vs-lsm.mermaid
│       ├── cassandra-write-path.mermaid
│       ├── consistent-hashing.mermaid
│       └── polyglot-persistence.mermaid
│
└── 05-Cheat-Sheets/
    ├── decision-trees.md
    ├── one-page-summaries.md
    └── flashcards-anki.txt
```

## How to Use

1. **First Pass:** Read each concept file top to bottom
2. **Active Recall:** Cover sections, try to explain out loud
3. **Diagrams:** Redraw each Mermaid diagram on paper
4. **Flashcards:** Import `flashcards-anki.txt` into Anki, review daily
5. **Practice:** Use decision trees during mock interviews

## Template Used for Each Concept

Every concept file follows this structure:
- Problem It Solves
- How It Works (with diagram)
- When to Use
- When NOT to Use
- Trade-offs
- Real World Example
- Interview Soundbite

## Mermaid Rendering

`.mermaid` files render automatically in:
- GitHub (paste content into a `.md` file inside ` ```mermaid ` block)
- Obsidian (native support)
- Notion (use `/mermaid`)
- VS Code (install Mermaid Preview extension)

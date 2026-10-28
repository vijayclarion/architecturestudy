# Topic 1: How the Context Window Works — Tokens, Limits, and Tradeoffs
- Everything a model does happens inside its context window — the fixed amount of text it can "see" at once
- Tokens are the unit. Models don't process characters or words — they process tokens, chunks of text (roughly ¾ of a word on average, but variable).
- The context window is a fixed token budget. The window has a maximum size
- Exceed it and something has to give. When content would exceed the window, it can't all be included — older content gets dropped/truncated, or the request fails
- The context window is a fixed token budget that everything (instructions, data, history, tool outputs, the response) shares; exceeding it means losing content, and filling it has real costs — more money, more latency, and potentially worse quality from diluted attention

# Topic 2: Context Window Economics — Cost, Speed, and Quality
- Cost scales with tokens. You p
- Speed (latency) scales with tokens. More tokens take longer to process, so a fatter
- Quality is non-monotonic with tokens — the key economic subtlety.
- spend the token budget deliberately: include what's relevant and valuable, exclude what isn't, because every token costs money, costs time, and (past relevance) costs quality


# Topic 3: Why Summaries Lose Facts
- The obvious fix for a bloated context (too much history, too many tokens) is to summarize — compress the long history into a short summary and carry that forward instead.
- But summarization has a dangerous property: it loses information, and often loses exactly the information you needed

# Topic 4: Designing a Persistent Fact Block

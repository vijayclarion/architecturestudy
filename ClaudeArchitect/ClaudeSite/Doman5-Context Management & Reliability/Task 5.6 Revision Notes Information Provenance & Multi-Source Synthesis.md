# Task 5.6 Revision Notes: Information Provenance & Multi-Source Synthesis

Oct 30, 2026 · @Vijay

## Purpose: why provenance matters

Information provenance means every claim in a final output can be traced back to the exact source it came from. In a multi-agent pipeline, it is what lets a human or downstream agent verify, trust and correctly interpret each finding.

**Key points**

- Attribution is not lost deliberately. It dies quietly during summarisation, when findings are compressed.
- Without provenance, conflicts between sources become invisible and the output shows false certainty.
- Without dates, valid trends are misread as contradictions or data quality issues.
- Provenance is what makes conflict handling and temporal reasoning possible at all.

**Example**

A pipeline has three agents: search, extract, synthesise. The final report states "The UK EV market grew 12%." A reviewer asks where that figure came from and when. If the synthesis agent dropped the source, nobody can verify it, check whether another source disagreed, or tell whether it is this year's figure or last year's.

**Remember:** Attribution dies during summarisation unless it is explicitly preserved.

## 1. Structured claim-source mappings

Every finding must carry five elements. Without them, attribution dies during summarisation.

| Element | What it is | Why it is needed | Example value |
| --- | --- | --- | --- |
| Claim | The specific assertion | Defines exactly what is being attributed | UK EV sales grew 12% |
| Source URL | Where the claim was found | Lets anyone go back and verify it | https://example.org/ev-report |
| Document name | Title of the source | Lets readers judge credibility at a glance | UK Electric Vehicle Market Review |
| Relevant excerpt | The supporting passage, in the source's own words | Evidence can be checked without re-reading the whole source | "Sales of battery EVs rose 12% year on year." |
| Publication date | When published or when data was collected | Lets differences be read correctly over time | 15 March 2024 |

**Rules**

- All five elements, for every finding. Not four, not "where available".
- An excerpt is not a description. A description is someone's summary; the excerpt is the source's own evidence.
- Page number is not one of the five elements.

**Memory aid:** C-U-D-E-D: Claim, URL, Document name, Excerpt, Date.

**Most-missed element:** publication date.

## 2. Conflict handling with dual attribution

When two credible sources give different values for the same thing and the same period, report both, each with full attribution and date, and let the consumer decide.

**Rules**

- Do NOT pick one value, whether the most recent, the most official or the most convenient.
- Do NOT average them. An average is a number neither source reported.
- DO annotate both values with source, document name and publication date.
- DO let the consumer decide. They may know which definition or method suits their purpose.

**Why:** selecting one value destroys information and presents false certainty. The agent rarely knows why the sources differ.

**Example**

Source A (industry report, March 2025) says the market is worth £4.2 billion. Source B (government statistics, June 2025) says £5.0 billion, for the same year.

- Wrong: "The market is worth £5.0 billion." (picked one)
- Wrong: "The market is worth £4.6 billion." (averaged)
- Right: "Estimates differ. An industry report (March 2025) puts it at £4.2 billion; government statistics (June 2025) put it at £5.0 billion."

**Remember:** Conflicting statistics get BOTH values, not a winner.

## 3. Temporal awareness and publication dates

Different publication or data collection dates often explain different numbers. That is a trend, not a contradiction.

**Rules**

- Publication or data collection dates are required in all structured outputs.
- Check the dates before calling anything a conflict.
- Without dates, temporal differences are misread as contradictions and valid trends are wrongly flagged as data quality issues.

**The decision test**

| Situation | What it is | What to do |
| --- | --- | --- |
| Same period, different numbers | Genuine conflict | Dual attribution: report both, let the consumer decide |
| Different periods, different numbers | Likely a trend | Report in time order with dates and describe the change |

**Example**

Source A (2023) reports 8% market growth. Source B (2024) reports 12% growth. A colleague says "These contradict each other; flag a data quality issue."

- Wrong: flag it as a contradiction, or treat it as a conflict to be decided.
- Right: "Growth was 8% in 2023 (Source A) and 12% in 2024 (Source B), suggesting it is accelerating."

**Remember:** Different dates explain different numbers. Check the date first.

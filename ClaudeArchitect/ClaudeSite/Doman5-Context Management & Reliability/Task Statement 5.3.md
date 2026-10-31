# Exam Notes: Task Statement 5.3 — Error Propagation in Multi-Agent Systems

*Domain 5: Context Management & Reliability*

---

## 1. Purpose

When one subagent in a multi-agent system fails, the failure must be **handled locally where possible, reported honestly and in detail where not, and made visible in the final output**, so that the coordinator can recover and the reader knows how complete the result is.

**Example:** A coordinator sends five subagents to research five energy topics. One hits a database timeout. Done well, the subagent retries, then reports a structured error; the coordinator tries an alternative source, and the final report covers all topics it can and flags the one gap.

---

## 2. Terms to Remember

| Term | Meaning | Example |
|---|---|---|
| **Structured error context** | The four-part report a failing subagent returns: failure type, what was attempted, partial results, alternatives | See full example in Section 4, Rule 2 |
| **Transient failure** | Temporary problem; retrying later may work | Timeout, rate limit |
| **Validation failure** | The input was bad; retrying unchanged won't help | Date range "2024–2020" (end before start) |
| **Business failure** | A rule of the system was violated | Request exceeds an allowed query scope set by policy |
| **Permission failure** | Access was denied; needs a different route | "403: not authorised for this journal" |
| **Partial results** | Work gathered before the failure | 12 papers retrieved before the rate limit hit |
| **Local recovery** | Subagent's own retries, fallbacks or degraded responses *before* reporting upwards | Retrying a rate-limited call with exponential backoff |
| **Access failure** | Tool could not reach the data source; answer is *unknown* | Connection to the database timed out |
| **Valid empty result** | Tool reached the source and found no matches; that *is* the answer | Database searched; zero matching records |
| **Silent suppression** | Hiding a failure by returning empty results marked as success | `{results: [], status: 'success'}` after a timeout |
| **Workflow termination** | Killing the whole pipeline because one subagent failed | 4 of 5 subagents succeed, 1 fails, everything discarded |
| **Coverage annotation** | A note in the final output stating which areas are well supported and which have gaps | "Geothermal section is limited due to unavailable journal access." |

---

## 3. Important Points

**Point 1: Failure type drives recovery.** The coordinator chooses its response based on *why* the failure happened.
*Example:* A transient timeout → retry later. A permission denial → retrying is pointless; try a different source instead.

**Point 2: Silent suppression is the worst anti-pattern.** It prevents *all* recovery, because the coordinator believes the search succeeded and genuinely found nothing. The output looks complete but has hidden gaps.
*Example:* A timeout disguised as success means the coordinator never retries, never tries alternatives, and the final report silently omits the topic.

**Point 3: Workflow termination wastes successful work.** It throws away completed results and offers no recovery path.
*Example:* Four topics fully researched are discarded because the fifth timed out; the user receives nothing.

**Point 4: Access failure ≠ valid empty result.** Treating them the same causes either unnecessary retries or missed recovery.
*Example:* Retrying a genuinely empty search wastes resources; accepting a timeout as "nothing exists" leaves a real gap unaddressed.

**Point 5: Silent suppression disguises an access failure as a valid empty result.** This is the link between the two concepts.
*Example:* `{results: [], status: 'success'}` after a timeout looks identical to "searched, no matches".

**Point 6: Honesty must reach the reader.** Handling errors well internally is wasted if the final synthesis silently drops the gap.
*Example:* Omitting the geothermal section without comment is effectively silent suppression at the output level.

---

## 4. Rules

**Rule 1: Recover locally first.** Subagents attempt retries, fallbacks or degraded responses for transient failures before propagating.
*Example:* Rate limit hit → retry three times with backoff → only then report upwards.

**Rule 2: Propagate only errors that survive local recovery, with all four elements.**
*Example:*
```
Failure type:        Transient (rate limit; persisted after 3 local retries)
What was attempted:  Query "geothermal energy efficiency", 2020–2024,
                     academic journal database
Partial results:     12 relevant papers (attached)
Alternatives:        Retry after cooldown; query an alternative database;
                     proceed with 12 papers and note limited coverage
```

**Rule 3: "What was attempted" describes the request**, meaning the query, parameters and target system, not just the number of retries.
*Example:* ✅ "Query X, date range Y, database Z" ❌ "3 retries"

**Rule 4: Alternatives are coordinator-level options**, not local techniques the subagent should already have used.
*Example:* ✅ "Try a different database" ❌ "Exponential backoff"

**Rule 5: Never report failure as success.**
*Example:* ✅ `{status: 'error', failure_type: 'transient', ...}` ❌ `{results: [], status: 'success'}`

**Rule 6: Never terminate the whole pipeline for one failure.** Keep successful results and continue.
*Example:* Deliver four completed topics, attempt recovery on the fifth, annotate if it remains unavailable.

**Rule 7: Retry access failures; accept valid empty results.**
*Example:* Timeout → retry. "Searched, 0 matches" → record as the answer, no retry.

**Rule 8: Annotate coverage in the final synthesis.**
*Example:* "Findings on solar, wind, hydro and tidal are well supported. The geothermal section is limited because the journal database was unavailable."

---

## 5. Exam Traps

| Trap (wrong answer) | Why it's wrong | Correct answer |
|---|---|---|
| Return generic "search unavailable" | Hides query, partial results and alternatives | Structured error with all four elements |
| Catch timeout, return empty as success | Coordinator never knows to recover | Report the failure honestly with its type |
| Stop the whole pipeline on one failure | Wastes completed work from other subagents | Continue with successful results; recover or annotate the gap |
| Retry every empty result | A valid empty result is already the answer | Retry only access failures |
| Silently omit a topic with gaps | Reader assumes the report is complete | Add a coverage annotation |

---

## 6. Memory Aid

**Recover locally → Report fully → Keep going → Tell the reader where the gaps are.**

---

I can also make this a doc you can edit here, share, or download as Word or PDF, if you'd like.
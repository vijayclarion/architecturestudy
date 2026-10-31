# Exam Notes: Task Statement 5.5 – Human Review & Confidence Calibration
*Domain 5: Context Management & Reliability*

## Overall purpose

To decide **safely** which AI extractions can be automated and which need a human, and to **keep checking** quality once the system is live. The core message is: **don't trust a single number, whether an overall accuracy figure or a raw confidence score, without breaking it down and checking it against reality.**

## 1. Aggregate Metrics Trap

**Purpose:** To stop you automating based on one overall accuracy figure that hides weak areas.

**Important points**
- An overall (aggregate) accuracy can look excellent while specific document types perform badly.
- This happens because the **high-volume, easy documents dominate the average**, so small, difficult groups barely move it.
- E.g. 97% overall can hide error rates of 40% or more on particular types.

**Rule:** Always validate accuracy **by document type AND by field** before automating.

**Example:**

| Document type | Share of volume | Accuracy |
|---|---|---|
| Standard invoices | Most of the volume | 99.5% |
| Scanned PDFs | Small | 72% |
| Handwritten receipts | Small | 60% |
| International documents | Small | 45% |

Overall accuracy is about 97%, but more than half of international documents are wrong. Automating everything based on "97%" would let those errors through unchecked.

## 2. Document Type and Field Segment Accuracy

**Purpose:** To give the level of detail needed to make a sound automation decision.

**Important points**
- Accuracy varies by **document type** (invoice, receipt, claim form) and by **field** (date, amount, name).
- The same field can perform very differently across document types.
- This breakdown is **required before any automation decision**.

**Rule:** Make automation decisions **per segment**, not one decision for everything. Automate the strong segments and keep human review for the weak ones.

**Example:** Extracting dates from invoices achieves 99%, but extracting dates from handwritten receipts achieves only 60%. You might automate invoice dates while sending receipt dates to human review.

## 3. Stratified Random Sampling

**Purpose:** To monitor quality continuously after automation and catch new problems early.

**Important points**
- Divide the work into **strata** (groups), such as document type, confidence band and field type.
- Take a **random sample from each stratum** for human checking.
- **You must sample HIGH-confidence items**, not just low-confidence ones.
  - Low-confidence items already go to human review.
  - High-confidence items are automated, so nobody looks at them. **They are the blind spot.**
- If a **novel error pattern** appears among high-confidence items, only stratified sampling will catch it.

**Rule:** Sample across **every** stratum, **including high confidence**.

**Example:** An insurer automates high-confidence motor claims. A hospital then changes its receipt layout, and the model starts misreading amounts while still reporting high confidence. Weekly sampling of high-confidence motor claims spots the new error. Sampling only low-confidence items would have missed it entirely.

## 4. Field-Level Confidence Calibration

**Purpose:** To find out what the model's confidence score **actually** means in terms of real accuracy.

**Important points**
- The model outputs a confidence score **for each field**. This is its own self-estimate.
- **Raw** scores (used as produced) are **not calibrated**, so they don't reliably match actual accuracy.
- The same score means different things for different field and document combinations.
- The model may be **systematically overconfident** on certain field types.

**Rule:** Calibrate confidence **per field type, per document type**, using ground truth data, before setting thresholds.

**Example:** The model reports 0.90 confidence on two fields:
- **Date fields:** actually correct 94% of the time (slightly underconfident).
- **Amount fields:** actually correct only 82% of the time (overconfident).

Using one threshold for both would be too cautious with dates and far too trusting with amounts.

## 5. Labelled Validation Sets

**Purpose:** To provide the ground truth needed for calibration.

**Important points**
- A **labelled validation set** is a sample of documents where humans have recorded the **correct values**. These correct values are the **ground truth**.
- You compare the model's output and confidence scores with the ground truth to see how often each confidence level is actually right.
- **Without calibration against ground truth, raw confidence is unreliable.**

**Rule:** No ground truth means no calibration, and no calibration means you can't trust the thresholds.

**Example:** Take 300 overseas travel claims and have humans record the correct claim amounts. Run the model on them. Of the amounts it scored around 0.90, only 82% match the ground truth, so 0.90 really means 82% for this field and document type. If you need 95% reliability, find the score that actually delivers 95% and route everything below it to review.

## 6. Reviewer Capacity Prioritisation

**Purpose:** To get the most error-catching from a limited human review team.

**Important points**
- Human reviewers are **limited**. There are usually more flagged items than time to check them.
- Route the **highest-uncertainty items first** (lowest calibrated confidence).
- **Do NOT spread capacity evenly**, since that wastes time on high-confidence items.
- **Do NOT review in chronological order**, since arrival time has nothing to do with risk.
- Use **dynamic ordering**: each reviewer always gets the highest-uncertainty item **still remaining**.

**Rule:** Lowest confidence first, always.

**Example:** 500 claims are flagged, and reviewers can check only 200.
- Claim X: arrived 9 a.m., confidence 0.85
- Claim Y: arrived 4 p.m., confidence 0.55

**Claim Y goes first**, because it's more uncertain and more likely to be wrong, even though it arrived later. Like A&E triage, the riskiest case is seen first.

## Anti-patterns to avoid

| Anti-pattern | Why it's wrong | Correct approach |
|---|---|---|
| Using aggregate accuracy to justify automation | Hides weak document types | Break down by document type and field |
| Sampling only low-confidence items | Misses novel errors in automated high-confidence items | Stratified sampling across all bands |
| Using raw confidence without calibration | Model may be overconfident on some fields | Calibrate with labelled validation sets |
| Spreading reviewer capacity evenly | Wastes time on low-risk items | Highest uncertainty first |
| Assuming consistent performance across types | Performance varies widely | Validate each type separately |

## Exam traps

1. **97% overall does NOT mean every document type achieves 97%.**
2. **Stratified sampling must include high-confidence items.** They are the blind spot.
3. **Raw confidence of 0.90 means different things** for different field and document combinations.
4. **Reviewer capacity is limited.** Prioritise the highest uncertainty, not chronological order or an even spread.

## Key terms

| Term | Meaning | Example |
|---|---|---|
| **Aggregate accuracy** | One overall accuracy figure across all documents | "97% overall" |
| **Document type** | Category of document | Invoice, handwritten receipt, travel claim |
| **Field / field segment** | A specific data item extracted | Date, amount, policy number |
| **Stratum (plural: strata)** | A group used for sampling | All high-confidence travel claims |
| **Stratified random sampling** | Random sampling from every stratum | Checking 20 items from each confidence band weekly |
| **Confidence band** | A range of confidence scores | High (0.90+), medium, low |
| **Raw confidence** | The model's score as produced, unchecked | Model says 0.90 |
| **Calibration** | Mapping raw scores to actual accuracy | 0.90 on amounts really means 82% |
| **Overconfident** | Score higher than actual accuracy | Says 0.90, right 82% of the time |
| **Labelled validation set** | Sample with human-verified correct answers | 300 claims with checked amounts |
| **Ground truth** | The verified correct values | The true claim amount |
| **Blind spot** | Area nobody checks | Automated high-confidence items |
| **Novel error pattern** | A new type of mistake not seen before | Errors after a form layout change |
| **Reviewer capacity** | How many items humans can check | 200 per day |
| **Dynamic ordering** | Always giving the next reviewer the most uncertain remaining item | Queue re-sorted by uncertainty |

## One-line memory aids

- **Break it down:** by document type and field, never trust the average.
- **Sample the confident:** high confidence is the blind spot.
- **Check the score:** raw confidence needs ground truth calibration.
- **Riskiest first:** lowest confidence goes to reviewers first.

I can also make this a doc you can edit, share, or download as Word or PDF, if you'd like.
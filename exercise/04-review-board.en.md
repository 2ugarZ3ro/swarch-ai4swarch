**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | the ADR's author | defends the ADR |
| **AI analyst** | | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | | role card P (business side) |
| **Site reliability engineer (SRE)** | | role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-…` by me (draft: `exercise/outputs/2b-adr.md`, AI review: `exercise/outputs/3-review.md`)
**Board members and roles:** group of 4: Architect (ADR author), AI analyst, Product owner (card P), SRE (card S). No security officer, so card X was not used.

**Decision:** accept with changes (3 to 1 after discussion; the SRE wanted to reject until the ops work is planned)

**Required changes:**
1. **Points on the confirmation page.** Marketing promised the customer sees points "the moment" they order. Keep the award async, but have checkout return the *calculated* points so the page can show them. Add reversal of points when an order is cancelled or refunded.
2. **Fraud scoring is not a consumer.** It has to cancel an order before confirmation, so it must be a synchronous step in `PlaceOrder`. Remove it from the "just add a consumer" list and say so in the ADR.
3. **Ops readiness and schedule.** Before go-live: outbox for publishing, dashboards, alerts, dead-letter queue with replay, a runbook, an order ID passed through every message for tracing. Ask for the 3-node broker budget now (2 weeks), fit the runbook work (about 1 week) into the 3 weeks before 11.11, and write a fallback if it is not ready.

**Who raised what?**

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Campaign promises points "the moment" of ordering, but async gives up to 1 minute | PO | No | No. The campaign is not in `scenario.md`. |
| 2 | Fraud scoring must cancel an order before confirmation, so it can't be an async consumer | PO | Partly (it guessed fraud scoring "might need a synchronous answer") | Only as a guess. The firm rule was not in its files. |
| 3 | Points need reversing on cancel/refund; double points cost real money (100 points = 10 THB) | PO | Partly (idempotency for double points yes, refunds no) | No for the money value and refund rule. Idempotency yes. |
| 4 | Nobody has run a broker in production; the two "RabbitMQ" developers used it on laptops; 2-person on-call | SRE | Partly (it noted only 2 of 4 know RabbitMQ and it is a new stateful component) | No. "Laptops only" and the on-call size are not in the scenario. |
| 5 | Ops rules: runbook, dashboards, alerts, backups (about 1 week); 3-node broker needs budget (2 weeks); sale in 3 weeks | SRE | No | No. These are internal facts. |
| 6 | Who is paged, how to find stuck messages, and how to trace "I never got my email"; last year's 20-minute email outage went unnoticed | SRE | Partly (queue-depth alerts and dead-letter queue were mentioned, tracing and paging were not) | Partly. Alerting is standard advice, but the outage history is not in its files. |
| 7 | Publish can fail after the order succeeds, so the event is lost; no committed outbox | AI review | Yes | Yes. It follows from the ADR text itself. |
| 8 | Duplicate delivery could send a second email or double-count analytics, not only points | AI review | Yes | Yes. It follows from the ADR text itself. |

**The AI's best finding** (one the humans had missed): the lost-event problem (concern 7) and duplicate emails/analytics (concern 8). Nobody on the board thought of them until the AI analyst read them out. The SRE agreed the outbox fixes the "I never got my email" case too.
**The AI's weakest finding** (generic, wrong or irrelevant): the suggestion of NATS JetStream or MailFast batching as other options. It adds a second new broker the team has never run, and ignores the 3-week deadline.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.

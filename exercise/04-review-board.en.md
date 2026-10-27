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

**ADR reviewed:** `docs/adr/0002-use-rabbitmq-for-post-order-events.md` by Ashira Aungsumal
**Board members and roles:**

| Role | Who |
|---|---|
| Architect | Ashira Aungsumal |
| AI analyst | Kittichon Chaonawig |
| Product owner | Peeravas Piboolvorakul |
| SRE | Kittichet Arayasujin |
| Security & privacy officer | Pasin Thanyakasikol |

**Decision:** accept with changes (unanimous, 5 of 5; the AI analyst only read out the AI review's verdict, "Accept with changes"). The ADR goes back to `Proposed` until all changes are made.

**Required changes:**
1. **Publish-failure policy:** the order must not fail. Add a fallback or reconciliation so that a paid order still gets its email and loyalty points, set a bounded publish timeout, and test the "publish fails after payment" case.
2. **Size the email path:** the business agrees a maximum acceptable delay; size the number of consumers for 50 msg/s at 1.5 s per email; add a queue-length cap and backlog alerting; run a realistically long peak load test; make the email consumer de-duplicate by `order_id`.
3. **Specify HA and data handling:** queue type and replica count, TLS, per-service credentials, minimise event data to what each consumer needs, retention, DLQ owner, alerting and replay, on-call owner and runbook, rollback plan, and confirm a data processing agreement with MailFast before go-live.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Publish can fail after payment; the order must not fail, so a fallback is needed | AI review (F4); PO (PO2) | Yes (F4), but not the business cost per member | Technical gap: yes, visible in the ADR. Business cost: no, needs the loyalty promise to members and cost data |
| 2 | Unbounded email backlog, ~90 min to drain, no agreed maximum delay | AI review (F5, F10); PO (PO1); SRE (SRE2) | Yes (F5, F10), but not the customer impact (repeat orders, support contacts) | Numbers: yes, from the ADR. Buyer behaviour: no, needs product and sales knowledge |
| 3 | Duplicate emails, because the email consumer is not idempotent and the provider charges per email | PO (PO3) | No (F2 covers loyalty only) | Duplicates: probably, from the ADR. Billing: no, the provider's terms are not in the inputs `[ROLE ASSUMPTION]` |
| 4 | MailFast is a third party receiving customer data: contract, storage location, retention | Security (SEC2) | No | Only that MailFast receives email addresses. The contract and retention are not in the ADR or the scenario |
| 5 | Broker memory/disk alarms block publishers and can hang `PlaceOrder` | SRE (SRE2) | Partly (F9 confirm timeout, F5 growth), not the blocking | Unlikely; needs hands-on RabbitMQ operating experience `[ROLE ASSUMPTION]` |
| 6 | No on-call owner, runbook, node-failure drill or rollback plan before 11.11 | SRE (SRE1, SRE3) | Partly (F12 DLQ owner, F11 stateful), not the operational side | No; the team's processes are not in the inputs |
| 7 | The event sends the email address and order details to every consumer, and the DLQ stores personal data | AI review (F7); Security (SEC1, SEC3) | Yes (F7 and required change 3), but not TLS, credentials or retention | Data content: yes. Security settings: it could see the ADR is silent on them, but not the details |
| 8 | A single email consumer is a single point of failure | AI review (F6) | Yes | Yes, from the numbers in the ADR |

**The AI's best finding** (one the humans had missed): F5 and F6, which turned the ADR's numbers into a ~90-minute drain time and ~75 email calls in flight. The architect had only stated this vaguely, and the PO and SRE reused those numbers in the discussion.
**The AI's weakest finding** (generic, wrong or irrelevant): F12, a long unprioritised list of "what's missing" (other brokers, outbox pattern, peak duration). Also F3 ("2 of 4 developers know RabbitMQ") looked like a non-risk, but the SRE argued that knowing a tool is not the same as running a broker on call `[ROLE ASSUMPTION]`.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.

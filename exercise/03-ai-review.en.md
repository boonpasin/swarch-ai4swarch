**Language:** [ภาษาไทย](03-ai-review.md) | English

# Task 3: Get the AI's review (approximately 5 min)

> **Lecture link:** P5 human–AI collaboration: the AI as *Chief Analyst*. A 2026 study found LLMs often matched or beat student teams at spotting ATAM risks and trade-offs ([arXiv 2603.28914](https://arxiv.org/abs/2603.28914)). Part B tests that claim.

Start a new chat or session and send, with the path of **your final ADR**:

```
Follow prompts/3-review-adr.md
adr: docs/adr/0002-<your-title>.md
```

Want the review in Thai? Add a line `language: Thai` (finding types such as Risk and Trade-off point stay in English).

The agent writes a one-page ATAM-style review to [`outputs/3-review.md`](outputs/3-review.md). It must not edit your ADR; if it does, reject or undo the change.

**Don't act on the review yet.** Read it, commit it, and bring it to the review board in Part B. The board will decide which of its findings matter.

- **My first impression of the AI's review** (one sentence): Sharper than expected and specific to this ADR: it caught that the 3-node quorum cluster doesn't protect events held in a `checkoutservice` pod's memory, and that dead-lettering after 5 attempts would turn a MailFast outage into a 500k-message DLQ. But it can only check the ADR against `docs/scenario.md`, so it can't know business or operations facts the board may bring (e.g. whether MailFast's limit can be raised or who runs RabbitMQ).

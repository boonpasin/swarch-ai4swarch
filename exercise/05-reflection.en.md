**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Pasin Thanyakasikol / 6732025921
**My role on the review board:** Security & privacy officer

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI with evidence-bound work: listing dependencies from manifests, drafting options, and checking an ADR against its own numbers. In Task 1 all 17 of Run B's dependencies were correct. But AI only knows what it reads: it missed the HTTP-only `shoppingassistantservice` because no manifest declared it.

At the board, I raised MailFast as a third party holding customers' email addresses (SEC2). The AI review never mentioned it, and it could not have known whether we have a data processing agreement, where MailFast stores data, or for how long: none of that is in the ADR or the scenario. It did flag that the event sends email addresses to every consumer and the DLQ (F7), a useful first pass, but missed TLS, per-service credentials and retention. AI can review against a checklist; a human must confirm compliance and own the privacy risk.

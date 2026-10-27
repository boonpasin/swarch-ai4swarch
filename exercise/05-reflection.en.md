**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Pasin Thanyakasikol / …
**My role on the review board:** Security & privacy officer

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI with evidence-bound work: listing dependencies from manifests, drafting options, and checking an ADR against stated numbers. In Task 1 every one of Run B's 17 dependencies was correct, and the AI review of my ADR caught a real flaw: an event-loss window in an in-memory buffer. But AI only knows what it reads. It missed the HTTP-only `shoppingassistantservice` because no manifest declared it, and its ADR draft quietly broke "exactly once" while claiming to meet it. Plausible is not the same as correct.

At the board, [EXAMPLE: the security/privacy concern I raised from role card X, e.g. "…", which the AI review did / did not mention]. The AI could not have known this because [reason, e.g. it was never in `docs/scenario.md`]. Weighing context like that, and owning the final decision, stays with humans.

**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** (fill in)
**My role on the review board:** Architect (ADR author)

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI with the first pass: reading code to recover the architecture, drafting an ADR, and reviewing it for gaps. In our board meeting the AI found the lost-event problem and the duplicate emails, which I had missed. But it could not know that marketing had already promised customers they would see their points "the moment" they order, that nobody had run a broker in production here, or that fraud scoring must cancel an order before confirmation. My ADR looked complete and the AI review looked reasonable, but both were plausible rather than appropriate. I would keep the business and operations judgment for humans, and the decision itself. The AI is a good reviewer of what is on the page, not of what is missing from it.

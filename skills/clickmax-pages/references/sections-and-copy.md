# Structure and copy

## The mandatory spine

Five blocks, in this order. It works for a lead magnet, a course, a service, a SaaS, an ebook, or consulting:

1. **Hero** — the promise, one line of context, and the primary CTA marked `data-cx-cta`
2. **Proof** — real credibility, or a clearly-labeled layout preview of it
3. **Value** — three cards of what the person gets
4. **Objections** — three to five questions in `<details>`
5. **Close** — the final CTA, with the **same label** as the hero

Nothing beyond this enters without real information from the user.

## The two modes

**Capture** — the whole spine, a form in the hero or in block 5, **no price**. Deliberately short; less converts more. Ask for 1–2 fields (email, or name + email). Go to three only if the user says a phone number is needed.

**Sales** — the spine plus an **offer block** between 3 and 4 (what's included, guarantee, price, CTA). Proof can run longer. No form; the CTA goes to checkout.

## Optional blocks — only with real data

Schedule · modules and lessons · instructors · certificate · bonuses · case studies · comparison table · video.

Never generate these to fill space. Five honest blocks convert better than fifteen with placeholders, and a published placeholder is what makes a client stop trusting generated pages.

## Page jobs and default progressions

|Page job|Default progression|
|-|-|
|Sales|Hero → proof/mechanism → problem/alternatives → solution → offer → proof → guarantee → Q&A → CTA → footer|
|Presell/advertorial|Editorial hero → lead → discovery/story → evidence → mechanism → transition → CTA → footer|
|Capture|Hero → value of the magnet/event → proof → minimal capture area → friction reducer → footer|
|Upsell|Continuity headline → why now → incremental outcome → offer → proof → guarantee → single CTA|
|Downsell|Acknowledge the hesitation → smaller alternative → retained outcome → adjusted terms → guarantee → CTA|
|Checkout|Real urgency → offer summary → checkout area → proof → guarantee → Q&A → terms → footer|
|Thank-you|Confirmation → next action → access expectations → support → optional secondary step|
|Terms/privacy|Clear title → dated structured clauses → entity and contact → footer|

## Section jobs

|Section|Job|Required input|
|-|-|-|
|Urgency bar|Communicate a real deadline or limit|Verifiable scarcity only|
|Hero|Outcome + mechanism + primary action|Offer and audience|
|Subheadline|Clarify the promise, reduce first friction|The main objection|
|Lead|Name the situation, earn continued attention|Pain and goal language|
|Pitch|Why old attempts fail and why this mechanism differs|Alternatives, beliefs, mechanism|
|Story|Problem → failed attempts → discovery|A true narrative|
|Proof|Support the claim next to it|Approved testimonials, metrics, demonstrations|
|Offer|Deliverables, bonuses, terms, value|Approved commercial facts|
|Guarantee|Reverse the risk with concrete terms|An approved guarantee|
|Q&A|Resolve remaining objections|Real objections and real policy|
|CTA|One action, one benefit|A real destination|
|Footer|Legal, identity, contact|Approved legal data|

## Copy rules

- **Five-second test**: someone who has never heard of the product understands the offer from the hero alone.
- Sentences of 15–20 words. Active voice. No jargon.
- **One primary CTA, repeated three times, with the same label.** Competing CTAs ("watch the demo", "download the guide", "talk to sales") split the decision and drop conversion.
- The CTA label states the benefit, not the mechanic: "I want the 15-minute routine" beats "Submit".
- **No navigation at the top.** A link out of the page is a lost conversion. The footer carries legal and contact only.
- Headline = a specific promise plus the distinct mechanism. A promise that would fit any business sells none.
- Offer = value stack plus terms plus an honest anchor against the expensive alternative. Price without the inclusion list is a price with no context.
- Every section must advance belief, proof, desire, understanding, or action. Remove anything decorative that does not.
- Put proof next to the claim it supports, and friction reducers next to the action they protect.

## Proof, and what to do when there is none

**Never fabricate a testimonial, a student count, a rating, a logo, a metric, a deadline, a price, or a guarantee.** These are the elements a real buyer checks, and a fake one is visible.

Without proof, two honest options:

1. Omit the block and tell the user which fact is missing.
2. In an unpublished draft only, render a **layout preview**: the real final component, styled with the page's own radius, typography, and grid, but with a visible "Layout preview — replace before publishing" label, a dashed or otherwise distinct draft treatment, and literal slots — "Customer name", "Role or context", "Insert a verified testimonial", "Verifiable number".

```html
<section class="proof-preview" data-draft-placeholder="proof" aria-label="Proof layout preview">
  <p class="proof-preview__label">Layout preview — replace before publishing</p>
  <div class="proof-preview__grid">
    <article class="proof-card">
      <div class="proof-card__avatar" aria-hidden="true"></div>
      <blockquote>Insert a verified customer testimonial here.</blockquote>
      <strong>Customer name</strong>
      <span>Role or relevant context</span>
    </article>
    <article class="proof-metric">
      <strong>Verifiable number</strong>
      <span>Students, customers, rating, or measured result</span>
    </article>
  </div>
</section>
```

Localize the labels to the page's language, and style the preview with the same radius, typography, surfaces, and responsive grid as the rest of the page — a preview that looks unfinished gets published by accident less often than one that looks broken.

Never render a bracketed sentence as a whole section. Before publishing, every slot is replaced with a fact or the block is removed. If the user asked for a publish-ready page and there is no proof, omit it.

## Anti-patterns

- Copying a dated-launch structure (schedule + certificate + instructors + scarcity) onto a product that is not a dated event.
- Padding the page with blocks to look professional.
- A hero generic enough to fit any product.
- More than one form on the page.
- Notes like "add a testimonial here" left in the final copy.

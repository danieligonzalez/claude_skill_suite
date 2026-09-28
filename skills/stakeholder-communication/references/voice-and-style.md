# Voice and style: worked examples

What a finished stakeholder doc from an analytics engineer should read like. The excerpts below are composites modeled on real, approved docs and recast into a neutral `customers` + `payments` warehouse. Every figure in them is illustrative. Match the register; do not copy the content.

The two exemplars sit at different points on a range, and **voice flexes with the output type**:

- **Exemplar A, the tight informational update.** A finance and BI update on retiring a legacy payment channel and introducing a `payment_channel` attribution. Terse, evidence-first, business-owner audience. Reach for this register for status updates and short informational notes.
- **Exemplar B, the research / context package.** A "Customers: Country Identification" doc that sets shared understanding, teaches how the transformation logic works, breaks down findings, and closes with recommendations. Warmer, first-person, semi-technical or mixed audience. Reach for this register for research and context packages (Step 0 output #4) and anywhere the reader needs to *understand* something before they can weigh in.

**The through-line across both:** explanatory prose in the author's own voice. Never a bullet-point fact dump, never a corporate or AI-flavored recap. Honest about what is inferred. Plain-language explanation of any mechanism the reader needs.

---

# Universal qualities (both registers)

## 1. Narrative prose, not a bullet-fact dump

The single most important quality. Each section **leads with the *why* and reasons through it in flowing paragraphs.** Tables and bullets are reserved for genuinely dense data (a breakdown of groups, a list of definitions), never used as a substitute for explanation.

**Do this** (Exemplar A):

> The pre-2018 group is the one worth understanding, because it is nearly all of the revenue in question. Every online payment carries a marker, a checkout-session id, that flags it as having come through the web store; that id is what the Web channel keys on to catch it. The problem is the id did not start getting recorded until **March 2018**. Every online payment placed before that has no session id at all, so nothing identifies it as web, and it falls through to `Legacy` by default. These are not in-store payments sitting in the wrong place: 94% carry the web order-number format, 99% are settled payments, and 98% shipped.

**Not this** (same facts, dumped):

> - Session ids began March 2018.
> - Pre-2018 web payments had no id, fell to `Legacy`.
> - 94% carry web order-number format; 99% settled; 98% shipped.

The facts are identical. The first tells the reader *what was happening and why it matters*; the second makes them reconstruct it.

## 2. Explain the mechanism, don't just strip the jargon

Stripping engineering vocabulary (`ref()`, grain, SCD2, incremental) is necessary but not sufficient. Where a mechanism is load-bearing to understanding, **spend a sentence or two, or a worked example, explaining what it is** in plain terms.

Naming what a grouping is and where it lives (Exemplar A):

> `Other` is not a channel produced by the payments model. It is a grouping applied downstream, in the reporting layer. The full channel list gets rolled up into the familiar high-level buckets (Web, In-store, Marketplace, Phone, Partner) and anything that does not fit one of those falls into `Other`.

Teaching a transformation with a concrete example (Exemplar B):

> A list of roughly 200 countries is defined at the top of the model. Each country entry includes all the common ways it might be written. The United Kingdom entry, for example, covers "United Kingdom," "U.K.," "UK," "Britain," "England," "Scotland," and "Wales." When the model runs, it generates a rule for every country on that list, checking whether the raw country value on the customer record matches any known variation.

> Before matching, the city field is trimmed to its first token when it carries a region suffix, so "Denver, CO" becomes "Denver" and the state is read from its own field rather than guessed from the city.

The reader comes away understanding *how it actually works*, not just that it does.

## 3. Honest about what is inferred; label reliability

Tell the reader plainly which claims are measured and which are inferred, especially the number a decision hangs on. When you bucket data, say how much to trust each bucket.

Marking the load-bearing number (Exemplar A):

> Since the session id simply did not exist back then, this read is inferred from the order-number format and fulfillment behavior rather than a flag we can point to. It also *reverses* an earlier working assumption that pre-2018 was in-store, which would have moved roughly $40M of web revenue into the In-store channel.

Reliability judgment on each category (Exemplar B):

> **Same as source.** The country on the record already matched a known country name. These are the most reliable records, since the data was correct at the source.
> **Cleaned via the state field.** No usable country value, so the country was inferred from a recognized state or province. These records carry the most uncertainty, since the country was inferred rather than stated.

## 4. Pronouns and stance

Consistent across both registers:

- **First-person "I"** for your own analysis, choices, and recommendations: "As I was analyzing the data," "I would recommend the following." Natural and good, especially in research packages.
- **Collective "we / our"** for shared team context and concerns raised together: "to be on the same page with our understanding," "we have started the process of auditing."
- **Never attribute a concern or ask back to the reader.** "This came up as a concern" and "We discussed," not "You raised" or "You asked." Keep the object general ("in reporting," not "in your reports").

**Confirm the framing at intake, always.** Who "we" is, whether the reader is inside that "we" or is a separate audience being addressed, and how much to use first-person all depend on who is sending the doc and to whom. This is a required Step 0 question (see SKILL.md), so nail it down before drafting rather than inferring silently. Getting we-vs-you wrong is jarring and cheap to avoid.

---

# Register A: the tight informational update

Terse, evidence-first, no warm-up. Additional moves specific to this register:

**Investigative, non-definitive tone on validation.** Tell the story of the check; let the result land as an observation, not a flat verdict. Name the signals and checks without dumping the full apparatus.

> This came up as a concern: that marketplace payments (orders placed through third-party marketplaces) might be slipping into `Other`. So we ran some checks. A marketplace payment only lands in a marketplace channel because it carries a concrete signal: a storefront flag, a line-item flag, a marketplace customer record, or a marketplace tag. So we took every channel in `Other` and checked each payment for any of those signals, then ran it in reverse, taking every payment with a marketplace signal and checking where it landed. Neither direction turned anything up. From what we can see, nothing marketplace is sitting in `Other`.

Note "From what we can see" and "do not appear to have been." Honest confidence, not overclaim.

**Close the loop on meeting action items, end with a clear ask.** A "what's next / who does what" section ties back to what was assigned, states your own work, and names the specific stakeholder asks. End with the explicit ask, including "no action needed" when that is the truth:

> Nothing needed today. Reach out if any of the reporting shifts above touch a number the team relies on and we will walk through it before user acceptance testing.

**Structure A:** brief meeting-follow-up intro; one section per change (each: what it is, why, what it means for reporting); validation section; "what's next and who does what"; linked detail docs.

---

# Register B: the research / context package (teaching voice)

Warmer and more first-person than A. The reader is being brought up to speed so they can give useful input or make a call. Distinctive moves:

**Set shared understanding first, and state the collaborative intent.** Open by establishing why this exists and where it is headed, explicitly:

> In order to be on the same page with our understanding, I will provide some context for what is currently available in the staging and mart layers.
> Ultimately the intent of this explanation is a collaboration between the analytics engineering team and the source-system teams to create a more robust solution that unlocks reliable reporting.

**Credit prior work honestly; history without blame.** Name who did what and why, and where it falls short, matter-of-factly:

> The engineer originally assigned this request found many inconsistencies in how the country is presented on customer records. He implemented a series of data transformations that attempt to assign a country from whatever fields are populated. While this process did assist in providing directional insight, it is by no means the most reliable or sustainable solution.

**Walk the logic end-to-end, example-driven.** Break a transformation into its tools and steps and teach each one with a concrete example (see the country-matcher and "Denver, CO" excerpts under Universal quality #2). A numbered priority order is appropriate *here* because it mirrors how the model actually resolves; the bullets carry real explanatory content, not stripped facts.

**Define each segment with plain meaning plus reliability, so findings are interpretable.** Before showing counts, give the reader the vocabulary and how far to trust each bucket (see the category excerpts under Universal quality #3).

**Invite the reader deeper, then close on directional recommendations.** Offer paths to go further, then land concrete "we should" improvements:

> If you would like to analyze further or ask follow-up questions, we can make the data available in a dashboard or an exploration view to accommodate.
> I would recommend the following: each customer record should have a country specified at entry; the shipping address should be populated even when it is the same as the billing address; and we should ingest the canonical country reference table from the source system rather than maintaining our own list.

**Structure B:** overview (context, history, the problem, collaborative intent); how the logic works (tools, step by step, how the reference data is built); findings (segments defined with reliability, then counts and revenue); conclusion and recommended improvements.

---

Both drafts land as a local markdown file or inline by default. Inverted-pyramid overall for A, context-building order for B. Cut filler either way.

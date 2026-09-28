# Voice and style: worked examples

What a finished stakeholder doc from an analytics engineer should read like. The excerpts below are fictional, written for a neutral `customers` + `payments` warehouse to illustrate two registers. Every scenario and figure in them is invented. Match the register; do not copy the content.

The two exemplars sit at different points on a range, and **voice flexes with the output type**:

- **Exemplar A, the tight informational update.** A finance and BI update after moving subscription billing to a new payment provider, explaining why the recurring-revenue metric doubled and what was done about it. Terse, evidence-first, business-owner audience. Reach for this register for status updates and short informational notes.
- **Exemplar B, the research / context package.** A "Customers: Household Matching" doc that sets shared understanding, teaches how the matching logic works, breaks down findings, and closes with recommendations. Warmer, first-person, semi-technical or mixed audience. Reach for this register for research and context packages (Step 0 output #4) and anywhere the reader needs to *understand* something before they can weigh in.

**The through-line across both:** explanatory prose in the author's own voice. Never a bullet-point fact dump, never a corporate or AI-flavored recap. Honest about what is inferred. Plain-language explanation of any mechanism the reader needs.

---

# Universal qualities (both registers)

## 1. Narrative prose, not a bullet-fact dump

The single most important quality. Each section **leads with the *why* and reasons through it in flowing paragraphs.** Tables and bullets are reserved for genuinely dense data (a breakdown of groups, a list of definitions), never used as a substitute for explanation.

**Do this** (Exemplar A):

> The post-cutover group is the one worth understanding, because it is where the doubling comes from. The old provider sent us one record per subscription payment, written when the card was charged. The new provider sends two: one when the charge is authorized and a second, a few seconds later, when it is captured. Both records carry their own payment id, so the dedup step that keys on payment id sees two distinct payments and keeps both. From the **October 2023** cutover onward, every subscription payment is counted twice. These are not real second charges: 97% of the pairs are timestamped within five seconds of each other, 100% share an invoice number, and 99.6% match to the cent.

**Not this** (same facts, dumped):

> - New provider sends authorization + capture records, separate ids.
> - Dedup keys on payment id, keeps both since Oct 2023.
> - 97% within 5s; 100% same invoice; 99.6% match to the cent.

The facts are identical. The first tells the reader *what was happening and why it matters*; the second makes them reconstruct it.

## 2. Explain the mechanism, don't just strip the jargon

Stripping engineering vocabulary (`ref()`, grain, SCD2, incremental) is necessary but not sufficient. Where a mechanism is load-bearing to understanding, **spend a sentence or two, or a worked example, explaining what it is** in plain terms.

Naming what a grouping is and where it lives (Exemplar A):

> `Unearned` is not a status produced by the payments model. It is a label applied downstream, in the finance reporting layer. The payment statuses roll up into three buckets there (Recognized, Deferred, Refunded), and any payment whose service period starts after the report date lands in `Unearned` regardless of its status.

Teaching a transformation with a concrete example (Exemplar B):

> The matcher runs three passes in a fixed order, and a customer record is linked on the first pass that finds a partner. The first pass compares email addresses after lowercasing them and stripping dots from the local part, so "J.Smith@Example.com" and "jsmith@example.com" are treated as the same address. The second pass compares phone numbers after reducing them to digits, so "(555) 010-2233" and "555.010.2233" both become "5550102233". The third pass compares last name plus postal code, and only runs for records that neither earlier pass could place.

The reader comes away understanding *how it actually works*, not just that it does.

## 3. Honest about what is inferred; label reliability

Tell the reader plainly which claims are measured and which are inferred, especially the number a decision hangs on. When you bucket data, say how much to trust each bucket.

Marking the load-bearing number (Exemplar A):

> Neither record is labeled as an authorization or a capture, so this read is inferred from the timing and the shared invoice number rather than a field we can point to. It also *reverses* an earlier working assumption that the step-up was real growth from the annual price change, which would have overstated fourth-quarter recurring revenue by roughly $1.8M.

Reliability judgment on each category (Exemplar B):

> **Matched on email.** Both records carry the same normalized email address. These are the most reliable links, since an email address is unique to a login.
> **Matched on last name and postal code.** No email or phone in common; the link rests on a shared surname within a postal code. These carry the most uncertainty, since two unrelated people can share both.

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

> This came up as a concern: that the fix might be collapsing genuine refunds into their original payments, since a refund also lands seconds after a charge and references the same invoice. So we ran some checks. A refund only carries the refund status because it has a concrete signal: a negative amount, a refund reason code, a reference to the original payment id, or a chargeback flag. So we took every collapsed pair and checked both records for any of those signals, then ran it in reverse, taking every refund in the period and checking whether it had been paired with anything. Neither direction turned anything up. From what we can see, no refunds are being absorbed by the dedup.

Note "From what we can see." Honest confidence, not overclaim.

**Close the loop on meeting action items, end with a clear ask.** A "what's next / who does what" section ties back to what was assigned, states your own work, and names the specific stakeholder asks. End with the explicit ask, including "no action needed" when that is the truth:

> Nothing needed today. Reach out if the fourth-quarter restatement touches a number that has already gone to the board and we will walk through the bridge before month-end close.

**Structure A:** brief meeting-follow-up intro; one section per change (each: what it is, why, what it means for reporting); validation section; "what's next and who does what"; linked detail docs.

---

# Register B: the research / context package (teaching voice)

Warmer and more first-person than A. The reader is being brought up to speed so they can give useful input or make a call. Distinctive moves:

**Set shared understanding first, and state the collaborative intent.** Open by establishing why this exists and where it is headed, explicitly:

> In order to be on the same page with our understanding, I will provide some context for how customer records reach the staging and mart layers and where duplicates enter.
> Ultimately the intent of this explanation is a collaboration between the analytics engineering team and the account and web teams to stop duplicates at the point of signup rather than repairing them in reporting.

**Credit prior work honestly; history without blame.** Name who did what and why, and where it falls short, matter-of-factly:

> The analyst originally assigned this request found that customers routinely create a second account when a password reset fails, so one household can appear as three or four customers. She built a set of matching rules that attempt to link those accounts after the fact. While this process did assist in providing directional insight into repeat-purchase rates, it is by no means the most reliable or sustainable solution.

**Walk the logic end-to-end, example-driven.** Break a transformation into its tools and steps and teach each one with a concrete example (see the three-pass matcher excerpt under Universal quality #2). A numbered priority order is appropriate *here* because it mirrors how the matcher actually resolves; the bullets carry real explanatory content, not stripped facts.

**Define each segment with plain meaning plus reliability, so findings are interpretable.** Before showing counts, give the reader the vocabulary and how far to trust each bucket (see the category excerpts under Universal quality #3).

**Invite the reader deeper, then close on directional recommendations.** Offer paths to go further, then land concrete "we should" improvements:

> If you would like to analyze further or ask follow-up questions, we can make the matched pairs available in a dashboard or an exploration view to accommodate.
> I would recommend the following: signup should check the normalized email against existing accounts before creating a new one; phone numbers should be captured in a single standard format at entry; and we should ingest the identity provider's login table so identity comes from an authoritative source rather than being inferred from profile fields.

**Structure B:** overview (context, history, the problem, collaborative intent); how the logic works (tools, step by step, how each pass is built); findings (segments defined with reliability, then counts and revenue); conclusion and recommended improvements.

---

Both drafts land as a local markdown file or inline by default. Inverted-pyramid overall for A, context-building order for B. Cut filler either way.

---
name: stakeholder-communication
description: "Get the right analytics information to the right stakeholder in a form they can act on. Use when writing a sprint or project status update, a decision request, a delay or data-quality (bad news) note, a leadership readout, a request for help or resources, or when packaging research so a business owner has enough context to give input or make a call. Always starts by asking which output is needed, then branches. Triggers: stakeholder update, status update, sprint update, decision brief, decision request, communicate a delay, share this with the business owner, research package for stakeholders, exec readout, leadership update, manage up, how do I explain this to a non-technical audience."
---

# stakeholder-communication

Get the right analytics information to the right stakeholder, in a form they can act on. The substance is usually a data model, a metric definition, a pipeline issue, or a design decision that a business owner needs to weigh in on. Stakeholders live in the ticketing system, the wiki, email, and chat. The skill drafts for whichever of those the user names and publishes to none of them unless asked.

The single most important behavior: **this skill does not assume the output.** A weekly status update, a decision request, and a research package for a department owner are different deliverables with different structures. Step 0 asks what the user actually needs and branches from there.

One framing rule holds across every output: the takeaway goes first, the "so what" is stated explicitly, and the ask is never buried.

## Critical rules

- **Always run Step 0 first.** Ask what output is needed before drafting anything. Never guess the format from a vague request.
- **Never fabricate.** Every number, date, and claim comes from the user, a document, or a verified query. This applies to every output, no exceptions. Many business rules live in the data rather than in documentation and are inferred, not stated; do not assert a meaning you have not grounded, and say plainly when a read is inferred.
- **Label confirmed vs. unconfirmed for decision-bearing outputs.** For decision requests, research and context packages, and bad-news notes (anywhere the stakeholder acts on the numbers), explicitly mark what is measured vs. assumed vs. needs-owner-confirmation. A lightweight status update does not need the full labeling apparatus; it just must not invent anything.
- **When drawing on existing research docs, restructure and reframe freely, but do not alter the underlying facts or figures.** Preserve them verbatim. Only wording and structure change.
- **Match the audience's detail tolerance.** A department owner gets outcomes and plain language; an engineering peer gets the technical detail and the decision log. Do not send the same register to both.
- **State the ask explicitly**, including who decides and by when. "No action needed" is a valid, required ask.
- **Recommend, don't just present.** For any decision request, give a recommendation with a reason. Not recommending puts the analytical work back on the stakeholder.
- **Do not publish on the user's behalf.** Draft inline or to a local markdown file by default. Post to the wiki, a ticket, email, or chat only when explicitly asked.
- **Plain declarative prose, no em-dashes** (CONVENTIONS.md rule).

## Step 0: intake, what output do you need?

Before anything else, ask the user which deliverable they want. Offer these and let them pick one, or describe their own:

1. **Sprint / project status update.** A recurring progress note to the team lead, product owner, or a business sponsor.
2. **Decision request.** A specific decision is needed; present options, tradeoffs, and a recommendation.
3. **Bad-news / delay / data-quality note.** A miss, a slip, or a discovered data problem that stakeholders need to know now.
4. **Research / context package for stakeholder input.** The stakeholder needs enough understanding of an issue to give useful input or make a call. The headline use case; see the dedicated branch below.
5. **Leadership / exec readout.** An upward summary for a director or VP: headlines, confident, detail available on click.
6. **Ask for help / resources.** A specific request: a decision right, another team's input, time, access.

**Also confirm all of the following before drafting. They are required, not optional.** Batch them into the same intake pass rather than dripping them out one at a time, so the draft lands right the first time. Favor more clarifying questions upfront over back-and-forth on drafts. Skip a question only when the user has already answered it or the context makes the answer unambiguous.

- **Audience.** Who specifically (named person or role), and their detail tolerance.
- **Medium.** Ticket comment, wiki page, local doc, email, chat message, or a talking point for a sprint review.
- **Timing.** When they need it.
- **Framing (required).** Who is "we" (the sender), and is the reader inside that "we" or a separate audience being addressed? This drives first-person vs. collective voice and whether the doc ever says "you." Always confirm this, even with a quick "sending as you, to the finance and BI owners?", rather than inferring silently. Getting we-vs-you wrong is jarring and cheap to avoid. See [references/voice-and-style.md](references/voice-and-style.md).

Then branch to the matching template in [references/stakeholder-templates.md](references/stakeholder-templates.md) and the guidance below.

## The five questions (answer before drafting any output)

1. **Who is the audience?** Be specific. For each named audience: what they care about (revenue, risk, data quality, velocity, cost, a specific report), what they already know, their detail tolerance, and the action you want.
2. **What is the headline?** If they read one sentence, what should they take away? It goes first, always.
3. **What is the so-what?** Answer the implicit "and?". "We hit 80% of the milestone" becomes "which puts the report one sprint behind plan."
4. **What do you want?** Inform, decide, escalate, or ask. State which.
5. **What format and medium?** Async written (wiki, doc, email) for anything with a decision trail. Sync (chat, meeting) only for debate and alignment. Mixed (doc read in advance plus a meeting) for real decisions.

## Structure: inverted pyramid

```
Headline (one sentence)
The "so what" and the request (one paragraph)
Status / progress / evidence (the body)
Risks and asks (what we need)
Detail / appendix / links (for the curious)
```

Cut from the bottom. If the message gets shortened, the top survives.

## Audiences

| Audience | What they care about | Tone | Detail | Length |
|---|---|---|---|---|
| Business or department owner (sales, finance, operations, marketing) | Does the data answer their question; can they trust it; when | Plain, outcome-first, no jargon | Low | Short |
| Product owner or team lead | Sprint progress, scope, risk, dependencies | Candid, specific | Mid | Medium |
| Engineering peer | Model design, grain, lineage, tradeoffs | Direct, technical | High | Long |
| Leadership (director, VP) | Business impact, risk, cost, timeline | Polished, confident, headlines | Low (detail linked) | Short |

Translate for the non-technical audiences: no `ref()`, no grain, SCD2, or incremental talk to a business owner. Say what the data will and will not tell them, and how confident you are.

## The research / context package branch (headline use case)

When the user picks output #4, the goal is to give a stakeholder **enough context to give useful input or make a decision**, not to dump the full research. Ask which shape fits the need; per the intake principle, do not assume.

- **A. Decision brief on top of research (default).** A short, stakeholder-facing brief (issue, what we found, options with tradeoffs, recommendation, what we need from you) that **links to the full research docs**. The research stays authoritative; the brief is the readable entry point. Best when full research already exists.
- **B. Stakeholder-ready research docs.** When the research docs themselves are the deliverable, **hand the doc-building to the `document` skill**, which owns structure and the data guide and analysis doc shapes. This skill contributes only the stakeholder-framing layer on top: what leads, how each open question is posed, and confirmed-vs-unconfirmed marking. Use when the audience is technical or semi-technical.
- **C. Brief plus delivery checklist.** The decision brief plus a packaging checklist: where the docs live, how to frame each open question so the stakeholder knows exactly what is being asked of them, and how to invite input (comment, meeting, async reply).

For all three, write in the **research-package / teaching register** (Register B in [references/voice-and-style.md](references/voice-and-style.md)): warmer and more first-person than a status update. Set shared understanding first, credit any prior work honestly, walk the logic end-to-end with concrete examples, define each segment with a plain meaning *and* a reliability judgment before showing counts, and close on directional recommendations plus an invitation to go deeper. Then:

- **Lead with the decision or question the stakeholder owns**, then the minimum context to make it. Do not make them read the whole investigation to find their question.
- **Every open question is addressed to a specific person** with what a "good answer" looks like. Many questions about business rules can only be answered by the business-process owner; name them and say so.
- **Preserve facts and figures verbatim** from the source research; reframe only wording and structure.
- **Distinguish confirmed from unconfirmed.** Mark what is measured vs. assumed vs. needs-owner-confirmation so the stakeholder weighs input on solid ground.

See the research / context package template in [references/stakeholder-templates.md](references/stakeholder-templates.md).

## Workflow

1. **Step 0 intake.** Output, audience, medium, timing, framing.
2. **Answer the five questions.** If you cannot write the headline, you do not yet know what you are communicating.
3. **Gather substance.** Pull from the user, existing docs, the ticket, or a verified query. Never fabricate.
4. **Draft in the matching template and in the right voice.** Explanatory prose in the author's own register, never a bullet-fact dump or a corporate recap. Voice flexes with output type: a tight, terse, evidence-first register for status updates and short informational notes (Register A); a warmer, first-person teaching register for research and context packages that set up and recommend (Register B). Read [references/voice-and-style.md](references/voice-and-style.md) first to calibrate. It has annotated worked examples of both registers: narrative over bullets, explain the mechanism, honesty about what is inferred, pronoun stance, and the moves specific to each register.
5. **Cut.** First drafts run 30 to 50% long. Cut activity-without-outcome, adjectives, filler (just, really, very), and hedging when you are actually confident.
6. **Verify the ask.** Clear, specific, owner named, deadline named.
7. **Jargon pass (non-technical audiences).** Before anything goes to a business owner or leadership, strip the engineering vocabulary: no `ref()`, grain, SCD2, incremental, lineage. Replace with what the data will and will not tell them and how confident you are. Stripping jargon is not enough on its own: where a mechanism is load-bearing to understanding (what a checkout-session id is, what the `Other` bucket is and where it lives), spend a sentence or two explaining it in plain terms rather than assuming it or dropping it.
8. **Check the audience.** Right register? Skipped needed context? Included context they do not need?
9. **Hand off.** Deliver the draft to the user for their chosen medium. Publish only if explicitly asked.

## Failure patterns

- **Bullet-fact dump with no narrative.** The most common miss. A wall of bullets or a bare table makes the reader reconstruct the meaning themselves. Lead each section with the *why* and reason through it in prose; reserve tables and bullets for genuinely dense data (a breakdown, a set of definitions), never as a substitute for explanation. See [references/voice-and-style.md](references/voice-and-style.md).
- **Long preamble before the point.** Lead with the takeaway.
- **Status yellow, three sprints running.** Unchanged yellow is red. Be honest about progress.
- **Update with no clear ask.** If nothing is needed, say "no action needed."
- **Sandwiching bad news between wins.** The bad news gets lost. If it is the headline, lead with it.
- **Activity vs. outcome.** "Built the staging model, wrote tests" invites "so what?". "The report will be trustworthy by Friday" is the outcome.
- **Technical register to a business owner.** Grain, lineage, and incremental logic mean nothing to a sales director. Translate.
- **Unrecommended decision request.** Options with no recommendation puts the work back on the stakeholder.
- **Open questions with no owner.** "Some questions remain." For whom? Name the person and what a good answer looks like.
- **Optimistic projections.** "Back on track next sprint," repeatedly. Trust erodes. Be honest about the path.
- **No comms when no news.** Silence reads worse than "no change this week." Set a cadence and hold it.
- **Publishing without being asked.** Draft; let the user decide when and where it ships.

## Hand off

- The research docs themselves are the deliverable (shape B): **document** builds them; this skill frames the open questions and the confirmed-vs-unconfirmed marking on top.
- A headline number in the draft has no second path behind it: **validate** before the note leaves the room.
- The update reports on a model's design findings: take them from **review**'s ranked list rather than re-deriving them.
- A discovered data-quality problem: if the team keeps a data-quality log or tracker, the bad-news note links to that entry rather than restating everything.

## Reference files

- [references/stakeholder-templates.md](references/stakeholder-templates.md): ready-to-use templates for each Step 0 output: status update, decision request, bad-news / data-quality note, research / context package (shapes A, B, C), leadership readout, and help / ask.
- [references/voice-and-style.md](references/voice-and-style.md): annotated worked examples of the target voice across two registers: a tight informational update (A) and a research / context package with a teaching voice (B). Read before drafting to calibrate.

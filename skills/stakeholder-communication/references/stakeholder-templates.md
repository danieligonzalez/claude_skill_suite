# Stakeholder communication templates

One template per Step 0 output. All are inverted-pyramid: headline first, ask never buried. Fill the brackets and delete the guidance lines. Keep numbers and facts grounded in a source; never invent.

---

## 1. Sprint / project status update

Audience: team lead, product owner, or business sponsor. Medium: ticket comment, wiki page, or email.

```
**Project:** [Name / epic]
**Status:** [On track / At risk / Off track]
**Headline:** [One sentence, the takeaway]

**This sprint:**
- [Outcome, not activity]
- [Outcome]

**Next sprint:**
- [Plan]

**Risks / blockers:**
- [Risk]: [what we are doing / what we need]

**Metrics (if any):**
- [Metric: value vs. target]
```

The status indicator is essential; stakeholders scan for it. Unchanged yellow is red.

---

## 2. Decision request

Audience: whoever holds the decision right. Medium: wiki page or email; a meeting only if it needs debate.

```
**Decision:** [What needs deciding, in one line]
**Need by:** [Date]   **Decides:** [Named person / role]

**Context:** [Minimum needed to understand; link the full research if it exists]

**Options:**
1. [Option A]: pros / cons / the business-process assumption it makes
2. [Option B]: pros / cons / assumption
3. [Option C]: pros / cons / assumption

**Recommendation:** [Option X, because ...]

**What we need from you:** [The specific input or approval]
```

Always include a recommendation. Tie each option to the assumption it makes; many business rules are inferred from the data rather than written down anywhere, and the stakeholder needs to see which inference each option rests on.

---

## 3. Bad-news / delay / data-quality note

The hardest to write well. Do not soften the headline. Soft headlines on bad news erode trust faster than the news itself.

```
**Headline:** [The bad news, plainly stated]

**What happened:** [Specific facts, no euphemisms; grounded, not guessed]

**Impact:** [Who is affected, which report or metric, how much, by when]

**What we are doing:** [Specific actions plus owners]

**What we need:** [Specific asks, if any]

**Next update:** [When the next update will land]
```

For a discovered data-quality issue, link the entry in the team's data-quality log or tracker (if one exists) rather than restating everything here.

---

## 4. Research / context package for stakeholder input

Goal: give the stakeholder enough to give useful input or make a call, not the full investigation. Pick shape A, B, or C per the Step 0 principle in SKILL.md.

### 4A. Decision brief on top of research (default)

```
**The question you own:** [The decision or input needed from this stakeholder]
**Need by:** [Date]

**What we found (short):**
- [Key finding, measured]
- [Key finding, measured]
- [Assumption still to confirm, flagged as unconfirmed]

**Options:**
1. [Option]: tradeoff / assumption
2. [Option]: tradeoff / assumption

**Our recommendation:** [X, because ...]

**What we need from you:**
- [Open question], best answered by [named owner]; a good answer looks like [example]

**Full research:** [Link to the research docs]
```

### 4B. Stakeholder-ready research docs

Hand the doc-building to the **`document`** skill. It owns the document shapes (the data guide for a pipeline, the analysis doc for a kept ad hoc analysis) and the rule that facts are preserved verbatim while only wording and structure are reworked. This skill contributes the stakeholder-framing layer on top:

- **What leads.** The decision or question the stakeholder owns goes first, not the investigation.
- **Open questions.** Each names its owner and a "good answer" example.
- **Confirmed vs. unconfirmed.** Marked throughout so input rests on solid ground.

### 4C. Brief plus delivery checklist

Deliver 4A, plus:

```
**Delivery checklist**
- [ ] Docs live at: [wiki page / docs folder / shared drive]
- [ ] Each open question names its owner and a "good answer" example
- [ ] Confirmed vs. unconfirmed clearly marked throughout
- [ ] Input channel stated: [comment / meeting / async reply] by [date]
- [ ] Right people notified in the right channel
```

---

## 5. Leadership / exec readout

Audience: director or VP. They skim, and click into detail when interested.

```
**Headline:** [One sentence, business impact]

**Key points:**
1. [Point plus one supporting sentence]
2. [Point plus one supporting sentence]
3. [Point plus one supporting sentence]

**Decisions needed:**
- [Decision: options plus recommendation]

**Asks:**
- [Specific request]

**Detail:** [Linked doc / appendix]
```

Low detail inline, full detail one click away. Confident tone, calibrated to real risk.

---

## 6. Ask for help / resources

```
**The ask:** [Specific request: a decision right, another team's input, access, time]
**Why it matters:** [What is blocked or at risk without it]
**By when:** [Date]
**Who I think can help:** [Named person / team]
```

Pull the ask to the top. Do not bury a request inside a wall of status.

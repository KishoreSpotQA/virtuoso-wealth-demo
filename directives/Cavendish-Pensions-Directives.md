# Cavendish Pensions — Touchstone Directives

> Directives to create in Touchstone, **not** a source to upload. Companion to `Cavendish-Pensions-KB-Source.md`.

**Applies to:** Cavendish Pensions — Retirement Income and Dashboards member matching, build 2026.09.19
**Source:** `Cavendish-Pensions-KB-Source.md`
**Application:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/platform.html`
**Set at:** project level, before the first generation

---

## How to read this set

Ten directives. Six are the same wording you already use for wealth and motor insurance and are marked **portable** — set them once per project and forget them. Four are **specific to pensions** and carry the domain into generation: two independent processes in one module, twenty-four decision codes across them under strict precedence, three separate human controls, and a settlement calculation with reachable and unreachable branches.

Directives are injected into **every** generation, so the set stays small deliberately. Set these before the first run; retro-fitting means regenerating.

### Where each one goes

| Target | Directives | Notes |
|---|---|---|
| **Requirement** | 1, 2, 4, 5, 6, 7, 10 | Templates exist here and nowhere else |
| **Journey** | 3, 8 | Instructions and conventions only |
| **Autopilot** | 9 | Instructions and conventions only |

---

## 1. Requirement · Instruction — `One branch per requirement`

**Pensions-specific. Rewritten for this module.**

```
Produce a separate requirement for each branch of a branching control rather than one requirement listing several outcomes. The branching controls are: the retirement option, the retirement decision code, the guidance and advice status, the vulnerability declaration, the reviewer's decision at authorisation and at review, the member matching outcome code, and the resolution decision on a possible match.
```

**Fixes:** the failure seen on the wealth project's first run, where six branching controls were collapsed into four requirements. This module is worse exposed: **twenty-four decision codes across two engines**, four retirement options, three guidance statuses, three graded matching outcomes and three independent human controls. A generation that writes "the instruction is accepted, referred or declined" as one requirement has covered nothing, and the coverage figure will say otherwise.

---

## 2. Requirement · Instruction — `Decision precedence`

**Pensions-specific. No equivalent in the other two projects.**

```
This module contains two decision engines, each evaluating its rules in a fixed order and returning the first that matches: the retirement decision with sixteen codes, and member matching with eight. Where a requirement covers one code, its test data must not satisfy any earlier code in that engine. Where two codes can be satisfied at once, produce a requirement asserting that the earlier code is the one reported.
```

**Fixes:** the single most likely source of wrong requirements here. Several conditions overlap and only the earlier one is ever observable:

- A member giving up a guaranteed annuity rate for an **unsustainable** drawdown income reports RP-R03, never RP-R07.
- A member in the Rest of world taking drawdown is declined under RP-D05; the overseas referral RP-R10 is unreachable on that path.
- A 51-year-old with a 620,000 fund declines on age, not on fund size.
- In member matching, **ambiguity is evaluated first**: two records agreeing equally return MM-N02, not a possible match, even though each agrees on two identifiers.
- MM-M02 requires the NI number to be **absent**, not merely wrong. A wrong NI number alongside an agreeing surname, date of birth and postcode returns MM-P02.

Without this directive, generation will write a requirement for RP-R07 with data that actually produces RP-R03, and the journey will fail for a reason nobody can find.

---

## 3. Journey · Instruction — `Four-eyes`

**Pensions-specific. Rewritten — this module has two controls, not one.**

```
This module has two independent four-eyes controls. An accepted instruction is released at payment authorisation; a referred instruction is settled at retirement review. A journey that reaches benefits in payment must sign out after submission and sign back in as a different user before deciding. A journey that ends at submission is a partial journey and must be named as one.
```

**Fixes:** a journey that never exercises the control the process exists for. Three things make this module different from the other two:

- **The accept path now has a control.** An accepted instruction does not go into payment on submission — it waits at *Awaiting authorisation* with no payment reference. The happy-path journey therefore runs through four-eyes rather than stopping short of it.
- **Signing out returns to the launcher.** A journey that signs out mid-run lands on the application picker and must activate `app-pensions` or navigate to a pensions route before continuing. A journey written against the older behaviour walks straight past this.
- **Member matching has a role gate but no identity check.** The person resolving a possible match is not the person who raised the request, because the request came from a dashboard rather than from staff. A journey asserting a self-resolution block there is asserting behaviour the application does not have.

For the retirement instruction controls the separation is enforced at two independent levels: only a Compliance user sees the actions at all, and a Compliance user still cannot decide a record they submitted.

---

## 4. Requirement · Instruction — `Two processes, one module`

**Pensions-specific. New.**

```
This module contains two independent processes: the retirement instruction workflow and dashboards member matching. A requirement must cover one process only, name it in its scope, and must not mix a retirement decision code with a matching outcome code.
```

**Fixes:** requirements that blur the two. They share a module, a sign-in and a member population but nothing else — different screens, different engines, different code prefixes, different controls. A requirement that cites both is testable against neither, and the coverage figure for each process stops meaning anything.

It also tells generation that member matching is a **read-only lookup**, not a wizard: there is no multi-step form and no draft, so a journey against it is a single request and an assertion on the outcome.

---

## 5. Requirement · Instruction — `Advisory versus blocking`

**Portable — same wording as the wealth and insurance projects.**

```
The source distinguishes controls that block progress from advisory warnings that do not. For an advisory control, acceptance criteria must assert both that the warning is displayed and that the user can still continue.
```

**Fixes:** requirements that assert the wrong behaviour. Two traps in this module specifically:

- The **tax-free cash maximum is not enforced by validation**. A request above 25 per cent is accepted by the form and declined at submission by RP-D04. A requirement asserting a field-level error is asserting something that does not exist.
- The **minimum pension age is not enforced at step 1** either. A 51-year-old passes validation and is declined at submission.

Both are decision-stage controls wearing the costume of field validation.

---

## 6. Requirement · Instruction — `Exact messages`

**Portable.**

```
Where the source records an exact error or banner message, quote that string verbatim in the acceptance criteria. Reference the field by its label and its data-testid.
```

**Fixes:** paraphrased assertions. The source carries every string the application renders, including the twenty-four reason texts across the two engines, and both four-eyes toasts — which differ between the two controls and are easy to conflate.

---

## 7. Requirement · Instruction — `Boundary cases`

**Portable.**

```
Where the source states a numeric, length or date threshold, produce coverage for the value that just fails and the value that just passes, using the verified boundary values given in the source.
```

**Fixes:** computed boundaries that are wrong. The age rules are the trap, as in the wealth project: age is elapsed milliseconds over a 365.25-day year, not a calendar birthday, so the minimum pension age cut-off lands a day early. A date of birth of `23-09-1971` evaluates to 54 and declines; `22-09-1971` evaluates to 55.

Several of the others are not where arithmetic would put them either — a fund of exactly 30,000 with safeguarded benefits and no advice **accepts**; 30,001 declines. The source states the verified values; this directive tells generation to read them rather than calculate.

---

## 8. Journey · Instruction — `Start and end state`

**Portable.**

```
Every journey opens by signing in and resetting the demo data, and closes with a verification checkpoint asserting the terminal state: the stage reached, the reference assigned, or the decision code reported.
```

**Fixes:** inherited state. Local storage under `cavendish.platform.v1` is **shared with the wealth and motor insurance modules** of this build, and one reset restores all three. Without a reset, a journey inherits whatever a previous run left behind — including an account left locked by a failed sign-in test in a different module.

---

## 9. Autopilot · Instruction — `Re-read the form after a revealing control`

**Pensions-specific wording; same idea as the other projects.**

```
Changing the vulnerability answer on Protection and safeguards, or the retirement option on Retirement option, adds or removes fields on the page. After changing either, re-read the form before interacting with further fields. On Income modelling, the settlement block updates in place without a page re-render; read the figures after the change rather than waiting for a reload.
```

**Fixes:** an agent typing into fields that are no longer there. Vulnerability = Yes reveals a mandatory textarea; each of the four retirement options replaces the panel below it.

The second sentence matters as much. Step 4 recalculates the settlement table **without re-rendering the step**, so an agent waiting for a page change waits forever, and one that re-reads too early gets the previous figures.

---

## 10. Requirement · Template

**Portable.**

```
Scope / Rationale / Pre-conditions / Test data / Acceptance criteria / Branches not covered / Source references
```

**Fixes:** requirements that look complete but say nothing about their limits. **Branches not covered** is the non-obvious section — with twenty-four codes under precedence across two engines, it is what turns the requirement set into a coverage argument rather than a list.

---

## Setup

1. Create the Requirement directives first, then Journey, then Autopilot.
2. Check for **name collisions** with organisation-level directives, and with the wealth and insurance projects if those are set at organisation level. A project directive of the same name **replaces** the organisation one; a differently-named one stacks alongside it. `One branch per requirement` and `Four-eyes` are deliberately reused names carrying different text — that is the point, and it only works if they are set at project level.
3. Run **Preview prompt** on each and read what is actually injected.
4. Confirm each is **saved and enabled**. A saved-but-disabled directive is the commonest cause of output that ignores the house style, and it looks identical to one that is working.
5. Only then generate.

## If you only set three

`One branch per requirement`, `Decision precedence`, `Two processes, one module`. Those three carry nearly all the value now that the module holds twenty-four codes across two engines. `Four-eyes` is the fourth, and the first one to add.

## Verifying they landed

Check these on the next generation. If one fails, the directive is not being applied — open it, confirm it is enabled, and run Preview prompt before regenerating.

| Directive | What to look for in the output |
|---|---|
| One branch per requirement | More than one requirement per branching control; the decision codes alone should produce well over twenty |
| Two processes, one module | No requirement cites both an RP code and an MM code |
| Decision precedence | At least one requirement asserts that an earlier code is reported where two conditions are met |
| Four-eyes | The end-to-end journey signs out and signs back in, **and** re-enters the module through the launcher |
| Advisory versus blocking | A requirement asserts that an over-maximum tax-free cash request is accepted by the form and declined at submission |
| Exact messages | Acceptance criteria contain quoted strings, and the two four-eyes toasts are not conflated |
| Boundary cases | The age pair reads `23-09-1971` / `22-09-1971`, and a 30,000 / 30,001 pair appears |
| Start and end state | Journeys open with a demo data reset |
| Re-read the form | Journeys re-read after changing the retirement option |
| Template | Every requirement carries a **Branches not covered** section |

---

## Two caveats worth keeping in mind

Directives are **guidance applied by an analysis step, not rules enforced by an engine**. They shape generation reliably in practice but are not guaranteed. Verify on the output that matters, which is what the table above is for.

Generation is also **non-deterministic**: the same source and prompt can produce different output on two runs. Treat any single run as a starting point to refine, and never promise a customer identical output twice.

# Marchmont Bank - Touchstone Directives

> Directives to create in Touchstone, **not** a source to upload. Companion to
> `Marchmont-Banking-KB-Source.md`.

**Applies to:** Marchmont Bank - Payments and Payment Operations, build 2026.10.26
**Source:** `Marchmont-Banking-KB-Source.md`
**Application:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/banking.html`
**Set at:** project level, before the first generation

---

## How to read this set

Thirteen directives. Six are the same wording used on the wealth, insurance,
pensions and Halcyon projects and are marked **portable**. Seven are **specific
to this application** and carry what makes it different from the other four:
a step-up that interrupts and resumes, a cut-off that depends on the time of
day, an irrevocability boundary, and two four-eyes controls that cannot both be
exercised by one journey.

Directives are injected into **every** generation, so the set stays small
deliberately. Set them before the first run; retro-fitting means regenerating.

### Where each one goes

| Target | Directives | Notes |
|---|---|---|
| **Requirement** | 1, 2, 3, 4, 7, 8, 9, 13 | Templates exist here and nowhere else |
| **Journey** | 5, 6, 10, 11 | Instructions and conventions only |
| **Autopilot** | 12 | Instructions and conventions only |

---

## 1. Requirement · Instruction — `One branch per requirement`

**Rewritten for this application.**

```
Produce a separate requirement for each branch of a branching control rather than one requirement listing several outcomes. The branching controls are: the channel decision code, the step-up trigger, the payment type, the rail, the hold code, the release or reject decision, the case ownership gate, the investigator identity check, and whether a payment can be cancelled.
```

**Fixes:** collapsed coverage. This application carries **seventeen decision
codes across two engines** - twelve PA codes in the channel and five PH codes in
operations - and they are mutually exclusive by precedence. A generation that
writes "the payment is accepted, held or rejected" as one requirement has
covered nothing, and the coverage figure will say otherwise.

---

## 2. Requirement · Instruction — `Name the system`

**Specific to this application.**

```
This application contains two separate systems: Marchmont Online and Sentinel Payment Operations. Every requirement must name the system it applies to in its scope, and must not mix a PA code with a PH code. Where a requirement covers the hand off between the two, say so explicitly and name both.
```

**Fixes:** requirements that blur the systems. They share a sign in and a
storage key but nothing else: different screens, different engines, different
code prefixes, different roles. A requirement citing both a PA code and a PH
code is testable against neither.

It also tells generation that the **hand off is itself a thing to be tested**,
not an implementation detail to be skipped over.

---

## 3. Requirement · Instruction — `Decision precedence`

**Specific to this application.**

```
Each engine evaluates its rules in a fixed order and returns the first that matches. Where a requirement covers one code, its test data must not satisfy any earlier code in that engine. Where two codes can be satisfied at once, produce a requirement asserting that the earlier code is the one reported.
```

**Fixes:** the single most likely source of wrong requirements here. Four
overlaps were confirmed against the running application, and in each only the
earlier code is ever observable:

- A 30,000.00 payment from an account holding 28,450.80 reports **PA-D01**,
  never PA-D05. A test for the daily limit written with too large a number
  tests the balance rule instead.
- A 5,000.00 payment to a new payee reports **PA-S01**, never PA-S02.
- A 1,500.00 payment to an established overseas payee reports **PA-S01**, never
  PA-S03. PA-S03 is only observable below 1,000.00.
- A 20,000.00 payment to a watchlisted payee reports **PH-H01**, never PH-H02.

Without this directive, generation will write a requirement for PA-S02 using a
5,000.00 amount, the test will report PA-S01, and the failure will look like a
product defect.

---

## 4. Requirement · Instruction — `The step-up hides the acceptance`

**Specific to this application. There is no equivalent in the other projects.**

```
A step-up code is not an outcome. When a payment triggers step-up, the review shows the step-up code, and once the passcode is accepted the engine runs again with the step-up rules skipped and the payment is recorded with whichever acceptance code then applies. A requirement covering a stepped-up payment must state the indicative code and the recorded code separately, and must not assert that they are equal.
```

**Fixes:** a requirement that asserts the wrong code on every payment above
1,000.00 - which is most of the interesting ones. A 12,000.00 payment shows
`PA-S01` on review and is recorded as `PA-A01` with a `PH-H02` hold. A
requirement asserting that `outcome-code` reads `PA-S01` fails against correct
behaviour.

It is also the clearest thing in this application that a DOM-driven tool cannot
infer: the relationship between the two codes exists only in the rules.

---

## 5. Journey · Instruction — `Cross system hand off`

**Specific to this application.**

```
The payment reference minted on submission is the key to the other system. A journey that reaches Sentinel must read that reference from Marchmont Online and use it to locate the payment or its case, rather than assuming a reference. A journey that ends in one system is a partial journey and must be named as one.
```

**Fixes:** two single system journeys pretending to be a chain. Concretely:
read the reference from the payment detail or `grid-payments`, store it, type it
into `filter-search` on the Sentinel enquiry screen, open `open-<reference>`,
and follow `open-case` to the case.

---

## 6. Journey · Instruction — `Two four-eyes controls, three journeys`

**Specific to this application. This is the one most likely to be got wrong.**

```
This application has two independent controls on a case decision. A case owned by Financial Crime cannot be decided by Payment Operations at all. Whoever saved the investigation note cannot decide that case, whatever their role. There is one account per operations role, so cover these as three journeys: one where Payment Operations is refused a financial crime case, one where the investigator is refused their own case, and one where a different user completes the release. A single journey cannot exercise the identity check and then complete the release.
```

**Fixes:** a journey that cannot complete, and a journey that never exercises
the control the process exists for. Two traps:

- **The controls render nothing.** Release and reject are absent from the page,
  with a notice in `decision-blocked` instead. A requirement asserting a
  disabled button is asserting something that does not exist.
- **There is a third refusal: the release limit.** Payment Operations may
  release up to 5,000.00 and no further. The limit sits below the anti money
  laundering threshold deliberately, so that it is reachable at all - above it,
  `PH-H02` would route every case to Financial Crime first and the limit could
  never fire. Cover 5,000.00 and 5,000.01 as a boundary pair.
- **Saving the note is what records the investigator.** A journey that saves a
  note as the user it intends to release with has just disqualified that user.
- **A decision cannot absorb unsaved text.** Activating Release or Reject with a
  note still in the box returns `Save the investigation note before recording a
  decision.` A journey must activate `btn-save-note` first.

---

## 7. Requirement · Instruction — `Advisory versus blocking versus absent`

**Portable, extended for this application.**

```
The source distinguishes controls that block progress, advisory indications that do not, and controls that are withheld entirely. Where a control is withheld, assert that it is absent rather than disabled. Where an indication is advisory, assert both that it is shown and that the user can continue.
```

**Fixes:** requirements that assert the wrong shape of behaviour. Three traps:

- **The review outcome is advisory.** It says so on the page, and the decision
  is confirmed on submission. A requirement that treats the review code as the
  recorded code is wrong - see directive 4.
- **The cancel control is absent, not disabled**, on a settled, released or
  rejected payment. A notice appears in `not-cancellable` in its place.
- **The launcher card for an unreachable system is still rendered.** The role
  gate acts on navigation and returns the user to the launcher with a toast, so
  a requirement must assert the resulting route or the toast, not the absence
  of the card.

---

## 8. Requirement · Instruction — `Unreachable rules`

**Portable in form, specific in content.**

```
PA-D02, the Faster Payment maximum of 1,000,000.00, cannot be reached through the user interface: no seeded account holds more than 389,500.00 and the highest daily limit is 100,000.00, so PA-D01 or PA-D05 always matches first. Do not produce a requirement that submits a payment above 1,000,000.00. Record PA-D02 under branches not covered with the reason.
```

**Fixes:** a requirement that cannot be executed, and a coverage figure that
quietly counts it anyway. Sixteen of the seventeen codes are reachable; stating
which one is not, and why, is what makes the count honest.

---

## 9. Requirement · Instruction — `Exact messages`

**Portable.**

```
Where the source records an exact error, banner or reason text, quote that string verbatim in the acceptance criteria. Reference the field by its label and its data-testid.
```

**Fixes:** paraphrased assertions. The source carries every string the
application renders, including all seventeen reason texts. Several are easy to
conflate: the four separate account number messages on the payee form, and the
two different refusals in `decision-blocked`.

---

## 10. Journey · Instruction — `Start and end state`

**Portable, with two settings specific to this application.**

```
Every journey opens by navigating with ?session=out so it starts signed out, pinning the release with ?release=off or ?release=on, pinning the clock with ?now= where the cut-off matters, then resetting the demo data from the sign in screen, then signing in. It closes with a verification checkpoint asserting the terminal state: the reference assigned, the decision code reported, or the payment status reached.
```

**Fixes:** inherited state, and two settings a journey cannot set any other way.

- **The daily limit is cumulative across the day.** Without a reset, a journey
  inherits the previous run's spending and the 25,000.01 boundary moves. This
  is the application where skipping the reset does the most damage.
- **All three URL setters are applied at page load.** A session survives a
  reload, so without `?session=out` a journey that navigates and then signs in
  finds itself already signed in and the sign in form absent.
- `login-reset` on the sign in screen is a **single** control with no
  confirmation, and it clears lockouts as well as data.

---

## 11. Journey · Instruction — `Read the passcode, never hard code it`

**Specific to this application.**

```
Where a payment triggers step-up, read the passcode from the passcode-value element on the page and type it into input-passcode. The passcode is derived from the amount, so it differs per payment and a hard coded value will fail. Three incorrect attempts abandon the payment and record nothing.
```

**Fixes:** a journey that works once against the amount it was authored with and
fails on every other. It also tells generation that the step-up screen is part
of the ordinary path rather than an error state, which is how it reads from the
decision table alone.

---

## 12. Autopilot · Instruction — `Revealing controls, exact options, no waiting`

**Specific to this application.**

```
Changing the payee mode on step 1, the country on a new payee, or the payment type on step 2 adds or removes fields on the page. After changing any of them, re-read the form before interacting with further fields. Select dropdown options by their complete visible text: the country list contains three entries beginning United. Type dates as dd-mm-yyyy into the date text inputs. Each one has a calendar button beside it; do not drive it, because the typed value reaches the same result by a route that is reliable. Nothing here is asynchronous, so do not insert a fixed pause. Before entering data on a step, confirm the wizard shows the expected step number.
```

**Fixes:** four separate ways a generated journey stalls or writes to the wrong
place.

- Choosing **Someone new** replaces the payee select with four fields; choosing
  **Standing order** hides the rail and reveals frequency and end date.
- Changing the **country** switches which fields are mandatory and which pattern
  the account number is checked against.
- A partial match on `United` is a three way coin flip, and the wrong one
  changes the outcome - Korea, Iran and Syria are declined outright under
  PA-D04 while the United Kingdom is not.
- A validation failure keeps the user on the current step, so a journey that
  does not check the step number will enter step 2 values into step 1 fields and
  fail somewhere unrelated.
- The calendar button writes `dd-mm-yyyy` back into the text input, so the two
  routes are interchangeable in their result and only one of them is reliable
  to generate.

---

## 13. Requirement · Template

**Portable, with one addition.**

```
Scope / System / Rationale / Pre-conditions / Test data / Acceptance criteria / Branches not covered / Source references
```

**Fixes:** requirements that look complete but say nothing about their limits.
**System** is added because the same verb means different things in the two.
**Branches not covered** is the non-obvious one: with seventeen codes under
precedence across two engines and one of them unreachable, it is what turns the
requirement set into a coverage argument rather than a list.

---

## Setup

1. Create the Requirement directives first, then Journey, then Autopilot.
2. Check for **name collisions** with organisation level directives and with the
   Cavendish and Halcyon projects. A project directive of the same name
   **replaces** the organisation one; a differently named one stacks alongside
   it. `One branch per requirement`, `Decision precedence`, `Name the system`,
   `Exact messages`, `Start and end state` and `Unreachable rules` are
   deliberately reused names carrying different text. That only works if they
   are set at project level.
3. Run **Preview prompt** on each and read what is actually injected.
4. Confirm each is **saved and enabled**. A saved but disabled directive is the
   commonest cause of output that ignores the house style, and it looks
   identical to one that is working.
5. Only then generate.

## If you only set three

`One branch per requirement`, `Decision precedence`, `The step-up hides the
acceptance`. Those three carry nearly all the value. `Two four-eyes controls,
three journeys` is the fourth, and the first one to add.

## Verifying they landed

| Directive | What to look for in the output |
|---|---|
| One branch per requirement | More than one requirement per branching control; the decision codes alone should produce at least sixteen |
| Name the system | Every requirement names one of the two systems, and none cites both a PA and a PH code |
| Decision precedence | At least one requirement asserts that 30,000.00 reports PA-D01 rather than PA-D05 |
| The step-up hides the acceptance | A requirement states the indicative code and the recorded code separately for a payment above 1,000.00 |
| Cross system hand off | The end to end journey reads the payment reference in Marchmont Online and searches for it in Sentinel |
| Two four-eyes controls, three journeys | Three journeys exist, and none tries to investigate and release as the same user |
| Advisory versus blocking versus absent | A requirement asserts the cancel control is **absent** on a settled payment |
| Unreachable rules | PA-D02 appears under branches not covered with a stated reason, not as a submitted payment |
| Exact messages | Acceptance criteria contain quoted strings, and the four account number messages are not conflated |
| Start and end state | Journeys navigate with `?session=out`, pin the release, reset, then sign in |
| Read the passcode | No journey contains a literal six digit passcode |
| Revealing controls | Journeys re-read the form after changing the payment type |
| Template | Every requirement carries a **System** line and a **Branches not covered** section |

---

## Two caveats worth keeping in mind

Directives are **guidance applied by an analysis step, not rules enforced by an
engine**. They shape generation reliably in practice but are not guaranteed.
Verify on the output that matters, which is what the table above is for.

Generation is also **non-deterministic**: the same source and prompt can produce
different output on two runs. Treat any single run as a starting point to
refine, and never promise a customer identical output twice.

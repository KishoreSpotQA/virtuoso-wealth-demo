# Cavendish Onboarding — Touchstone Directives

> Companion to `Cavendish-New-Client-Onboarding-KB-Source.md`. These are directives to create in Touchstone, **not** a source to upload.

**Applies to:** Cavendish Client Management & Onboarding (CMO), build 2026.09.2
**Application:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/platform.html`
**Set at:** project level
**Set before:** the first generation — retro-fitting means regenerating

---

## How directives work

Grounded in the Touchstone product documentation:

- **Three types.** *Instructions* shape the logic of what is generated. *Conventions* enforce naming and structural rules. *Templates* define the output format of generated requirements.
- **Three targets.** *Requirement*, *Journey* and *Autopilot*. Each reaches only its own stage. **Templates apply to requirements only** — journey and Autopilot directives offer instructions and conventions.
- **Name-based override.** A project directive **replaces** an organisation directive of the *same name*. Differently-named directives stack. Priority orders same-type directives but does **not** settle overlaps — the name does.
- **Preview prompt** shows exactly what will be injected. Run it on each directive once.
- **Guidance, not enforcement.** Directives are applied by an analysis step, not a rule engine. Expect them to be followed; verify on output that matters.
- **Content is treated as data, not as a prompt.** A legitimate rule phrased as an instruction to disregard something may be reworded on save. Every directive below is phrased positively for that reason.

Directives are injected into **every** generation, so keep each one short. The set below is deliberately minimal.

---

## Requirement directives

### Convention — `Stage prefix`

```
Prefix every requirement title with the stage it belongs to, in square brackets, using the stage names as written: [Sign in], [Prospect details], [KYC & AML], [Risk & suitability], [Documents], [Accounts & mandate], [Review & submit]. Use [Compliance approval] for anything after submission.
```

*Why:* the source is organised by stage. A titled set sorts into the same shape as the process, which makes coverage gaps visible at a glance.

### Instruction — `Exact messages`

```
Where the source records an exact error or banner message, quote that string verbatim in the acceptance criteria. Reference the field by its label and its data-testid.
```

*Why:* the source carries every error string the application renders. A requirement that paraphrases them produces an assertion that passes against the wrong text.

### Instruction — `One branch per requirement`

```
Produce a separate requirement for each branch of a branching control rather than one requirement listing several outcomes. The branching controls are: client classification, screening outcome, risk band, suitability cross-check, portfolio alignment, and the approve/reject decision.
```

*Why:* six branching controls with 4, 4, 4, 3, 2 and 2 outcomes respectively. Collapsed into single requirements, the coverage number stops meaning anything.

### Instruction — `Boundary cases`

```
Where the source states a numeric, length or date threshold, produce coverage for the value that just fails and the value that just passes, using the verified boundary values given in the source.
```

*Why:* the source gives verified boundaries for age, document expiry, minimum mandate, adjudication rationale length, rejection reason length and telephone digits. Two of those do not land where hand arithmetic would put them.

### Instruction — `Advisory versus blocking`

```
The source distinguishes controls that block progress from advisory warnings that do not. For an advisory control, acceptance criteria must assert both that the warning is displayed and that the user can still continue.
```

*Why:* the highest-value directive in this set. The suitability mismatch and portfolio-alignment banners both state that a documented rationale or exception will be required, and neither blocks. A requirement that asserts blocking is wrong, and it is the natural assumption from the banner text.

### Instruction — `Documented behaviour only`

```
Base every requirement on behaviour the source describes. Where the source records a gap between stated intent and actual behaviour, write the requirement against the actual behaviour and note the discrepancy.
```

*Why:* section 12 of the source records ten gaps, including an appropriateness warning that is promised in a field hint and never implemented. Without this directive, generation is likely to write requirements for the promised behaviour.

### Template

```
Scope / Rationale / Pre-conditions / Test data / Acceptance criteria / Branches not covered / Source references
```

*Why:* **Branches not covered** is the non-obvious section. It makes each requirement state what it deliberately leaves out, which is what turns a requirement set into a coverage argument.

---

## Journey directives

### Convention — `Journey naming`

```
Journey names follow <Stage> - <Scenario> - <Expected outcome>, e.g. KYC & AML - Watchlist surname without adjudication - Continue blocked.
```

### Instruction — `Single branch`

```
Each journey covers one branch of one decision. A journey must not assert two outcomes of the same control.
```

### Instruction — `Start and end state`

```
Every journey opens by signing in and resetting the demo data, and closes with a verification checkpoint asserting the terminal state: the stage reached, the message displayed, or the record status.
```

*Why:* state is held in browser local storage under `cavendish.platform.v1`. Without a reset, a journey inherits whatever the previous run left behind — including an account left locked by a failed sign-in test.

### Instruction — `Sign-in as a library checkpoint`

```
Express the sign-in sequence as a reusable checkpoint rather than repeating its steps in every journey. Journeys that test sign-in behaviour itself are the exception and must author the steps inline.
```

*Why:* every journey begins with it. One checkpoint to maintain rather than dozens of copies.

### Instruction — `Four-eyes`

```
An end-to-end journey that reaches an activated mandate must sign out after submission and sign back in as a different user before approving. A journey that ends at submission is a partial journey and must be named as one.
```

*Why:* the submitting user cannot approve their own record, and the application has no role selector — identity comes from signing in. An end-to-end journey that stops at submission never exercises the control the process exists for.

---

## Autopilot directives

### Convention — `Element identification`

```
Identify elements by their data-testid attribute in preference to label text, placeholder or position. The test id index is in the source.
```

*Why:* the application carries full `data-testid` coverage on every interactive element, and the stage 1 field set changes with client classification. Label-based identification breaks on the Trust and Corporate branches; test ids do not.

### Instruction — `Recover from validation errors`

```
After activating Continue, if the step counter has not advanced, read the validation messages shown on the page, set the fields they name, and activate Continue again. Do not repeat the same Continue without changing a field.
```

*Why:* the highest-value Autopilot directive for this application. Every mandatory field is reported at once on submit, each message naming its field. Without this, one missed field becomes a retry loop; with it, the page itself tells Autopilot what to fix.

### Instruction — `Exact option text`

```
Select dropdown options by their full exact visible text, never a partial match. The country list contains both "United Kingdom" and "United Arab Emirates", and the placeholder "Select…" is a selectable but invalid choice.
```

*Why:* a partial match on "United" is a coin flip between two countries — and the wrong one can change the screening outcome. The placeholder is a valid-looking selection that fails validation.

### Instruction — `Re-read the form after a revealing control`

```
Changing client classification, the PEP declaration, or running screening adds or removes fields on the page. After changing any of these, re-read the form before interacting with further fields.
```

*Why:* Trust and Corporate replace the whole name block and relabel Nationality; PEP = Yes reveals a mandatory textarea; a Potential match reveals a checkbox and a rationale field. An agent working from a stale read types into fields that are no longer there.

### Instruction — `Complete the questionnaire before continuing`

```
On Risk & suitability, answer all eight questions, then confirm the score and band elements show a value before activating Continue.
```

*Why:* thirty-two radio options is where an agent most easily drops one. The live score and band give a free readiness check — if the band is still showing "n of 8 answered", the step is not ready.

### Instruction — `Attach documents with the sample control`

```
On the Documents step, attach each required document using its "Attach sample" control. The file input beside it is hidden and is not the intended path.
```

*Why:* file upload is a documented Autopilot weak spot, and this application provides a purpose-built alternative. The real file input is `display:none` and driving it will fail.

### Instruction — `Radio and checkbox verification`

```
After setting a radio or checkbox, verify it is selected before moving on.
```

*Why:* the adviser declaration gates submission and the screening-override checkbox gates progress past KYC. Both are single controls whose silent failure surfaces much later.

### Instruction — `Step verification`

```
After activating Continue, verify the wizard heading and step counter show the expected stage before interacting with any field on it.
```

*Why:* a validation failure keeps the user on the current stage. Without this check a journey silently enters stage 3 values into stage 2 fields and fails somewhere unrelated.

### Instruction — `Screening wait`

```
Screening is asynchronous. Wait for the screening outcome banner to appear rather than pausing for a fixed duration.
```

*Why:* generated wait durations track the authoring session rather than the application, which is a documented weak spot. An explicit wait on the outcome element removes the guesswork.

### Instruction — `Dates`

```
Enter dates as literal values supplied by the test data. Do not derive a date from the current date, because the application evaluates all date rules against a fixed internal clock.
```

*Why:* the application's clock is fixed at 2026-09-22. A relative date drifts against a fixed clock and the journey rots. Date pickers are also a documented weak control type, so expect to check these steps.

### Instruction — `Environment values`

```
Take the base URL, username and password from environment variables rather than hardcoded literals. Do not read credentials from the demo credentials panel on the sign-in screen.
```

*Why:* the sign-in screen displays the demo credentials, which makes hardcoding them tempting. A credential change should be a configuration change, not an edit to every journey.

---

## Setup order

1. Create the Requirement directives first, then Journey, then Autopilot.
2. Check for **name collisions** with any organisation-level directives. A project directive of the same name replaces the org one; a differently-named one stacks alongside it. Neither is wrong — but know which you are doing.
3. Run **Preview prompt** on each and read what is actually injected.
4. Confirm each is **saved and enabled**. A saved but disabled directive is the commonest cause of output that ignores the house style.
5. Only then generate.

## If you only set three

`Element identification`, `Advisory versus blocking`, `Exact messages`. Those three carry most of the value for this application.

## Verifying they landed

| Directive | Check on the generated output |
|---|---|
| Stage prefix | Every requirement title opens with a bracketed stage name |
| Exact messages | Acceptance criteria contain quoted strings, not paraphrases |
| One branch per requirement | More than one requirement exists per branching control |
| Boundary cases | Just-fails and just-passes values both appear |
| Advisory versus blocking | At least one requirement asserts a warning displays *and* progress continues |
| Documented behaviour only | No requirement asserts an execution-only appropriateness warning |
| Element identification | Journey steps reference test ids rather than label text |
| Four-eyes | The end-to-end journey signs out and signs back in before approving |
| Recover from validation errors | A journey that hits a validation failure fills the named field rather than retrying Continue |
| Exact option text | Country selections use full names, not partial matches |
| Attach documents with the sample control | Document steps use "Attach sample", not the hidden file input |
| Sign-in as a library checkpoint | Sign-in appears as a checkpoint, not as repeated inline steps |

If a check fails, the directive is not landing. Open it, confirm it is enabled, and run Preview prompt before regenerating.

---

## Note on the sign-in stage

Sign-in was added to the application in build 2026.09.2 and is documented in section 2 of the KB source. It changes two things for generation:

1. **The four-eyes control now runs through authentication.** Previously identity came from a dropdown; it now comes from signing in. The journey directive above reflects that, and a journey written against the old dropdown will not work.
2. **Credentials give the environment-variable directive something to bite on.** Before sign-in existed, the base URL was the only environment value, so the directive was inert.

Three sign-in behaviours are worth checking appear in the generated requirements, because each is easy to get wrong:

- The failure message is **identical** for a wrong password and an unknown username. A requirement asserting a distinct "user not found" message is asserting behaviour the application does not have.
- **Blank-field submission does not count towards the lockout.** Field validation runs before any credential check.
- **No warning precedes lockout**, and the lockout **has no time-based expiry** — it clears only on a demo data reset.

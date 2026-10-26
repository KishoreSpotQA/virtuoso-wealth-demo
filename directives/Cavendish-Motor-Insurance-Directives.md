# Cavendish Motor Insurance — Touchstone Directives

> Directives to create in Touchstone, **not** a source to upload. Companion to `Cavendish-Motor-Insurance-KB-Source.md`.

**Applies to:** Cavendish Motor Insurance — Quote & Apply, build 2026.09.27
**Source:** `Cavendish-Motor-Insurance-KB-Source.md`
**Application:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/platform.html`
**Set at:** project level, before the first generation

---

## How to read this set

Eleven directives. Five are the same wording used on the wealth, pensions and Halcyon projects and are marked **portable**. Six are **specific to motor insurance** and carry what makes this module different: fourteen underwriting rules under strict precedence, four fields that are collected and never used, a premium floor that silently swallows discounts, and a four-eyes control that cannot be exercised on any seeded record.

Directives are injected into **every** generation, so the set stays small deliberately. Set them before the first run; retro-fitting means regenerating.

### Where each one goes

| Target | Directives | Notes |
|---|---|---|
| **Requirement** | 1, 2, 3, 4, 5, 6, 7, 11 | Templates exist here and nowhere else |
| **Journey** | 8, 9 | Instructions and conventions only |
| **Autopilot** | 10 | Instructions and conventions only |

---

## 1. Requirement · Instruction — `One branch per requirement`

**Rewritten for this module.**

```
Produce a separate requirement for each branch of a branching control rather than one requirement listing several outcomes. The branching controls are: the underwriting decision code, the cover type, the retirement of a referral at accept or decline, the role gate on the referral queue, and the identity check on a referral the acting user submitted.
```

**Fixes:** collapsed coverage. This module carries **fourteen underwriting codes** — five declines, eight referrals and one accept — and they are mutually exclusive by precedence. A generation that writes "the application is accepted, referred or declined" as one requirement has covered nothing, and the coverage figure will say otherwise.

---

## 2. Requirement · Instruction — `Decision precedence`

**Specific to this module.**

```
The underwriting engine evaluates fourteen rules in a fixed order and returns the first that matches: five declines, then eight referrals, then accept. Where a requirement covers one code, its test data must not satisfy any earlier code. Where two codes can be satisfied at once, produce a requirement asserting that the earlier code is the one reported.
```

**Fixes:** the single most likely source of wrong requirements here. Several conditions overlap and only the earlier one is ever observable:

- **A DR10 conviction is both a decline (UW-D03) and a conviction referral (UW-R03).** A DR10 quote always declines. UW-R03 is only reachable with SP30, CU80 or IN10.
- **Three fault claims (UW-D02) outranks two (UW-R02).** A test that sets "three or more" and expects a referral is wrong.
- **UW-D04 and UW-R06 both key on group 15 or above.** A proposer with under a year's licence on a group-15 business-use vehicle declines; the referral is never seen.
- **Value above 100,000 with group 17 is a referral (UW-R04), not a decline** — UW-D05 needs group 18 as well.
- **A single quote can satisfy five referral rules at once** and will report only the earliest.

Without this directive, generation will write a requirement for UW-R03 using a DR10 conviction, the test will decline instead of refer, and the failure will look like a product defect.

---

## 3. Requirement · Instruction — `Fields that do not rate`

**Specific to this module. There is no equivalent in the other projects.**

```
Four fields in this module are collected but do not affect the premium. Estimated value and licence type drive underwriting outcomes only. Occupation and payment method are collected and never used at all. Non-fault claims rate the premium but never trigger a referral. Where a requirement covers one of these fields, its acceptance criteria must state what the field does affect and what it does not, and must not assert a premium change where none occurs.
```

**Fixes:** requirements that assert a price movement the engine never produces. This is also the strongest coverage argument in the module: two quotes differing only in estimated value — 18,500 against 74,000 — produce **the same total of 292.32** and **different outcomes**, one accepted and one referred. A requirement asserting that the outcome changed while the premium did not is a test no DOM-driven tool would ever think to write.

The same holds for licence type: `Provisional UK` refers under UW-R08 and leaves the premium at 292.32.

---

## 4. Requirement · Instruction — `Unreachable rules`

**Specific to this module. New.**

```
UW-D01, the decline for a proposer under 17, cannot be reached through the user interface: step 2 validation rejects an under-17 date of birth before submission is possible. Do not produce a requirement that submits an under-17 application. Cover the step 2 validation instead, and record UW-D01 under branches not covered with the reason.
```

**Fixes:** a requirement that cannot be executed, and a coverage figure that quietly counts it anyway. The rule exists in the decision engine and is first in precedence, so a generation reading the rule table will try to cover it. Thirteen of the fourteen codes are reachable; stating which one is not, and why, is what makes the count honest.

---

## 5. Requirement · Instruction — `Advisory versus blocking versus absent`

**Portable, extended for this module.**

```
The source distinguishes controls that block progress, advisory indications that do not, and controls that are withheld entirely. Where a control is withheld from a role, assert that it is absent rather than disabled. Where an indication is advisory, assert both that it is shown and that the user can continue.
```

**Fixes:** requirements that assert the wrong shape of behaviour. Three traps here:

- **The referral role gate renders nothing.** For an Adviser or Operations user the accept and decline actions are not in the page at all, and a notice appears instead. A requirement asserting a disabled button is asserting something that does not exist.
- **The release limit warning does disable a button.** Above the limit `btn-release` is genuinely disabled and a warning names the limit. That one *is* a disabled-state assertion, and the distinction from the role gate is deliberate.
- **The review step shows an indicative outcome before submission.** It is advisory; the decision is confirmed on submission.

---

## 6. Requirement · Instruction — `Exact messages`

**Portable.**

```
Where the source records an exact error, banner or reason text, quote that string verbatim in the acceptance criteria. Reference the field by its label and its data-testid.
```

**Fixes:** paraphrased assertions. The source carries every string the application renders, including all fourteen underwriting reason texts and both four-eyes toasts. Several are easy to conflate — the three licence-years messages, and the two separate refusals on the decline reason field.

---

## 7. Requirement · Instruction — `Boundary cases`

**Portable.**

```
Where the source states a numeric, length or date threshold, produce coverage for the value that just fails and the value that just passes, using the verified boundary values given in the source rather than calculating them.
```

**Fixes:** computed boundaries that are wrong. Three traps in this module:

- **Age is elapsed milliseconds over a 365.25-day year, not a calendar birthday**, so the age-17 cut-off lands a day early. A date of birth of `23-09-2009` evaluates to 16.9976 and is rejected; `22-09-2009` is accepted.
- **The licence-years cap is age minus 17**, so it moves with the date of birth. For a proposer born `11-02-1988` the maximum accepted is 21; 22 fails.
- **The premium floor silently swallows discounts.** Below a net of 180 the no-claims discount and the excess discount stop affecting the total. A sweep test on a cheap risk will see the same number repeatedly, and that is not a defect.

---

## 8. Journey · Instruction — `Four-eyes`

**Specific to this module. Two mechanisms, and one of them cannot be reached from seeded data.**

```
This module has two independent controls on a referral decision. Only a Compliance user sees the accept and decline actions at all. A Compliance user who submitted the application is then stopped by a separate identity check. Cover them as two journeys, not one: a journey that submits as the adviser and decides as the compliance officer exercises the role gate and reaches an issued policy, and a journey that submits as the compliance officer and attempts to decide their own referral exercises the identity check and ends blocked.
```

**Fixes:** a journey that never exercises the control the process exists for, and a journey that cannot complete. Two traps:

- **The identity check never fires on a seeded record.** Seeded quotes store the submitting user with an ordinary hyphen while the account name carries an em dash, and the check compares the two strings exactly. A requirement using a seeded referral to cover the identity check passes for the wrong reason.
- **There is one Compliance account.** So a single journey cannot both trigger the identity check and go on to complete the acceptance. Hence two journeys.

---

## 9. Journey · Instruction — `Start and end state`

**Portable, with a module-specific reset path.**

```
Every journey opens by signing in, opening the motor insurance application and resetting the demo data from inside it, and closes with a verification checkpoint asserting the terminal state: the stage reached, the reference assigned, or the decision code reported. The applications launcher carries no reset control, so the reset cannot be done before the module is open. The reset is two steps: open the confirmation, then confirm.
```

**Fixes:** inherited state and a reset that silently does nothing. Three things to know:

- Local storage under `cavendish.platform.v1` is **shared with the wealth and pensions modules**, but a reset from inside motor insurance restores **motor insurance only**.
- On the launcher the reset button is **hidden, not removed**. A journey asserting that the element does not exist will fail; assert that it is not visible.
- A single click on the reset control changes nothing — the reset happens on the confirmation.

---

## 10. Autopilot · Instruction — `Selects, dates and no waiting`

**Specific to this module.**

```
Select dropdown options by their complete visible string. Three option sets contain prefix collisions where one option's full text opens another's: cover type, and two pairs within use class. Voluntary excess displays as formatted currency and must be selected by the displayed text rather than the stored number. Type dates as dd-mm-yyyy rather than driving the calendar button. Nothing in this module is asynchronous, so do not insert a fixed pause: rating and underwriting are synchronous and no wait is required beyond the ordinary re-render. Before entering data on a step, confirm the wizard shows the expected step number.
```

**Fixes:** four separate ways a generated journey stalls or writes to the wrong place.

- A partial select match on `Third party, fire and theft` can land on `Third party only`. The same applies to the two social use classes and to business class 1 against class 2.
- The excess options render as `£250` and `£1,000` while the stored value is the bare number.
- A validation failure keeps the user on the current step, so a journey that does not check the step number will enter step 3 values into step 2 fields and fail somewhere unrelated.
- Generated fixed waits are pure cost here and make the journey look slower than the product is.

---

## 11. Requirement · Template

**Portable.**

```
Scope / Rationale / Pre-conditions / Test data / Acceptance criteria / Branches not covered / Source references
```

**Fixes:** requirements that look complete but say nothing about their limits. **Branches not covered** is the non-obvious section — with fourteen codes under precedence and one of them unreachable, it is what turns the requirement set into a coverage argument rather than a list.

---

## Setup

1. Create the Requirement directives first, then Journey, then Autopilot.
2. Check for **name collisions** with organisation-level directives and with the wealth, pensions and Halcyon projects. A project directive of the same name **replaces** the organisation one; a differently-named one stacks alongside it. `One branch per requirement`, `Decision precedence`, `Boundary cases`, `Four-eyes` and `Start and end state` are deliberately reused names carrying different text. That only works if they are set at project level.
3. Run **Preview prompt** on each and read what is actually injected.
4. Confirm each is **saved and enabled**. A saved-but-disabled directive is the commonest cause of output that ignores the house style, and it looks identical to one that is working.
5. Only then generate.

## If you only set three

`One branch per requirement`, `Decision precedence`, `Fields that do not rate`. Those three carry nearly all the value. `Four-eyes` is the fourth, and the first one to add.

## Verifying they landed

| Directive | What to look for in the output |
|---|---|
| One branch per requirement | More than one requirement per branching control; the decision codes alone should produce at least thirteen |
| Decision precedence | At least one requirement asserts that DR10 declines rather than refers |
| Fields that do not rate | A requirement asserts that changing the estimated value changes the outcome and not the premium |
| Unreachable rules | UW-D01 appears under branches not covered with a stated reason, not as a submitted application |
| Advisory versus blocking versus absent | A requirement asserts the referral actions are **absent** for an adviser, and separately that the release button is **disabled** above the limit |
| Exact messages | Acceptance criteria contain quoted strings, and the two four-eyes toasts are not conflated |
| Boundary cases | The age pair reads `23-09-2009` / `22-09-2009`, and the premium floor appears |
| Four-eyes | Two journeys exist, not one, and neither uses a seeded referral for the identity check |
| Start and end state | Journeys open the module before resetting, and the reset is two steps |
| Selects, dates and no waiting | No journey contains a fixed pause, and no select uses a partial option string |
| Template | Every requirement carries a **Branches not covered** section |

---

## Two caveats worth keeping in mind

Directives are **guidance applied by an analysis step, not rules enforced by an engine**. They shape generation reliably in practice but are not guaranteed. Verify on the output that matters, which is what the table above is for.

Generation is also **non-deterministic**: the same source and prompt can produce different output on two runs. Treat any single run as a starting point to refine, and never promise a customer identical output twice.

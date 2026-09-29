# Cavendish Motor Insurance — Quote & Apply

**System:** Cavendish Client Ops — Motor Insurance module
**Build:** 2026.09.29
**Release:** previous (New features off)
**Module:** Motor insurance → Quotes → New quote
**Application URL:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/platform.html`
**Document type:** End-to-end process specification
**Status:** Current — describes the application as built

> **Scope of this document.** This describes the **motor insurance module only** — sign-in, the application launcher, the five-step quote and apply workflow, the rating engine, the underwriting decision, and the underwriting referral queue.
>
> **This is a different URL from the wealth application.** The wealth app is deployed on its own at the site root and documented in `WAM-Cavendish-Onboarding-KB-Source.md`. The `/platform.html` build described here carries **three** modules behind a launcher — wealth, motor insurance and pensions — and keeps its own separate state, so records created here are invisible to the root deployment and vice versa. Only the insurance module is in scope here; the wealth module is a copy of the standalone app, and the pensions module is covered by `Cavendish-Pensions-KB-Source.md`.
>
> The launcher can be bypassed: a direct hash route such as `#/insurance/quote/new` resolves straight to the quote wizard after sign-in.

---

## 0. Which release this describes

The application carries a **New features** control in the sidebar, `data-testid="release-toggle"`, which switches Motor Insurance to a later release. **This document describes the application with that control off**, which is its default state.

With the release on, the application adds fields to the quote, adds a referral rule and changes how the premium is calculated. None of that is in scope here; it is covered by `Cavendish-Motor-Insurance-KB-Source-v2.md`.

**A journey written against this document must pin the release off** rather than relying on the default, because a browser reused between runs can still carry the release from an earlier one:

```
platform.html?release=off#/insurance/quote/new
```

The setter is absolute and idempotent. Do **not** click `release-toggle` from a journey: it is a toggle, so it is only correct from a known starting state.

> **The URL setter is estate-wide.** `?release=on` turns the release on for **every** application in this shell that has one, not only the one you navigate to. The sidebar control is per application; the URL setter is not. A journey that needs one application on its new release and another on its previous one cannot express that with this flag, and should use the sidebar control manually before the run.

While the release is off, no new field is present in the DOM, the rating engine has nine factors, and the last referral rule is UW-R08.

**Download KB source** in the sidebar serves this document while the release is off. With the release on the control reads **Download KB source v2** and serves the other document instead, so the application never hands out a source describing behaviour it does not have.

> **Existing records are unaffected either way.** Each quote is stamped with the release it was written under and is rated on those rules permanently, so turning the release on does not reprice or re-render any record described in this document. The seven seeded quotes keep nine rating factors whatever the toggle does.

---

## 1. What this process does

Cavendish Motor Insurance takes a proposer from a blank quote to one of three terminal outcomes: a **policy issued**, an **application declined**, or a **referral to an underwriter**. The process is a **five-step wizard** completed by a Relationship Manager. On submission the application is rated and underwritten in one action, and the outcome is decided by rule, not by a human.

Where the outcome is a referral, a **second user** — never the submitting one — accepts or declines it. That is the four-eyes control in this module.

### Actors

The same three accounts serve all three modules.

| Actor | Name in the system | Username | Role | What they can do here |
|---|---|---|---|---|
| Adviser | Lena Fairbrook — Relationship Manager | `lena.fairbrook` | Adviser | Build a quote, submit an application |
| Underwriter | Desmond Achebe — Compliance Officer | `desmond.achebe` | Compliance | Accept or decline a referral, apply a loading |
| Operations | Yuki Mori — Client Operations | `yuki.mori` | Operations | View records |

Password for all three: `Cavendish#2026`.

**The underwriting role is the `Compliance` role.** The application has no separate underwriter role — the same role that approves a wealth mandate decides an insurance referral. A requirement that assumes a distinct underwriter account is asserting behaviour the application does not have.

The acting user is established by **signing in**. There is no role selector.

### Navigation

| Route | Screen | Notes |
|---|---|---|
| `#/apps` | Application launcher | `app-insurance` opens this module |
| `#/insurance` | Insurance dashboard | Open quotes, referrals, declined, gross written premium |
| `#/insurance/quotes` | Quotes | List of quotes and applications |
| `#/insurance/quote/new` | **New quote wizard** | The process described here |
| `#/insurance/quote/{id}` | Quote detail | Risk, proposer, premium, decision, audit; referral actions |
| `#/insurance/referrals` | Underwriting referrals | Applications awaiting an underwriter decision |
| `#/insurance/policies` | Policies | Issued policies |
| `#/dashboard…`, `#/pensions…` | The wealth and pensions modules | Documented separately; out of scope here |

### Record lifecycle

```
Quote ──submit──> ACCEPT  ──> Policy issued
                  DECLINE ──> Declined
                  REFER   ──> Referred ──accept──> Policy issued
                                        ──decline─> Declined
```

The stage `Application` exists in the data model and appears in the stage pill map, but a wizard submission moves a record directly from **Quote** to one of the three terminal stages. There is no intermediate state a journey can reach through the UI.

A quote reference is assigned on submission in the form `MQ-{n}`. A policy reference `POL-{n}` is assigned only when a policy is issued.

---

## 2. Sign in and the launcher

Identical to the wealth module. The sign-in screen is described in full in the wealth source; the behaviour that matters here:

- The failure message is **identical** for a wrong password and an unknown username.
- **Blank-field submission does not count towards the lockout** — field validation runs before any credential check.
- The account locks on the **fifth** failed attempt with no prior warning, and the lockout **has no time-based expiry** — it clears only on a demo data reset.

After sign-in the **application launcher** is shown.

| Element | `data-testid` | Opens |
|---|---|---|
| Launcher container | `app-picker` | — |
| Wealth & Asset Management | `app-wam` | `#/dashboard` |
| Motor Insurance | `app-insurance` | `#/insurance` |
| Pensions | `app-pensions` | `#/pensions` |
| Switch application | `nav-switch-application` | Returns to the launcher |

**The platform is branded as a group.** The sign-in card and the launcher read *Cavendish · Financial Services · Client Ops*. Once inside a module the sidebar sub-line (`brand-sub`) names that business instead — *Motor Insurance · Client Ops* here, *Private Wealth* and *Retirement* in the other two. The browser title follows the same pattern.

**Signing out returns to the launcher.** A journey that signs out mid-run and signs back in lands on the application picker, not on the module it left, and must activate `app-insurance` or navigate to an insurance route. A journey that deep-links to a route *while signed out* still resolves to that route after signing in.

A journey may navigate directly to `<base>/platform.html#/insurance/quote/new` and skip the launcher entirely. Note the hash follows the filename — the route does not work against the site root, which serves the standalone wealth application.

---

## 3. Step 1 — Cover & vehicle

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Cover type | Select | Yes | Non-empty | `Select a cover type.` |
| Registration | Text | Yes | Non-empty | `Registration is required.` |
| Make | Text | Yes | Non-empty | `Make is required.` |
| Model | Text | Yes | Non-empty | `Model is required.` |
| Year of manufacture | Number | Yes | Non-empty; 1980–2026 | `Year of manufacture is required.` / `Enter a year between 1980 and 2026.` |
| Insurance group | Number | Yes | Non-empty; 1–50 | `Insurance group is required.` / `Insurance group must be between 1 and 50.` |
| Estimated value (GBP) | Number | Yes | Non-empty; greater than zero | `Estimated value is required.` / `Enter a value greater than zero.` |
| Use class | Select | Yes | Non-empty | `Select a use class.` |
| Overnight parking | Select | Yes | Non-empty | `Select where the vehicle is kept overnight.` |
| Annual mileage | Number | Yes | Non-empty; greater than zero | `Annual mileage is required.` / `Enter an annual mileage greater than zero.` |
| Modifications declared | Radio | Yes | Answered | `Answer the modifications question.` |

### Option lists

**Cover type**
Comprehensive · Third party, fire and theft · Third party only

**Use class**
Social, domestic and pleasure · Social, domestic, pleasure and commuting · Business use - class 1 · Business use - class 2

**Overnight parking**
Locked garage · Private driveway · Off-street parking · On street outside home · On street elsewhere

The modifications question carries the hint *"Declared modifications are referred to an underwriter."* That hint is accurate — see UW-R05.

---

## 4. Step 2 — Driver & history

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| First name | Text | Yes | Non-empty | `First name is required.` |
| Last name | Text | Yes | Non-empty | `Last name is required.` |
| Date of birth | **Text, `dd-mm-yyyy`, with calendar button** | Yes | Non-empty; age ≥ 17; age ≤ 100 | `Date of birth is required.` / `The proposer must be at least 17 years old.` / `Enter a valid date of birth.` |
| Postcode | Text | Yes | Non-empty | `Postcode is required.` |
| Licence type | Select | Yes | Non-empty | `Select a licence type.` |
| Years licence held | Number | Yes | Non-empty; ≥ 0; ≤ (age − 17) | `Years licence held is required.` / `Enter a positive number of years.` / `Licence years cannot exceed the time since the proposer turned 17.` |
| Occupation | Select | Yes | Non-empty | `Select an occupation.` |
| Fault claims | Number | Yes | Non-empty (zero is valid) | `Enter the number of fault claims, or zero.` |
| Non-fault claims | Number | Yes | Non-empty (zero is valid) | `Enter the number of non-fault claims, or zero.` |
| Motoring convictions | Select | Yes | Non-empty | `Select a conviction, or None.` |

### Option lists

**Licence type**
Full UK · Full EU · Provisional UK · International

**Occupation**
Accountant · Teacher · Nurse · Software engineer · Construction worker · Professional driver · Company director · Student · Retired · Unemployed

**Motoring convictions**
None · SP30 - exceeding statutory speed limit · CU80 - using a mobile phone · IN10 - driving uninsured · DR10 - driving with excess alcohol

> **Occupation is collected but never used.** It does not appear in the rating engine or the underwriting rules. A requirement asserting that occupation changes the premium is asserting behaviour the application does not have.

### The licence-years cap

Years licence held is capped at **age − 17**. For a proposer born `11-02-1988` — age 38 against the fixed clock — the maximum accepted value is **21**; 22 produces the cap error. The cap is evaluated against the proposer's date of birth, so it changes with it.

---

## 5. Step 3 — Quote

The premium is calculated and displayed live on this step.

| Field | Control | Mandatory | Options | Exact error message |
|---|---|---|---|---|
| No-claims discount (years) | Select | Yes | 0 – 9 | `Select the no-claims discount.` |
| Voluntary excess | Select | Yes | £0 / £250 / £500 / £750 / £1,000 | `Select a voluntary excess.` |
| Optional extras | Checkboxes | No | Three, below | — |

### Optional extras

| Extra | `data-testid` | Price |
|---|---|---|
| Breakdown cover | `extra-breakdown` | £42 |
| Motor legal protection | `extra-legal` | £28 |
| Guaranteed courtesy car | `extra-courtesy` | £35 |

Each renders with the label `{name} - {price}`, for example `Breakdown cover - £42`.

### The rating engine

```
gross     = base × (nine factors multiplied together)
net       = gross × (1 − NCD) × (1 − excess discount) + extras
            floored at 180
total     = net + loading + IPT,   IPT = 12% of (net + loading)
```

**Base premium by cover type**

| Cover | Base |
|---|---|
| Comprehensive | 480 |
| Third party, fire and theft | 390 |
| Third party only | 340 |

**The nine rating factors**

| # | Factor | Rule |
|---|---|---|
| 1 | Vehicle group | `1 + (group − 10) × 0.045`, rounded to 3 dp. Group 10 is neutral. |
| 2 | Driver age | under 21 → 2.30 · under 25 → 1.70 · under 30 → 1.25 · under 60 → 1.00 · under 70 → 1.10 · 70+ → 1.35 |
| 3 | Licence held | under 1 yr → 1.45 · under 3 → 1.20 · under 5 → 1.05 · 5+ → 1.00 |
| 4 | Claims history | `1 + (fault × 0.35) + (non-fault × 0.05)`, rounded to 3 dp |
| 5 | Motoring convictions | any conviction other than None → 1.25, else 1.00 |
| 6 | Annual mileage | ≤5,000 → 0.92 · ≤10,000 → 1.00 · ≤20,000 → 1.12 · ≤30,000 → 1.25 · above → 1.40 |
| 7 | Overnight parking | Locked garage 0.92 · Private driveway 0.96 · Off-street 1.00 · On street outside home 1.06 · On street elsewhere 1.12 |
| 8 | Use class | Social 1.00 · Social + commuting 1.08 · Business class 1 → 1.22 · Business class 2 → 1.38 |
| 9 | Declared modifications | yes → 1.18, else 1.00 |

**No-claims discount** — indexed by years, capped at 9

| Years | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| Discount | 0% | 20% | 30% | 38% | 45% | 50% | 55% | 60% | 63% | 65% |

**Voluntary excess discount**

| Excess | £0 | £250 | £500 | £750 | £1,000 |
|---|---|---|---|---|---|
| Discount | 0% | 4% | 8% | 11% | 14% |

**Minimum premium.** The net premium is floored at **180** before loading and tax. Below the floor the discount stack stops having any effect, which is the trap in any test that sweeps NCD or excess on a cheap risk.

> **Estimated value does not affect the premium.** It is collected, and it drives two underwriting rules (UW-R04, UW-D05), but it is not a rating factor. The same is true of **licence type**, which drives UW-R08 but does not rate. Two identical quotes differing only in value — £18,500 against £74,000 — produce **the same premium** and different outcomes.

### Worked example

Comprehensive · group 14 · value 18,500 · social domestic and pleasure · private driveway · 9,000 miles · no modifications · proposer born `11-02-1988` · Full UK · 6 years held · no claims · no convictions · NCD 5 years · excess £250 · no extras

| Line | Value |
|---|---|
| Base (Comprehensive) | 480.00 |
| Combined factor | ×1.133 |
| Gross | 543.74 |
| Less NCD 50% | |
| Less excess discount 4% | |
| Net | 261.00 |
| IPT at 12% | 31.32 |
| **Total** | **292.32** |

### Discount sweeps on that same quote

| NCD years | 0 | 1 | 3 | 5 | 9 |
|---|---|---|---|---|---|
| Total | 584.63 | 467.71 | 362.47 | 292.32 | 204.62 |

| Excess | £0 | £250 | £500 | £750 | £1,000 |
|---|---|---|---|---|---|
| Total | 304.50 | 292.32 | 280.14 | 271.00 | 261.87 |

---

## 6. Step 4 — Application

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Policy start date | **Text, `dd-mm-yyyy`, with calendar button** | Yes | Non-empty; today or within 60 days | `Policy start date is required.` / `The policy cannot start in the past.` / `The policy must start within the next 60 days.` |
| Payment method | Select | Yes | Annual - single payment / Monthly direct debit | `Select a payment method.` |
| Accuracy declaration | Checkbox | Yes | Ticked | `Confirm the information is accurate.` |
| Disclosure declaration | Checkbox | Yes | Ticked | `Confirm all material facts have been disclosed.` |

> **Payment method is collected but never used.** Monthly direct debit does not produce instalments, interest or a different total. The premium is identical either way.

---

## 7. Step 5 — Review & submit

The review step shows an **indicative outcome** banner (`quote-decision`) computed from the current data, alongside risk, proposer and policy summaries. The banner states that the decision is confirmed on submission.

Advancing to this step revalidates **every earlier step**. A gap anywhere blocks submission with the toast *"Submission blocked — complete the earlier steps."*

On submission the application is rated and underwritten, a reference `MQ-{n}` is assigned, and the record moves to one of the three terminal stages.

---

## 8. The underwriting decision

Fourteen rules, evaluated in **strict precedence order**. The first rule that matches decides the outcome; nothing later is considered.

### Declines — evaluated first, in this order

| # | Code | Condition | Reason text |
|---|---|---|---|
| 1 | UW-D01 | Proposer age under 17 | `The proposer must be at least 17 years old to hold a motor policy.` |
| 2 | UW-D02 | Three or more fault claims | `Three or more fault claims in the last five years falls outside our underwriting appetite.` |
| 3 | UW-D03 | Conviction code begins `DR10` | `A DR10 conviction falls outside our underwriting appetite.` |
| 4 | UW-D04 | Licence held under 1 year **and** group ≥ 15 | `A licence held for less than one year cannot be combined with a vehicle in group 15 or above.` |
| 5 | UW-D05 | Value above 100,000 **and** group ≥ 18 | `A vehicle valued above 100,000 in group 18 or above falls outside our underwriting appetite.` |

### Referrals — evaluated only if no decline matched

| # | Code | Condition | Reason text |
|---|---|---|---|
| 6 | UW-R01 | Age under 21 **and** group ≥ 12 | `A driver under 21 on a vehicle in group 12 or above requires underwriter review.` |
| 7 | UW-R02 | Exactly two fault claims | `Two fault claims in the last five years require underwriter review.` |
| 8 | UW-R03 | Any conviction other than None | `A declared motoring conviction requires underwriter review.` |
| 9 | UW-R04 | Value above 50,000 | `A vehicle valued above 50,000 requires underwriter review.` |
| 10 | UW-R05 | Modifications declared | `Declared modifications require underwriter review.` |
| 11 | UW-R06 | Use class begins `Business` **and** group ≥ 15 | `Business use on a vehicle in group 15 or above requires underwriter review.` |
| 12 | UW-R07 | Mileage above 30,000 | `Annual mileage above 30,000 requires underwriter review.` |
| 13 | UW-R08 | Licence type is `Provisional UK` | `A provisional licence requires underwriter review.` |

### Accept

| # | Code | Condition | Reason text |
|---|---|---|---|
| 14 | UW-A01 | Nothing above matched | `Accepted on standard terms.` |

### Why precedence matters for coverage

Several conditions overlap, and only the earlier one is ever observable.

- **DR10 is both a decline (UW-D03) and a conviction (UW-R03).** A DR10 quote always declines. UW-R03 can only be reached with SP30, CU80 or IN10.
- **Three fault claims (UW-D02) outranks two (UW-R02).** They are mutually exclusive by construction, but a test that sets "three or more" and expects a referral is wrong.
- **UW-D04 and UW-R06 both key on group ≥ 15.** A proposer with under a year's licence on a group-15 business-use vehicle declines; the referral is never seen.
- **Value above 100,000 with group 17** is a *referral* (UW-R04), not a decline — UW-D05 needs group ≥ 18 as well.
- **A single quote can satisfy five referral rules at once** and will report only the earliest.

**One requirement per code.** Fourteen codes with strict precedence cannot be covered by a handful of requirements, and a coverage figure that collapses them says nothing.

---

## 9. The referral queue and the four-eyes control

A referred application waits at `#/insurance/referrals` for an underwriter decision.

### The role gate

Only a user with the **Compliance** role sees the accept and decline actions. For an Adviser or Operations user the buttons are **not rendered at all** — a notice (`uw-role-notice`) appears instead, reading:

> **Read-only for this role** You are acting as {role}. Sign in as the compliance officer, who holds underwriting authority in this demo, to decide referrals.

A test that asserts a disabled button is asserting something that does not exist; assert absence.

### The identity check

A Compliance user who submitted the application themselves passes the role gate and is stopped by a second, independent check, with the toast:

> `Four-eyes control: the submitting user cannot decide a referral they raised.`

The record stays at **Referred**.

**These are two separate mechanisms**, exactly as in the wealth module. Covering only the role gate leaves the control that matters untested.

### Accepting a referral

Opens a modal showing the referral reason and a **loading** field (`uw-loading`, 0–100, step 5). Confirming issues a policy.

The loading is applied to the **net premium before tax**: `total = net + loading + IPT(net + loading)`. A 10% loading on a net of 261.00 therefore adds 10% to the net *and* 12% tax on that addition.

Accepting recalculates the premium from the stored quote data with the loading applied — it does not simply add a percentage to the previously displayed total.

### Declining a referral

Opens a modal with a mandatory reason of **at least 15 characters** (`uw-decline-reason`). Below that, the error `A reason of at least 15 characters is required.` is shown and the decline does not proceed.

---

## 10. Element identification

Every interactive element carries a stable `data-testid`. Prefer these over label text or position.

### Shared shell

`app-picker` · `app-wam` · `app-insurance` · `app-pensions` · `nav-switch-application` · `brand-sub` · `main-nav` · `current-user` · `sign-out` · `reset-data` · `reset-confirm` · `view` · `breadcrumb` · `toasts` · `demo-disclosure` · `sidebar-disclosure`

### Dashboard

`tile-open` · `tile-referrals` · `tile-declined` · `tile-gwp` · `card-funnel` · `card-referrals` · `insfunnel-{stage}` · `dash-new-quote` · `open-referrals`

### Quote list, referrals, policies

`quotes-new` · `quote-row-{id}` · `open-{id}` · `referral-table` · `referral-row-{id}` · `referral-empty` · `back-to-quotes`

### Wizard frame

`quote-card` · `quote-stepper` · `qstep-cover` · `qstep-drivers` · `qstep-quote` · `qstep-application` · `qstep-review` · `quote-back` · `quote-next` · `quote-submit` · `discard-quote` · `discard-quote-confirm` · `qerror-{field}`

### Step 1

`input-cover` · `input-vehicleReg` · `input-vehicleMake` · `input-vehicleModel` · `input-vehicleYear` · `input-vehicleGroup` · `input-vehicleValue` · `input-useClass` · `input-parking` · `input-mileage` · `input-modifications` · `mods-no` · `mods-yes`

### Step 2

`input-proposerFirst` · `input-proposerLast` · `input-proposerDob` · `calendar-proposerDob` · `input-postcode` · `input-licenceType` · `input-licenceYears` · `input-occupation` · `input-faultClaims` · `input-nonFaultClaims` · `input-convictions`

### Step 3

`input-ncdYears` · `input-excess` · `extras-list` · `extra-breakdown` · `extra-legal` · `extra-courtesy` · `premium-summary` · `premium-headline` · `premium-table` · `premium-total`

### Step 4

`input-startDate` · `calendar-startDate` · `input-payment` · `input-declarationAccuracy` · `input-declarationDisclosure`

### Step 5 and the referral decision

`quote-decision` · `review-risk` · `review-proposer` · `review-policy` · `decision-banner` · `policy-banner` · `quote-risk` · `quote-proposer` · `quote-proposer-detail` · `quote-audit` · `quote-not-found` · `uw-role-notice` · `uw-approve` · `uw-approve-confirm` · `uw-loading` · `uw-decline` · `uw-decline-confirm` · `uw-decline-reason` · `uw-decline-reason-error`

---

## 11. Test data that produces each branch

The application's clock is **fixed at 2026-09-22T09:00Z**. All date arithmetic runs from that date, so date-driven tests are deterministic and do not drift.

### Baseline quote — produces ACCEPT (UW-A01), total 292.32

| Field | Value |
|---|---|
| Cover type | `Comprehensive` |
| Registration | `AB12 CDE` |
| Make / Model | `Volkswagen` / `Golf` |
| Year | `2021` |
| Insurance group | `14` |
| Estimated value | `18500` |
| Use class | `Social, domestic and pleasure` |
| Overnight parking | `Private driveway` |
| Annual mileage | `9000` |
| Modifications | No |
| Proposer | `Freya` `Lindqvist`, born `11-02-1988` |
| Postcode | `EC3A 1AB` |
| Licence | `Full UK`, `6` years |
| Occupation | `Accountant` |
| Claims / convictions | `0` / `0` / `None` |
| NCD | `5` |
| Excess | `£250` |
| Policy start | `01-10-2026` |
| Payment | `Annual - single payment` |

### One change from the baseline produces each outcome

Verified premiums, computed from the application's own engine.

| Change | Outcome | Code | Total |
|---|---|---|---|
| *(none — baseline)* | ACCEPT | UW-A01 | 292.32 |
| Fault claims `2` | REFER | UW-R02 | 496.94 |
| Convictions `SP30 - exceeding statutory speed limit` | REFER | UW-R03 | 365.40 |
| Estimated value `74000` | REFER | UW-R04 | 292.32 |
| Modifications Yes | REFER | UW-R05 | 344.93 |
| Use class `Business use - class 1` **and** group `16` | REFER | UW-R06 | 383.83 |
| Annual mileage `31000` | REFER | UW-R07 | 409.24 |
| Licence type `Provisional UK` | REFER | UW-R08 | 292.32 |
| DOB `03-07-2006`, licence `1` year, group `13` | REFER | UW-R01 | 776.03 |
| Fault claims `3` | DECLINE | UW-D02 | 599.25 |
| Convictions `DR10 - driving with excess alcohol` | DECLINE | UW-D03 | 365.40 |
| Licence years `0` **and** group `15` | DECLINE | UW-D04 | 440.02 |
| Value `120000` **and** group `18` | DECLINE | UW-D05 | 336.91 |

Note the three rows whose total equals the baseline exactly — UW-R04, UW-R08 and the baseline itself. Those are the non-rating fields called out in section 5, and they make a useful assertion in their own right: **the outcome changed and the premium did not.**

**UW-D01** (proposer under 17) cannot be reached through the wizard: step 2 validation rejects an under-17 date of birth before submission is possible. The code exists and is unreachable by UI; see section 12.

### Premium floor

Group `1` · mileage `4000` · `Locked garage` · proposer born `04-03-1975` · NCD `9` · excess `£1,000` → gross 241.73, net floored at **180.00**, total **201.60**. Raising the NCD further does not move it.

### Boundaries

Computed from the fixed clock, verified against the application.

| Rule | Just fails | Just passes |
|---|---|---|
| Proposer age 17 | Date of birth `23-09-2009` | `22-09-2009` |
| Policy start in the past | `21-09-2026` | `22-09-2026` (today) |
| Policy start within 60 days | `22-11-2026` (day 61) | `21-11-2026` (day 60) |
| Year of manufacture | `1979` / `2027` | `1980` / `2026` |
| Insurance group | `0` / `51` | `1` / `50` |
| Estimated value | `0` | `1` |
| Annual mileage | `0` | `1` |
| Licence years cap (DOB `11-02-1988`) | `22` | `21` |
| Decline reason length | 14 characters | 15 characters |

> **Note on the age rule.** Age is computed as elapsed milliseconds divided by a 365.25-day year, not as a calendar birthday, exactly as in the wealth module. A date of birth of `23-09-2009` evaluates to 16.9976 years and is rejected; `22-09-2009` is accepted. Arithmetic on calendar birthdays gets this wrong by a day. Tests that only need a valid adult driver should use a date comfortably clear of the boundary.

### Seeded records

Seven quotes are seeded, giving every stage a starting example without building one.

| Reference | Proposer | Stage | Code |
|---|---|---|---|
| MQ-40014 | Freya Lindqvist | Policy issued | UW-A01 |
| MQ-40015 | Owen Brannigan | Referred | UW-R01 |
| MQ-40016 | Priya Raghunathan | Policy issued | UW-A01 |
| MQ-40017 | Callum Whitcombe | Referred | UW-R04 |
| MQ-40018 | Sinead O'Halloran | Policy issued | UW-A01 |
| MQ-40019 | Marcus Adeyemi | Declined | UW-D02 |
| MQ-40020 | Helena Vasquez | Quote | — |

**The seeded records cannot trigger the identity check.** Their `submittedBy` is stored as `Lena Fairbrook - Relationship Manager` with an ordinary hyphen, while the account name is `Lena Fairbrook — Relationship Manager` with an em dash. The identity check compares those two strings exactly, so it never matches on a seeded record. See section 12.

The identity check can therefore only be exercised on a record created during the run, by a user who holds the Compliance role.

---

## 12. Known gaps and constraints

Properties of the application as built, recorded so that tests are written against actual behaviour rather than assumed behaviour.

- **UW-D01 is unreachable through the UI.** Step 2 rejects an under-17 date of birth, so an application that would decline on age can never be submitted. The rule exists in the decision engine and is first in precedence.
- **Occupation is collected and never used.** It is not a rating factor and not an underwriting condition.
- **Payment method is collected and never used.** Monthly direct debit produces no instalments, no interest and no change to the total.
- **Estimated value and licence type do not rate.** Both drive underwriting outcomes only. Two quotes differing only in value produce identical premiums.
- **Non-fault claims rate but never refer.** They carry a 0.05 factor each; only fault claims reach the underwriting rules.
- **The `Application` lifecycle stage is unreachable.** It exists in the stage pill map but a submission goes straight from Quote to a terminal stage.
- **There is no quote expiry.** A quote persists indefinitely and its premium is recalculated from stored data whenever it is displayed, so a quote never goes stale and never needs re-quoting.
- **The premium floor silently swallows discounts.** Below a net of 180 the NCD and excess discounts stop affecting the total. A sweep test on a cheap risk will see the same number repeatedly and it is not a bug.
- **Loading is applied before tax**, so a stated loading percentage increases the total by slightly more than that percentage.
- **The underwriting role is the Compliance role.** There is no separate underwriter account, and only one Compliance account exists.
- **The identity check never fires on a seeded record.** Seeded quotes store `submittedBy` as `Lena Fairbrook - Relationship Manager` (hyphen) while the account name is `Lena Fairbrook — Relationship Manager` (em dash). The check is an exact string comparison, so the two never match. Only a record submitted during the test run can trigger it. A requirement that uses a seeded referral to cover the four-eyes identity check will pass for the wrong reason.
- **The role gate renders nothing** for non-Compliance users. Assert absence, not a disabled state.
- **State is held in browser local storage** under the key `cavendish.platform.v1`, shared with the Private Wealth and Pensions applications in this build. An older standalone wealth build remains deployed at the site root under `cavendish.cmo.v1`, so the two deployments never see each other's records even though they share an origin. Point a journey at `platform.html` explicitly; the bare site root reaches that other application.
- **Reset demo data is scoped to the application you are in.** Resetting from any `#/insurance` route restores the seeded quotes and policies, any quote draft and the quote and policy counters, and **leaves wealth and pensions exactly as they are**. **The Applications launcher carries no reset control.** On `#/apps` the `reset-data` button is **hidden, not removed**: it stays in the DOM carrying the `hidden` attribute and `display:none`. A journey must therefore assert that it is **not visible**; asserting that the element does not exist will fail. No application is in scope on the launcher, so there is nothing for it to reset. A full reset of all three is on the sign in screen (`login-reset`), which is single-step with no confirmation modal. Any run that depends on a clean start should open the application it is about to exercise, then reset from inside it.
- **Reset demo data is a two-step control.** `reset-data` opens a confirmation modal; the reset happens on `reset-confirm` ("Reset now"). A single click changes nothing, and the modal names the application about to be reset, so it can be asserted.
- **Any reset clears a sign-in lockout.** The failed-attempt counters and the locked list are not demo records, so they are cleared whichever application you reset from. That keeps the sign-in message *"This account is locked after 5 failed sign-in attempts. Reset the demo data to unlock it."* true from everywhere.
- **Local storage belongs to one browser profile.** Two people demonstrating on their own machines, or in two different browsers on one machine, never affect each other's data; there is no server and no shared state. Two tabs of the same application in the same profile do share storage, and neither tab is told when the other writes.
- **The application is a single self-contained HTML file.** No server, no API, no network calls. Rating and underwriting run in the browser. There is nothing to stub and nothing to seed through an interface — every test starts from the UI.
- **The application must be served over HTTP to be automated remotely.** Run from a local `file://` path it is reachable only by a browser on that same machine.

---

## 13. The end-to-end journey

The canonical path from a blank quote to an issued policy through a referral — the path that exercises the four-eyes control.

**Preconditions:** application reachable over HTTP. Credentials supplied from environment variables.

| # | Step | Expected |
|---|---|---|
| 0 | Navigate to the application | Sign-in screen; nothing else reachable |
| 1 | Sign in as `desmond.achebe` | Application launcher shown with three cards; user is Desmond Achebe, role Compliance |
| 2 | Open the application with `app-insurance`, then reset demo data from inside it: `reset-data`, then `reset-confirm` | The launcher carries no reset control, so this cannot be done at step 1. Restores **motor insurance only**; wealth and pensions are left exactly as they are. Any draft cleared, still signed in. For a full reset of all three, use `login-reset` on the sign in screen before signing in |
| 3 | Activate `app-insurance`, or navigate to `#/insurance/quote/new` directly | Quote wizard at Step 1 of 5, Back disabled |
| 4 | Complete cover and vehicle using the baseline, but set **Estimated value `74000`** | — |
| 5 | Continue | Step 2 of 5, Driver & history |
| 6 | Complete the proposer, licence, occupation, claims and convictions | — |
| 7 | Continue | Step 3 of 5, Quote |
| 8 | Select NCD `5` and excess `£250` | `premium-headline` shows a total; premium table itemises the factors |
| 9 | Continue | Step 4 of 5, Application |
| 10 | Enter policy start `01-10-2026`, select payment, tick both declarations | — |
| 11 | Continue | Step 5 of 5, Review & submit |
| 12 | Verify `quote-decision` | Indicative outcome **REFER (UW-R04)** |
| 13 | Submit | Toast with the assigned `MQ-` reference; quote detail opens at stage **Referred** |
| 14 | Open `#/insurance/referrals` and select the record | Accept and Decline actions are visible — Desmond holds the Compliance role |
| 15 | Activate `uw-approve` | **Blocked.** Toast: *"Four-eyes control: the submitting user cannot decide a referral they raised."* Record stays at Referred |
| 16 | **Sign out, then sign in as a different Compliance-role user** | Signing out returns to the launcher; sign in and activate `app-insurance` again |
| 17 | Open the referral and activate `uw-approve` | Modal with the referral reason and a loading field |
| 18 | Enter a loading and confirm | Stage **Policy issued**; `POL-` reference assigned; premium recalculated with the loading; audit records the acceptance |

> **Step 15 is the point of this journey.** A journey that submits as an Adviser and decides as Desmond exercises only the role gate. To exercise the identity check, the *same Compliance user* must attempt to decide their own submission — which is why step 1 signs in as Desmond rather than Lena.
>
> **A caveat worth knowing before writing step 16.** The application ships **one** Compliance account. A journey that must reach a completed acceptance therefore has to submit as Lena (Adviser) and decide as Desmond, and the identity check can only be *observed being triggered*, not stepped past. Cover the identity check and the completed acceptance as **two journeys**, not one.

### Branch journeys worth deriving from this one

| Branch | Diverges at |
|---|---|
| Straight-through acceptance | Step 4 — baseline value; submission issues a policy immediately |
| Each of the seven reachable referral codes | Step 4 or 6 — one field change each |
| Each of the four reachable decline codes | Step 4 or 6 |
| Precedence: DR10 declines rather than refers | Step 6 |
| Precedence: licence under 1 year on group 15 declines rather than refers | Step 6 |
| Referral declined instead of accepted | Step 17 — reason of at least 15 characters |
| Role gate | Step 14 as `lena.fairbrook` or `yuki.mori` — actions absent |
| Premium floor | Step 4 and 8 — cheap risk, maximum NCD and excess |
| Non-rating fields | Step 4 — value and licence type change the outcome, not the premium |
| Licence-years cap | Step 6 |
| Policy start date window | Step 10 |

---

## 14. Notes for automation

### Select options must match exactly

Every option string in this module is ASCII — no typographic dashes. Values typed with an ordinary keyboard hyphen match.

Three option sets contain **prefix collisions**, where one option's full text opens another's:

| Dropdown | Colliding options |
|---|---|
| Cover type | `Third party, fire and theft` · `Third party only` |
| Use class | `Social, domestic and pleasure` · `Social, domestic, pleasure and commuting` |
| Use class | `Business use - class 1` · `Business use - class 2` |

Select by the complete string. The placeholder `Select…` is a selectable option that fails validation.

The **voluntary excess** options display as formatted currency — `£0`, `£250`, `£1,000` — while the stored value is the bare number. Select by the displayed text.

### Dates are typed, not picked

`dd-mm-yyyy` into `input-proposerDob` and `input-startDate`. ISO is also accepted on entry and redisplays in `dd-mm-yyyy`. The record always stores ISO, which is why the boundary table gives both. The calendar buttons exist for human use; automation should type the value.

### Fields accept keystrokes

Every text and number field can be typed into character by character. Values entered by setting the value and firing a `change` event — the path most WebDriver-based tools take for a dropdown — are accepted on every control.

### Nothing is asynchronous

Unlike the wealth module, this module has no asynchronous operation. Rating and underwriting are synchronous. No waits are required beyond the ordinary re-render, and a generated fixed pause is pure cost.

### Re-read the form after a revealing control

Changing the step re-renders the wizard body. Within a step, no control on the quote wizard adds or removes fields, so a single read per step is sufficient — the exception being step 3, where changing NCD, excess or an extra recalculates and re-renders the premium block.

### Verify the step before entering data

A validation failure keeps the user on the current step. Check `quote-card` shows the expected *Step n of 5* before interacting with fields on it, or a journey will enter step 3 values into step 2 fields and fail somewhere unrelated.

### Recover from validation errors

Every mandatory field is reported at once on Continue, each message naming its field, in a `qerror-{field}` element. After activating Continue, if the step counter has not advanced, read the messages, set the fields they name and activate Continue again rather than repeating the same action.

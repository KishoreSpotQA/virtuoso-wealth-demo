# Cavendish Pensions — Retirement Income & Protection

**System:** Cavendish Client Ops — Pensions module
**Build:** 2026.09.27
**Module:** Pensions → Instructions, and Dashboards → Find requests
**Application URL:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/platform.html`
**Document type:** End-to-end process specification
**Status:** Current — describes the application as built

> **Scope of this document.** This describes the **pensions module only**, which carries two distinct processes:
>
> 1. **Retirement instruction** — the five-step workflow, settlement calculation, retirement decision, review queue and payment authorisation control. Sections 3 to 9.
> 2. **Dashboards member matching** — find requests arriving from a pensions dashboard, the scheme's matching rules, and the resolution of a possible match. Section 15.
>
> They share the sign-in, the launcher and the member population but are otherwise independent, and a requirement generated from one should not reference the other.
>
> The same build also carries a **wealth** module and a **motor insurance** module, each documented separately. Sign-in, the launcher and the navigation shell are shared; everything else here is specific to pensions.
>
> A separate **standalone wealth application** is deployed at the site root. It is a different deployment with its own storage and does not contain this module. The hash routes below only work against `/platform.html`.
>
> The launcher can be bypassed: `platform.html#/pensions/case/new` resolves straight to the wizard after sign-in.

---

## 1. What this process does

Cavendish Pensions takes a member from a blank retirement instruction to benefits in payment. The process is a **five-step wizard** completed by a Relationship Manager. On submission the instruction is priced and adjudicated in one action, and the outcome is decided by rule.

**No instruction reaches payment without a second person.** An accepted instruction waits for payment authorisation; a referred one waits for a retirement review. Either way a different user decides, and only then is a payment reference issued.

### Actors

The same three accounts serve all three modules.

| Actor | Name in the system | Username | Role | What they can do here |
|---|---|---|---|---|
| Adviser | Lena Fairbrook — Relationship Manager | `lena.fairbrook` | Adviser | Capture and submit a retirement instruction |
| Reviewer | Desmond Achebe — Compliance Officer | `desmond.achebe` | Compliance | Authorise a payment; approve or decline a referral |
| Operations | Yuki Mori — Client Operations | `yuki.mori` | Operations | View records |

Password for all three: `Cavendish#2026`.

**The retirement review and payment authority sits with the `Compliance` role.** There is no separate pensions reviewer account, and only one Compliance account exists. A requirement that assumes a distinct reviewer is asserting behaviour the application does not have.

The acting user is established by **signing in**. There is no role selector.

### Navigation

| Route | Screen | Notes |
|---|---|---|
| `#/apps` | Application launcher | `app-pensions` opens this module |
| `#/pensions` | Retirement dashboard | Instructions, referrals, authorisations, fund value |
| `#/pensions/cases` | Instructions | Every instruction, whatever stage |
| `#/pensions/case/new` | **New instruction wizard** | The process described here |
| `#/pensions/case/{id}` | Instruction detail | Member, settlement, audit; review and authorisation actions |
| `#/pensions/advice` | Retirement review queue | Referred instructions |
| `#/pensions/authorisations` | Payment authorisation | Accepted instructions awaiting release |
| `#/pensions/payments` | Benefits in payment | Settled instructions |
| `#/pensions/finds` | Find requests | Every request received from a dashboard |
| `#/pensions/find/new` | **New find request** | Section 15 |
| `#/pensions/find/{id}` | Find request detail | Identifier agreement, candidate record, audit; resolution action |
| `#/pensions/resolution` | Possible match resolution | Requests the rules could not settle |
| `#/pensions/members` | Member register | The records a find request is matched against |

### Record lifecycle

```
Case ──submit──> ACCEPT  ──> Awaiting authorisation ──authorise──> In payment
                                                    ──reject─────> Declined
                 REFER   ──> Referred ──approve──> In payment
                                      ──decline──> Declined
                 DECLINE ──> Declined
```

**Every instruction receives exactly one independent human check — never two, never none.** An accepted instruction is checked at payment authorisation. A referred one is checked at retirement review, and an approved referral goes straight to payment rather than round again for a second authorisation. A declined instruction is terminal and is never reviewed.

That symmetry is the control model, and it is the single most important thing to understand before writing requirements against this module.

An instruction reference `PN-{n}` is assigned on submission. A payment reference `PAY-{n}` is assigned **only when a human releases payment** — never on submission, not even for an accepted instruction.

---

## 2. Sign in and the launcher

Identical to the other modules. The behaviour that matters:

- The failure message is **identical** for a wrong password and an unknown username.
- **Blank-field submission does not count towards the lockout** — field validation runs before any credential check.
- The account locks on the **fifth** failed attempt with no prior warning, and the lockout **has no time-based expiry** — it clears only on a demo data reset.
- **Signing out returns to the launcher.** A journey that signs out mid-run and signs back in lands on the application picker, not on the module it left. A journey that deep-links to a route while signed out still resolves to that route after signing in.

| Element | `data-testid` | Opens |
|---|---|---|
| Launcher container | `app-picker` | — |
| Wealth & Asset Management | `app-wam` | `#/dashboard` |
| Motor Insurance | `app-insurance` | `#/insurance` |
| Pensions | `app-pensions` | `#/pensions` |
| Switch application | `nav-switch-application` | Returns to the launcher |

**The platform is branded as a group.** The sign-in card and the launcher read *Cavendish · Financial Services · Client Ops*. Once inside a module the sidebar sub-line (`brand-sub`) names that business instead — *Retirement · Client Ops* here.

---

## 3. Step 1 — Member & fund

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| First name | Text | Yes | Non-empty | `First name is required.` |
| Last name | Text | Yes | Non-empty | `Last name is required.` |
| Date of birth | **Text, `dd-mm-yyyy`, with calendar button** | Yes | Non-empty; age ≥ 18; age ≤ 100 | `Date of birth is required.` / `The member must be at least 18 years old.` / `Enter a valid date of birth.` |
| Member reference | Text | Yes | Non-empty | `Member reference is required.` |
| Scheme type | Select | Yes | Non-empty | `Select a scheme type.` |
| Plan number | Text | Yes | Non-empty | `Plan number is required.` |
| Current fund value (GBP) | Number | Yes | Non-empty; greater than zero | `Current fund value is required.` / `Enter a fund value greater than zero.` |
| Employment status | Select | Yes | Non-empty | `Select an employment status.` |
| Country of residence | Select | Yes | Non-empty | `Select a country of residence.` |
| Other taxable income (GBP) | Number | Yes | Non-empty; not negative | `Enter other taxable income, or zero.` |

### Option lists

**Scheme type**
Personal pension · Stakeholder pension · Self-invested personal pension · Group personal pension · Occupational money purchase

**Employment status**
Employed · Self-employed · Semi-retired · Retired · Not working

**Country of residence**
United Kingdom · Republic of Ireland · Other EEA · Rest of world

> **Step 1 does not enforce the minimum pension age.** It rejects only an implausible date of birth — under 18 or over 100. A member aged 51 passes validation and is **declined at submission** by RP-D01. This is deliberate and it matters: unlike the motor insurance module, where the age decline is unreachable through the UI, RP-D01 here is fully reachable and testable.

---

## 4. Step 2 — Protection & safeguards

These checks decide whether benefits can be put into payment at all, and whether the member must be referred first.

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Safeguarded benefits held | Select | Yes | Non-empty | `Select the safeguarded benefits held, or None.` |
| Guidance or advice status | Select | Yes | Non-empty | `Select the guidance or advice status.` |
| Protected pension age | Radio | Yes | Answered | `Answer the protected pension age question.` |
| Protected tax-free cash (per cent) | Number | **No** | If given, 0–100 | `Enter a percentage between 0 and 100.` |
| Money purchase annual allowance triggered | Radio | Yes | Answered | `Answer the money purchase annual allowance question.` |
| Vulnerability declared | Radio | Yes | Answered | `Answer the vulnerability question.` |
| Nature of support required | Textarea | Only when vulnerability = Yes | At least 20 characters | `Describe the support required in at least 20 characters.` |
| ScamSmart warning given | Checkbox | Yes | Ticked | `Confirm the ScamSmart warning has been given.` |

### Option lists

**Safeguarded benefits held**
None · Guaranteed annuity rate · Guaranteed minimum pension · Section 9(2B) rights

**Guidance or advice status**
Pension Wise guidance taken · Regulated advice taken · Guidance declined - opt out recorded

### Revealing control

Answering **Yes** to the vulnerability question reveals a mandatory textarea (`vulnerability-panel`, `input-vulnerabilityDetail`). Answering No removes it. An agent working from a stale read of the page will type into a field that is no longer there.

### The role of regulated advice

`Regulated advice taken` is the only guidance value that clears the safeguarded-benefits decline. A member holding safeguarded benefits on a fund above 30,000 is declined under RP-D02 with Pension Wise guidance, and **accepted** with regulated advice, all else equal. That single field flips the outcome.

---

## 5. Step 3 — Retirement option

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| How benefits are to be taken | Select | Yes | Non-empty | `Select how benefits are to be taken.` |
| Tax-free lump sum requested (GBP) | Number | Yes | Not negative; not above the fund value | `Tax-free lump sum is required; enter zero if none is taken.` / `The tax-free lump sum cannot be negative.` / `The tax-free lump sum cannot exceed the fund value.` |

**Retirement options**
Guaranteed annuity · Flexi-access drawdown · Uncrystallised funds pension lump sum · Full encashment

> **The tax-free cash maximum is not enforced by validation.** Step 3 only stops a lump sum larger than the whole fund. A request above the permitted 25 per cent is accepted by the form and **declined at submission** by RP-D04. The field hint states the permitted maximum for the fund entered.

### Conditional panels

Each option reveals its own fields. Changing the option replaces them.

| Option | Panel | Fields |
|---|---|---|
| Guaranteed annuity | `annuity-panel` | Annuity shape, Guarantee period, Underwriting basis — all mandatory |
| Flexi-access drawdown | `drawdown-panel` | Initial annual income (mandatory, above zero), Payment frequency, Income indexed radio |
| Uncrystallised funds pension lump sum | `ufpls-panel` | Lump sum requested — mandatory, above zero, not more than the fund after the tax-free cash |
| Full encashment | `encashment-notice` | No extra fields; a warning banner is shown |

**Annuity shape**
Level, single life · Level, joint life 50 per cent · Escalating 3 per cent, single life · Escalating 3 per cent, joint life 50 per cent

**Guarantee period** — None · 5 years · 10 years
**Underwriting basis** — Standard terms · Enhanced terms - declared medical factors
**Payment frequency** — Monthly · Quarterly · Annually

Additional messages: `Initial annual income is required.` / `Enter an annual income greater than zero.` / `Select a payment frequency.` / `Answer the income escalation question.` / `Select an annuity shape.` / `Select a guarantee period.` / `Select an underwriting basis.` / `Lump sum requested is required.` / `Enter a lump sum greater than zero.` / `The lump sum cannot exceed the fund remaining after the tax-free cash.`

---

## 6. Step 4 — Income modelling

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Intended first payment date | **Text, `dd-mm-yyyy`, with calendar button** | Yes | Today or within 90 days | `Intended first payment date is required.` / `The first payment cannot be dated in the past.` / `The first payment must fall within the next 90 days.` |
| Investment growth assumption | Select | Yes | Non-empty | `Select an investment growth assumption.` |

**Growth assumption** — 2 per cent - cautious · 4 per cent - central · 6 per cent - optimistic

The step displays a live settlement table (`settlement-table`) and a headline figure (`income-headline`). Changing the fund value, tax-free cash, drawdown income, lump sum or other income updates the table without re-rendering the page.

### The settlement calculation

**Tax-free cash maximum** = the lesser of the fund × 25 per cent (or the protected percentage, where one is recorded) and the lump sum allowance of **268,275**.

**Annuity.** Gross annual income = the fund after tax-free cash × the annuity rate.

Market rate by age:

| Age | Under 60 | 60–64 | 65–69 | 70–74 | 75+ |
|---|---|---|---|---|---|
| Rate | 4.20% | 4.85% | 5.65% | 6.60% | 7.80% |

Adjusted by: joint life 50 per cent ×0.88 · escalating 3 per cent ×0.72 · 5-year guarantee ×0.98 · 10-year guarantee ×0.95 · enhanced terms ×1.18.

Where the member holds a **guaranteed annuity rate**, a rate of **9.50 per cent** replaces the market rate whenever it is higher. On the baseline that is 9.50 against a market 4.85 — very nearly double the income.

**Flexi-access drawdown.** The requested income is taken each year and the remaining fund grows at the assumption less a **0.75 per cent** annual charge. Indexed income escalates at 2.5 per cent. The projection reports the age at which the fund is exhausted (`depletion-age`).

**UFPLS.** 25 per cent of the lump sum is tax free; 75 per cent is taxed as income.

**Full encashment.** The balance after the tax-free lump sum is taxed as income in one year.

**Income tax** is banded on top of other taxable income:

| Band | Threshold | Rate |
|---|---|---|
| Personal allowance | to 12,570 | 0% |
| Basic rate | to 50,270 | 20% |
| Higher rate | to 125,140 | 40% |
| Additional rate | above | 45% |

---

## 7. Step 5 — Review & submit

An indicative outcome banner (`case-decision`) shows the decision the current data would produce, alongside member, protection and settlement summaries.

| Field | Control | Mandatory | Exact error message |
|---|---|---|---|
| Tax and long-term provision explained | Checkbox | Yes | `Confirm the tax and long-term provision have been explained.` |
| Member understands it cannot be reversed | Checkbox | Yes | `Confirm the member understands this cannot be reversed.` |

Advancing to this step revalidates **every earlier step**. A gap anywhere blocks submission with the toast *"Submission blocked - complete the earlier steps."*

---

## 8. The retirement decision

Sixteen rules, evaluated in **strict precedence order**. The first match decides; nothing later is considered.

### Declines — evaluated first, in this order

| # | Code | Condition | Reason text |
|---|---|---|---|
| 1 | RP-D01 | Age under 55 without a protected pension age | `Benefits cannot be taken before the normal minimum pension age of 55 without a protected pension age.` |
| 2 | RP-D02 | Safeguarded benefits held, fund above 30,000, regulated advice not confirmed | `Safeguarded benefits valued above 30,000 require confirmed regulated advice before they can be given up.` |
| 3 | RP-D03 | Flexi-access drawdown on a fund below 10,000 | `A flexi-access drawdown arrangement cannot be established below a fund value of 10,000.` |
| 4 | RP-D04 | Tax-free cash above the permitted maximum | `The tax-free lump sum requested exceeds the permitted maximum for this arrangement.` |
| 5 | RP-D05 | Flexi-access drawdown, resident outside the UK, Ireland and the EEA | `Flexi-access drawdown is not offered to members resident outside the United Kingdom, Ireland and the EEA.` |

### Referrals — evaluated only if no decline matched

| # | Code | Condition | Reason text |
|---|---|---|---|
| 6 | RP-R01 | Fund above 500,000 | `A fund above 500,000 requires a large-case review before benefits are put into payment.` |
| 7 | RP-R02 | Vulnerability declared | `A declared vulnerability requires a specialist review before benefits are put into payment.` |
| 8 | RP-R03 | Drawdown income exhausts the fund before age 85 | `The requested income depletes the fund before age 85 and requires a sustainability review.` |
| 9 | RP-R04 | Guidance declined | `Guidance has been declined without regulated advice; the member must be referred for a suitability conversation.` |
| 10 | RP-R05 | Protected tax-free cash above 25 per cent | `Protected tax-free cash above 25 per cent must be verified against the scheme records.` |
| 11 | RP-R06 | Full encashment above 50,000 | `A full encashment above 50,000 requires a tax-consequence review before payment.` |
| 12 | RP-R07 | Guaranteed annuity rate held, a non-annuity option chosen | `A guaranteed annuity rate is available and is being given up by the option chosen.` |
| 13 | RP-R08 | Under 60 taking full encashment or UFPLS | `A member under 60 taking full access to the fund requires a review of long-term provision.` |
| 14 | RP-R09 | MPAA triggered while employed or self-employed | `The money purchase annual allowance is already triggered while contributions continue.` |
| 15 | RP-R10 | Resident outside the United Kingdom | `A member resident outside the United Kingdom requires overseas payment checks.` |

### Accept

| # | Code | Condition | Reason text |
|---|---|---|---|
| 16 | RP-A01 | Nothing above matched | `Accepted on standard terms.` |

### Why precedence matters for coverage

Several conditions overlap, and only the earlier one is observable.

- **Regulated advice flips RP-D02 to RP-A01.** Safeguarded benefits above 30,000 decline on guidance and accept on advice, with no other change.
- **RP-R03 outranks RP-R07.** A member giving up a guaranteed annuity rate for an *unsustainable* drawdown income reports the sustainability referral, never the give-up. To reach RP-R07 the option must be sustainable — UFPLS is the reliable way.
- **RP-D05 outranks RP-R10.** A member in the Rest of world taking drawdown is declined; the overseas-payment referral is never seen. RP-R10 is only reachable for a non-drawdown option or an EEA resident.
- **RP-D01 outranks everything.** A 51-year-old on a 620,000 fund declines on age; the large-case referral is not reported.
- **RP-R06 and RP-R08 both key on full encashment.** Above 50,000 reports RP-R06; under 60 on a smaller pot reports RP-R08.

**One requirement per code.** Sixteen codes under strict precedence cannot be covered by a handful of requirements, and a coverage figure that collapses them says nothing.

---

## 9. The two four-eyes controls

### Payment authorisation — the accept path

An accepted instruction sits at **Awaiting authorisation** with **no payment reference**. `#/pensions/authorisations` lists them.

- **Authorise** (`pa-authorise`) opens a modal requiring a note of at least 20 characters (`pa-authorise-note`). Confirming issues the `PAY-` reference and moves the instruction to In payment.
- **Reject** (`pa-reject`) requires a reason of at least 20 characters and moves the instruction to Declined.

### Retirement review — the refer path

A referred instruction waits at `#/pensions/advice`.

- **Approve** (`pa-approve`) requires a note of at least 20 characters and moves straight to In payment with a `PAY-` reference. There is no further authorisation.
- **Decline** (`pa-decline`) requires a reason of at least 20 characters.

### The role gate

Only a **Compliance** user sees any of these actions. For an Adviser or Operations user the buttons are **not rendered at all**; a notice appears instead (`pa-role-notice`, `auth-role-notice`):

> **Read-only for this role** You are acting as {role}. Sign in as the compliance officer, who holds retirement review authority in this demo, to decide referrals.

A test asserting a disabled button is asserting something that does not exist. Assert absence.

### The identity check

A Compliance user who submitted the instruction passes the role gate and is stopped by a second, independent check:

| Action | Toast |
|---|---|
| Authorise or reject | `Four-eyes control: the submitting user cannot authorise a payment they raised.` |
| Approve or decline a referral | `Four-eyes control: the submitting user cannot decide a referral they raised.` |

The record stays where it was. **These are two separate mechanisms** — covering only the role gate leaves the control that matters untested.

---

## 10. Element identification

Every interactive element carries a stable `data-testid`.

### Shared shell

`app-picker` · `app-wam` · `app-insurance` · `app-pensions` · `nav-switch-application` · `main-nav` · `current-user` · `sign-out` · `reset-data` · `reset-confirm` · `view` · `breadcrumb` · `brand-sub` · `demo-disclosure` · `sidebar-disclosure`

### Dashboard

`tile-cases` · `tile-referrals` · `tile-authorisations` · `tile-payment` · `tile-funds` · `card-funnel` · `card-referrals` · `penfunnel-{stage}` · `dash-new-case` · `open-advice` · `open-authorisations`

### Lists and queues

`cases-new` · `case-row-{id}` · `open-{id}` · `advice-table` · `advice-row-{id}` · `advice-empty` · `authorisation-table` · `auth-row-{id}` · `authorisation-empty` · `payment-table` · `back-to-cases`

### Wizard frame

`case-card` · `case-stepper` · `pstep-member` · `pstep-safeguards` · `pstep-option` · `pstep-modelling` · `pstep-review` · `case-back` · `case-next` · `case-submit` · `discard-case` · `discard-case-confirm` · `perror-{field}`

### Step 1

`input-memberFirst` · `input-memberLast` · `input-memberDob` · `calendar-memberDob` · `input-memberRef` · `input-schemeType` · `input-planNumber` · `input-potValue` · `input-employmentStatus` · `input-residence` · `input-otherIncome`

### Step 2

`input-safeguardedBenefits` · `input-guidanceStatus` · `input-protectedPensionAge` · `protectedPensionAge-yes` · `protectedPensionAge-no` · `input-protectedCashPct` · `input-mpaaTriggered` · `mpaaTriggered-yes` · `mpaaTriggered-no` · `input-vulnerability` · `vulnerability-yes` · `vulnerability-no` · `vulnerability-panel` · `input-vulnerabilityDetail` · `input-scamWarning`

### Step 3

`input-retirementOption` · `input-taxFreeCash` · `annuity-panel` · `input-annuityShape` · `input-guaranteePeriod` · `input-underwritingBasis` · `drawdown-panel` · `input-drawdownIncome` · `input-paymentFrequency` · `input-incomeIndexed` · `incomeIndexed-yes` · `incomeIndexed-no` · `ufpls-panel` · `input-ufplsAmount` · `encashment-notice`

### Step 4

`input-firstPaymentDate` · `calendar-firstPaymentDate` · `input-growthAssumption` · `settlement-summary` · `income-headline` · `settlement-live` · `settlement-table` · `depletion-age` · `tax-total` · `net-total`

### Step 5, detail and decisions

`case-decision` · `review-member` · `review-protection` · `review-settlement` · `input-declarationUnderstood` · `input-declarationIrreversible` · `decision-banner` · `authorisation-banner` · `payment-banner` · `decline-banner` · `case-member` · `case-settlement` · `case-audit` · `case-not-found` · `pa-role-notice` · `auth-role-notice` · `pa-approve` · `pa-approve-confirm` · `pa-approve-note` · `pa-approve-note-error` · `pa-decline` · `pa-decline-confirm` · `pa-decline-reason` · `pa-decline-reason-error` · `pa-authorise` · `pa-authorise-confirm` · `pa-authorise-note` · `pa-authorise-note-error` · `pa-reject` · `pa-reject-confirm` · `pa-reject-reason` · `pa-reject-reason-error`

---

## 11. Test data that produces each branch

The application's clock is **fixed at 2026-09-22T09:00Z**. All date arithmetic runs from that date, so date-driven tests are deterministic.

### Baseline instruction — produces ACCEPT (RP-A01)

| Field | Value |
|---|---|
| Member | `Rosalind` `Fairweather`, born `18-04-1966` (age 60) |
| Member reference | `CVP-004821` |
| Scheme | `Personal pension`, plan `PP-88213` |
| Current fund value | `240000` |
| Employment status | `Retired` |
| Country of residence | `United Kingdom` |
| Other taxable income | `14000` |
| Safeguarded benefits | `None` |
| Guidance | `Pension Wise guidance taken` |
| Protected pension age | No · MPAA No · Vulnerability No · ScamSmart ticked |
| Option | `Guaranteed annuity`, tax-free cash `60000` |
| Annuity | `Level, single life`, guarantee `None`, `Standard terms` |
| First payment | `22-10-2026`, growth `4 per cent - central` |

Maximum tax-free cash for this fund: **60,000**.

Settlement: rate **4.850%** · gross income **£8,730** · tax **£1,746.00** · net **£6,984.00**.

### One change from the baseline produces each code

Verified against the application's own engines.

| Change | Outcome | Code | Taxable | Tax |
|---|---|---|---|---|
| *(none — baseline)* | ACCEPT | RP-A01 | 8,730 | 1,746.00 |
| Date of birth `01-06-1975` | DECLINE | RP-D01 | 7,560 | 1,512.00 |
| Safeguarded `Guaranteed minimum pension` | DECLINE | RP-D02 | 8,730 | 1,746.00 |
| Drawdown, fund `9000`, cash `2250`, income `500` | DECLINE | RP-D03 | 500 | 100.00 |
| Tax-free cash `70000` | DECLINE | RP-D04 | 8,245 | 1,649.00 |
| Drawdown, residence `Rest of world`, income `9000` | DECLINE | RP-D05 | 9,000 | 1,800.00 |
| Fund `620000` | REFER | RP-R01 | 27,160 | 5,432.00 |
| Vulnerability Yes + detail | REFER | RP-R02 | 8,730 | 1,746.00 |
| Drawdown, income `30000` | REFER | RP-R03 | 30,000 | 6,000.00 |
| Guidance `Guidance declined - opt out recorded` | REFER | RP-R04 | 8,730 | 1,746.00 |
| Protected cash `30` | REFER | RP-R05 | 8,730 | 1,746.00 |
| Full encashment, fund `84000`, cash `21000` | REFER | RP-R06 | 63,000 | 17,946.00 |
| GAR held + UFPLS, fund `28000`, cash `0`, lump sum `10000` | REFER | RP-R07 | 7,500 | 1,500.00 |
| DOB `10-02-1970`, full encashment, fund `40000`, cash `10000` | REFER | RP-R08 | 30,000 | 6,000.00 |
| MPAA Yes + employment `Employed` | REFER | RP-R09 | 8,730 | 1,746.00 |
| Residence `Republic of Ireland` | REFER | RP-R10 | 8,730 | 1,746.00 |

All sixteen codes are reachable through the user interface.

### Settlement worked examples

Baseline member, one variable changed:

| Variation | Rate | Gross income | Tax | Net |
|---|---|---|---|---|
| Level, single life | 4.850% | £8,730.00 | £1,746.00 | £6,984.00 |
| Level, joint life 50 per cent | 4.268% | £7,682.40 | £1,536.60 | £6,145.80 |
| Escalating 3 per cent, single life | 3.492% | £6,285.60 | £1,257.20 | £5,028.40 |
| 5-year guarantee | 4.753% | £8,555.40 | £1,711.20 | £6,844.20 |
| 10-year guarantee | 4.607% | £8,292.60 | £1,658.60 | £6,634.00 |
| Enhanced terms | 5.723% | £10,301.40 | £2,060.40 | £8,241.00 |

**Guaranteed annuity rate.** Fund `28000`, cash `7000`, GAR held: market rate 4.850%, applied rate **9.500%**, income **£1,995**.

**Drawdown depletion**, baseline member, income 9,000:

| Growth assumption | Fund exhausted at age |
|---|---|
| 2 per cent - cautious | 83 |
| 4 per cent - central | 92 |
| 6 per cent - optimistic | not within 50 years |

Indexing the same income at 2.5 per cent brings the central case forward from **92 to 82** — enough to cross the sustainability threshold on its own.

**Income tax**, with no other income:

| Taxable amount | Tax | Bands reached |
|---|---|---|
| 5,000 | £0.00 | Personal allowance |
| 20,000 | £1,486.00 | Personal allowance, basic |
| 60,000 | £11,432.00 | to higher rate |
| 200,000 | £71,175.00 | to additional rate |

### Boundaries

Computed from the fixed clock and verified against the application.

| Rule | Just fails | Just passes |
|---|---|---|
| Minimum pension age 55 | Date of birth `23-09-1971` | `22-09-1971` |
| Member age 18 | Date of birth `22-09-2008` | `21-09-2008` |
| First payment in the past | `21-09-2026` | `22-09-2026` (today) |
| First payment within 90 days | `22-12-2026` (day 91) | `21-12-2026` (day 90) |
| Fund value | `0` | `1` |
| Drawdown minimum fund | `9999` | `10000` |
| Safeguarded advice threshold | `30001` without advice declines | `30000` without advice accepts |
| Large-case referral | `500000` accepts | `500001` refers |
| Encashment referral | `50000` | `50001` refers |
| Protected cash referral | `25` per cent | `26` per cent refers |
| Vulnerability detail | 19 characters | 20 characters |
| Reviewer note and reason | 19 characters | 20 characters |

> **Note on the age rules.** Age is computed as elapsed milliseconds divided by a 365.25-day year, not as a calendar birthday. The cut-off therefore lands roughly one day *before* the actual birthday. A date of birth of `23-09-1971` evaluates to 54 and is declined; `22-09-1971` evaluates to 55. Arithmetic on calendar birthdays gets this wrong by a day. Use the stated values for boundary tests and a date comfortably clear of them otherwise.

### Seeded records

Eight instructions are seeded, giving every stage a starting example.

| Reference | Member | Fund | Option | Stage | Code |
|---|---|---|---|---|---|
| PN-70001 | Rosalind Fairweather | 240,000 | Guaranteed annuity | In payment | RP-A01 |
| PN-70002 | Desmond Achebe-Clark | 620,000 | Flexi-access drawdown | Referred | RP-R01 |
| PN-70003 | Marianne Okoro | 148,000 | Guaranteed annuity | In payment | RP-A01 |
| PN-70004 | Tobias Renshaw | 84,000 | Full encashment | Referred | RP-R06 |
| PN-70005 | Aisling Donnelly | 310,000 | Flexi-access drawdown | In payment | RP-A01 |
| PN-70006 | Nikolai Brandt | 9,200 | Flexi-access drawdown | Declined | RP-D03 |
| PN-70007 | Perpetua Nwosu | 455,000 | Flexi-access drawdown | Case | — |
| PN-70008 | Gwendolyn Achterberg | 192,000 | Guaranteed annuity | Awaiting authorisation | RP-A01 |

All seeded instructions record **Lena Fairbrook — Relationship Manager** as the submitting user, matching the account name exactly. A Compliance user can therefore decide any of them, and the identity check can be exercised on a record created during the run.

---

## 12. Known gaps and constraints

Properties of the application as built, recorded so that tests are written against actual behaviour rather than assumed behaviour.

- **Occupation is not collected here.** Unlike the motor insurance module, there is no occupation field.
- **Scheme type and plan number are collected and never used.** Neither affects the settlement or the decision.
- **Payment frequency is collected and never used.** Monthly, quarterly and annual produce the same figures; no instalment schedule is generated.
- **Employment status only matters for RP-R09.** It is otherwise inert.
- **The MPAA limit of 10,000 is defined but never applied.** Only the triggered flag is used, and only in combination with employment status.
- **Protected tax-free cash above 25 per cent both raises the maximum and triggers a referral.** A member with 30 per cent protected cash on a 240,000 fund may take 72,000 rather than 60,000, and is referred under RP-R05 for doing so.
- **Guarantee periods reduce the annuity rate but nothing models the guarantee itself.** No beneficiary or payment term is captured.
- **Enhanced underwriting terms are a single select value.** No medical evidence is captured or required.
- **The 90-day first-payment window is not re-checked after submission.** An instruction can sit in a queue past its own start date.
- **No payment reference exists until a human releases it.** An accepted instruction shows a stage and a settlement but no `PAY-` reference.
- **An approved referral does not require a further authorisation.** It moves straight to In payment. One check per instruction, never two.
- **The retirement review and payment authority is the Compliance role.** There is no separate reviewer account, and only one Compliance account exists.
- **The role gate renders nothing** for non-Compliance users. Assert absence, not a disabled state.
- **State is held in browser local storage** under the key `cavendish.platform.v1`, shared with the Private Wealth and Motor Insurance applications in this build. An older standalone wealth build remains deployed at the site root under `cavendish.cmo.v1`, so the two deployments never see each other's records even though they share an origin. Point a journey at `platform.html` explicitly; the bare site root reaches that other application.
- **Reset demo data is scoped to the application you are in.** Resetting from any `#/pensions` route restores the seeded retirement instructions, the member register, find requests, any draft and the case, payment and find counters, and **leaves wealth and motor insurance exactly as they are**. **The Applications launcher carries no reset control.** On `#/apps` the `reset-data` button is **hidden, not removed**: it stays in the DOM carrying the `hidden` attribute and `display:none`. A journey must therefore assert that it is **not visible**; asserting that the element does not exist will fail. No application is in scope on the launcher, so there is nothing for it to reset. A full reset of all three is on the sign in screen (`login-reset`), which is single-step with no confirmation modal. Any run that depends on a clean start should open the application it is about to exercise, then reset from inside it.
- **Reset demo data is a two-step control.** `reset-data` opens a confirmation modal; the reset happens on `reset-confirm` ("Reset now"). A single click changes nothing, and the modal names the application about to be reset, so it can be asserted.
- **Any reset clears a sign-in lockout.** The failed-attempt counters and the locked list are not demo records, so they are cleared whichever application you reset from. That keeps the sign-in message *"This account is locked after 5 failed sign-in attempts. Reset the demo data to unlock it."* true from everywhere.
- **Local storage belongs to one browser profile.** Two people demonstrating on their own machines, or in two different browsers on one machine, never affect each other's data; there is no server and no shared state. Two tabs of the same application in the same profile do share storage, and neither tab is told when the other writes.
- **The application is a single self-contained HTML file.** No server, no API, no network calls. The settlement and the decision run in the browser. There is nothing to stub and nothing to seed through an interface — every test starts from the UI.
- **The application must be served over HTTP to be automated remotely.** Run from a local `file://` path it is reachable only by a browser on that same machine.

---

## 13. The end-to-end journey

The canonical path from a blank instruction to benefits in payment, through the payment authorisation control.

**Preconditions:** application reachable over HTTP. Credentials supplied from environment variables.

| # | Step | Expected |
|---|---|---|
| 0 | Navigate to the application | Sign-in screen; nothing else reachable |
| 1 | Sign in as `lena.fairbrook` | Application launcher shown |
| 2 | Open the application with `app-pensions`, then reset demo data from inside it: `reset-data`, then `reset-confirm` | The launcher carries no reset control, so this cannot be done at step 1. Restores **pensions only**; wealth and motor insurance are left exactly as they are. Any draft cleared, still signed in. For a full reset of all three, use `login-reset` on the sign in screen before signing in |
| 3 | Activate `app-pensions`, or navigate to `#/pensions/case/new` directly | Wizard at Step 1 of 5, Back disabled |
| 4 | Complete the member and fund using the baseline | — |
| 5 | Continue | Step 2 of 5, Protection & safeguards |
| 6 | Set safeguarded benefits, guidance status, the three radio questions, tick the ScamSmart warning | — |
| 7 | Continue | Step 3 of 5, Retirement option |
| 8 | Select `Guaranteed annuity`; enter tax-free cash `60000`; set shape, guarantee and underwriting basis | The annuity panel appears on selecting the option |
| 9 | Continue | Step 4 of 5, Income modelling |
| 10 | Enter first payment `22-10-2026` and the growth assumption | `income-headline` shows **£8,730**; the settlement table itemises rate, tax bands and net |
| 11 | Continue | Step 5 of 5, Review & submit |
| 12 | Verify `case-decision` | Indicative outcome **ACCEPT (RP-A01)** |
| 13 | Tick both declarations and submit | Toast with the assigned `PN-` reference; detail opens at **Awaiting authorisation** with no payment reference |
| 14 | **Sign out, then sign in as `desmond.achebe`** | Launcher shown; user is Desmond Achebe, role Compliance |
| 15 | Open `#/pensions/authorisations` and select the record | Authorise and Reject actions visible |
| 16 | Activate `pa-authorise`, enter a note of at least 20 characters, confirm | Stage **In payment**; `PAY-` reference assigned; audit records the authorisation |

> **The journey is not complete at step 13.** A journey ending at submission verifies the wizard and the decision engine but never touches the control the process exists for. Step 14 is the point.

### Branch journeys worth deriving from this one

| Branch | Diverges at |
|---|---|
| Each of the fifteen decline and referral codes | Step 4, 6 or 8 — one field change each |
| Four-eyes on authorisation | Step 14 omitted — submit and authorise as the same user, blocked |
| Four-eyes on review | A referred instruction decided by its own submitter, blocked |
| Rejected at authorisation | Step 16 — reject with a reason of at least 20 characters |
| Referral approved | A referred instruction approved by a second user, straight to payment |
| Referral declined | Same, declined with a reason |
| Role gate | Step 15 as `lena.fairbrook` or `yuki.mori` — actions absent, notice shown |
| Each retirement option | Step 8 — four options, four panels |
| Guaranteed annuity rate applied | Step 6 and 8 — GAR held, annuity chosen |
| Drawdown sustainability | Step 8 and 10 — income and growth assumption |
| Vulnerability revealing control | Step 6 |
| Advice clears the safeguarded decline | Step 6 — the same case declines on guidance, accepts on advice |

---

## 14. Notes for automation

### Select options must match exactly

Every option string in this module is ASCII — no typographic dashes. Values typed with an ordinary keyboard hyphen match.

The placeholder is **`Select...`** with three full stops, not the single ellipsis character used in the wealth and motor insurance modules. It is a selectable option that fails validation.

Watch the wording of the growth assumptions and the underwriting basis — they read `4 per cent - central` and `Enhanced terms - declared medical factors`, with a spaced hyphen, not a colon or a dash.

### Dates are typed, not picked

`dd-mm-yyyy` into `input-memberDob` and `input-firstPaymentDate`. ISO is also accepted on entry and redisplays in `dd-mm-yyyy`. The record always stores ISO, which is why the boundary table gives both. The calendar buttons exist for human use; automation should type the value.

### Fields accept keystrokes

Every text and number field can be typed into character by character. Values entered by setting the value and firing a `change` event — the path most WebDriver-based tools take for a dropdown — are accepted on every control.

### Re-read the form after a revealing control

Two controls change the field set: the **vulnerability** radio on step 2, and the **retirement option** select on step 3. After changing either, re-read the form before interacting with further fields.

### The settlement block updates without a re-render

On step 4, changing a numeric field updates the settlement table in place rather than rebuilding the page. Read `income-headline`, `tax-total`, `net-total` and `depletion-age` after the change; do not expect the step to reload.

### Nothing is asynchronous

Settlement and adjudication are synchronous. No waits are required beyond the ordinary re-render, and a generated fixed pause is pure cost.

### Verify the step before entering data

A validation failure keeps the user on the current step. Check `case-card` shows the expected *Step n of 5* before interacting with fields on it.

### Recover from validation errors

Every mandatory field is reported at once on Continue, each message naming its field, in a `perror-{field}` element. After activating Continue, if the step counter has not advanced, read the messages, set the fields they name and activate Continue again rather than repeating the same action.

### Signing out returns to the launcher

A journey that signs out mid-run lands on the application picker when it signs back in, and must activate `app-pensions` or navigate to a pensions route. A journey that deep-links while signed out still resolves to the linked route.

---

## 15. Dashboards member matching

A second, independent process in the same module. A **find request** arrives from a pensions dashboard carrying a citizen's details. The scheme compares it against its member register and answers one of three ways.

| Outcome | What happens |
|---|---|
| **MATCH** | The member record is returned to the dashboard as view data |
| **POSSIBLE** | Nothing is returned. The request is held for a person to resolve |
| **NO MATCH** | Nothing is returned. The request is closed |

**The risk runs both ways, and that is the point of testing it.** Matching too loosely discloses one citizen's pension to another. Matching too tightly means a citizen never finds a pension they own. The application states this on the find request screen (`find-notice`).

### The find request

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| First name | Text | No | — | — |
| Surname | Text | **Yes** | Non-empty | `Surname is required on a find request.` |
| Previous surname | Text | No | — | — |
| Date of birth | **Text, `dd-mm-yyyy`, with calendar button** | **Yes** | Non-empty; age ≥ 16; age ≤ 110 | `Date of birth is required on a find request.` / `A find request cannot be made for someone under 16.` / `Enter a valid date of birth.` |
| National Insurance number | Text | No | If given, `^[A-Za-z]{2}[0-9]{6}[A-Za-z]?$` | `Enter a National Insurance number in the format QQ123456C, or leave it blank.` |
| Postcode | Text | No | — | — |
| Dashboard making the request | Select | **Yes** | Non-empty | `Select the dashboard the request came from.` |

**Dashboard sources** — MoneyHelper pensions dashboard · Bramwell Wealth dashboard · Northgate Money dashboard

> Only surname, date of birth and the dashboard are mandatory. The NI number and postcode are optional, and **which of them is supplied changes the outcome** — that is the heart of the rule set, not an edge case.

### Normalisation

Before comparison, names are lower-cased with all non-letters removed, and NI numbers and postcodes are upper-cased with all non-alphanumerics removed. So `O'Halloran`, `ohalloran` and `O Halloran` are the same surname, and `T12 X2FR` and `t12x2fr` are the same postcode. A requirement asserting that punctuation causes a mismatch is asserting the opposite of the behaviour.

### Surname agreement has three routes

A surname agrees if **any** of these hold:

1. It equals the member's current surname
2. It equals a **previous surname recorded by the scheme**
3. A **previous surname supplied on the request** equals the member's current surname

Route 2 and route 3 are what stop a married name defeating the match, and they are the most commonly missed branch.

### The matching rules

Eight rules, evaluated in **strict precedence order** against the best-agreeing candidate. Candidates are only considered at all if the surname or the NI number agrees.

| # | Code | Outcome | Condition |
|---|---|---|---|
| 1 | MM-N02 | NO MATCH | More than one record agrees equally on surname and date of birth with no NI number to separate them |
| 2 | MM-M01 | MATCH | Surname, date of birth and NI number all agree |
| 3 | MM-M02 | MATCH | Surname, date of birth and postcode agree, and **no NI number was supplied** |
| 4 | MM-P01 | POSSIBLE | NI number and date of birth agree; the surname does not |
| 5 | MM-P02 | POSSIBLE | Surname and date of birth agree; an NI number was supplied and does not agree |
| 6 | MM-P03 | POSSIBLE | Surname and NI number agree; the date of birth does not |
| 7 | MM-P04 | POSSIBLE | Surname and date of birth agree; nothing further could be confirmed |
| 8 | MM-N01 | NO MATCH | Nothing above matched |

Reason strings, quoted verbatim:

- `MM-M01` — *Match made. Surname, date of birth and National Insurance number all agree.*
- `MM-M02` — *Match made. Surname and date of birth agree and the address agrees; no National Insurance number was supplied.*
- `MM-P01` — *Possible match. The National Insurance number and date of birth agree but the surname does not; a recorded previous surname was not supplied.*
- `MM-P02` — *Possible match. The surname and date of birth agree but the National Insurance number does not.*
- `MM-P03` — *Possible match. The surname and National Insurance number agree but the date of birth does not.*
- `MM-P04` — *Possible match. Surname and date of birth agree but neither the National Insurance number nor the address could be confirmed.*
- `MM-N01` — *No match. No member record agrees on enough identifiers to be considered.*
- `MM-N02` — *No match. More than one member record agrees equally; the request cannot be resolved to a single member.*

### Why precedence matters here

- **MM-N02 is evaluated first.** Two members sharing a surname and date of birth produce no match even though each individually agrees on two identifiers. Ambiguity is a refusal, not a possible match.
- **MM-M02 requires the NI number to be absent, not merely wrong.** Supply a wrong NI number alongside an agreeing surname, date of birth and postcode and the result is MM-P02, not MM-M02. That distinction is easy to write a wrong requirement against.
- **MM-P01 through MM-P04 are mutually exclusive by construction** but describe genuinely different failures, and each needs its own requirement.

### The resolution control

A possible match is held at **Awaiting resolution** with nothing returned. `#/pensions/resolution` lists them.

- Only a **Compliance** user sees the resolve action. For others the buttons are not rendered and a notice appears (`resolve-role-notice`, `resolve-queue-role-notice`).
- The modal shows the **request and the member side by side** (`resolve-comparison`) so a human can judge.
- **Confirm** requires a note of at least 20 characters, releases the view data, and moves the outcome to MATCH.
- **Reject** requires a note of at least 20 characters, returns nothing, and moves the outcome to NO MATCH.

Unlike the retirement instruction controls, there is **no identity check here** — the person resolving is not the person who made the request, because the request came from a dashboard rather than from a member of staff. The role gate is the whole control.

### Element identification

`finds-new` · `find-card` · `find-notice` · `input-firstName` · `input-surname` · `input-previousSurname` · `input-dob` · `calendar-dob` · `input-nino` · `input-postcode` · `input-dashboard` · `ferror-{field}` · `find-run` · `find-clear` · `find-table` · `find-table-empty` · `find-row-{id}` · `open-{id}` · `find-outcome` · `agreement-table` · `candidate-member` · `find-audit` · `find-not-found` · `back-to-finds` · `resolution-table` · `resolution-table-empty` · `resolve-open` · `resolve-comparison` · `resolve-note` · `resolve-note-error` · `resolve-confirm` · `resolve-reject` · `resolution-banner` · `resolve-role-notice` · `resolve-queue-role-notice` · `member-table` · `member-row-{id}`

### The member register

Twelve seeded records, built so the rules bite.

| Reference | Name | Previous surname | Date of birth | NI number | Postcode |
|---|---|---|---|---|---|
| MB-80001 | Rosalind Fairweather | — | 18-04-1966 | QQ123456C | BS1 4TR |
| MB-80002 | Marianne Okoro | **Adeyemi** | 25-07-1959 | QQ234567D | L8 3SD |
| MB-80003 | Aisling Donnelly | — | 08-09-1964 | QQ345678A | BT9 6SN |
| MB-80004 | Tobias Renshaw | — | 19-01-1972 | QQ456789B | EX1 1EZ |
| MB-80005 | Perpetua Nwosu | — | 11-12-1966 | QQ567890C | G20 6BE |
| MB-80006 | Gwendolyn Achterberg | **Vandermeer** | 27-02-1963 | QQ678901D | CF11 9LJ |
| MB-80007 | Desmond Achebe-Clark | — | 02-11-1968 | QQ789012A | NE2 2EY |
| MB-80008 | Nikolai Brandt | — | 30-05-1975 | QQ890123B | CB2 1QA |
| MB-80009 | Eleanor Whitfield | — | 15-03-1970 | QQ901234C | YO30 7DH |
| MB-80010 | Edward Whitfield | — | 15-03-1970 | QQ012345D | YO30 7DH |
| MB-80011 | Sinead O'Halloran | — | 22-08-1961 | **none held** | T12 X2FR |
| MB-80012 | Callum Whitcombe | — | 04-06-1957 | QQ135791E | EH1 2JU |

Three records exist specifically to exercise the rules:

- **MB-80009 and MB-80010** share a surname, a date of birth and an address. They are the ambiguity case behind MM-N02.
- **MB-80011** has no NI number held by the scheme, so the postcode route is the only way to a match.
- **MB-80002 and MB-80006** carry previous surnames, covering both surname routes.

### Test data that produces each code

Verified against the application.

| Request | Outcome | Code |
|---|---|---|
| `Rosalind` `Fairweather` `18-04-1966` `QQ123456C` | MATCH | MM-M01 |
| `Sinead` `O'Halloran` `22-08-1961` postcode `T12 X2FR`, no NI number | MATCH | MM-M02 |
| `Rosalind` `Fairbrook` `18-04-1966` `QQ123456C` | POSSIBLE | MM-P01 |
| `Rosalind` `Fairweather` `18-04-1966` `QQ999999Z` | POSSIBLE | MM-P02 |
| `Rosalind` `Fairweather` `19-04-1966` `QQ123456C` | POSSIBLE | MM-P03 |
| `Rosalind` `Fairweather` `18-04-1966`, no NI number, no postcode | POSSIBLE | MM-P04 |
| `Zebedee` `Marchetti` `01-01-1980` `ZZ111111A` | NO MATCH | MM-N01 |
| `Whitfield` `15-03-1970`, no NI number | NO MATCH | MM-N02 |

Two further cases worth covering:

| Request | Outcome | Why |
|---|---|---|
| `ohalloran` `22-08-1961` postcode `t12x2fr` | MM-M02 | Normalisation — punctuation and case are stripped |
| `Marianne` `Adeyemi` `25-07-1959` `QQ234567D` | MM-M01 | The scheme's recorded previous surname resolves it |
| `Gwendolyn` `Vandermeer` `27-02-1963` `QQ678901D` | MM-M01 | A previous surname supplied on the request resolves it |

### Known gaps and constraints

- **There is no identity check on resolution**, only a role gate, for the reason given above.
- **Address is compared on postcode alone.** The street address is held and displayed but is not part of the comparison.
- **First name is scored but never decisive.** It contributes to ranking candidates and appears in the agreement panel, but no rule requires it.
- **The find request is not persisted as a draft.** Navigating away clears it; only submitted requests are stored.
- **There is no view-data payload.** A match reports that data would be returned and shows the member record; nothing is serialised or transmitted.
- **Find requests are never purged**, and there is no retention window.

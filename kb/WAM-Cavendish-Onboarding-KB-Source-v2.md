# Cavendish Private Wealth — New Client Onboarding

**System:** Cavendish Client Management & Onboarding (CMO)
**Build:** 2026.09.27
**Release:** 2.1
**Module:** Client management → Onboarding → New client
**Application URL:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/platform.html`
**Document type:** End-to-end process specification
**Status:** Current — describes the application as built

> **Scope of this document.** This describes the **Private Wealth application** — sign-in, the six-stage client onboarding wizard and compliance approval — as it exists inside `platform.html`, the three-application build. Signing in leads to the **Applications launcher**, from which `app-wam` opens Private Wealth. Motor insurance and pensions live in the same file and are documented separately in `Cavendish-Motor-Insurance-KB-Source.md` and `Cavendish-Pensions-KB-Source.md`.
>
> **A second, older build of the wealth application is also deployed**, at the site root `https://kishorespotqa.github.io/virtuoso-wealth-demo/`. It is build 2026.09.11, stores its state under `cavendish.cmo.v1`, has no launcher, and its reset is not scoped. **Demonstrations and automation target `platform.html`**, the URL above; that older deployment is not covered by this document. The two share an origin but never share data, because the storage keys differ.

---

## 0. Release 2.1

This document describes Private Wealth with **New features enabled**. Release 2.1 adds a vulnerability assessment to KYC, a sustainability preference to risk and suitability, and **reprices the fee schedule**.

| # | Change | Kind |
|---|---|---|
| 1 | Vulnerability assessment — indicator plus a support plan, and a raised screening verdict | New fields, **new screening rule** |
| 2 | Sustainability preference — the five UK SDR labels | New field |
| 3 | Fee schedule — entry tiers reduced and a fifth tier added above 25,000,000 | **Changes an existing calculation** |

### The release control

A **New features** control in the sidebar, `data-testid="release-toggle"`, scoped to Private Wealth and **off by default**. A banner with `data-testid="release-banner"` shows while it is on, and each new field carries a badge with `data-testid="rel-new"`.

A journey pins the release on its opening navigation rather than clicking the control, which is a toggle and therefore only correct from a known starting state:

```
platform.html?release=on#/onboarding/new     release on
platform.html?release=off#/onboarding/new    release off
```

The setter is absolute and idempotent. Any value other than `on` or `off` is ignored.

> **The URL setter is estate-wide.** `?release=on` turns the release on for **every** application in this shell that has one, not only the one you navigate to. The sidebar control is per application; the URL setter is not. A journey that needs one application on its new release and another on its previous one cannot express that with this flag, and should use the sidebar control manually before the run.

> **A journey must never click `release-toggle` or `reset-data`.** Both change the rules underneath a run.

### A client keeps the schedule it was onboarded under

Each submitted client is stamped with its release and priced on that schedule permanently. Repricing does not change the fee on a mandate that is already active, and the seeded client records keep the previous schedule whatever the toggle does.

**Download KB source** in the sidebar follows the release: with it on the control reads **Download KB source v2** and serves this file.

---

## 1. What this process does

Cavendish CMO onboards a new private wealth client from first contact to an activated investment mandate. The process is a **six-stage wizard** completed by a Relationship Manager, followed by a **four-eyes compliance approval** that activates or rejects the mandate.

Nothing is submitted to compliance until every mandatory item is complete. The wizard permits free navigation between completed stages, but forward progress past a stage requires that stage to validate.

### Actors

| Actor | Name in the system | Username | Role | What they can do |
|---|---|---|---|---|
| Adviser | Lena Fairbrook — Relationship Manager | `lena.fairbrook` | Adviser | Complete the wizard, run screening, submit for approval |
| Compliance | Desmond Achebe — Compliance Officer | `desmond.achebe` | Compliance | Approve or reject a submitted record |
| Operations | Yuki Mori — Client Operations | `yuki.mori` | Operations | View records |

The acting user is established by **signing in** — see section 2. There is no role selector in the application. **This matters for testing:** the four-eyes control compares the approving user against the submitting user, so any end-to-end journey that reaches an activated mandate must sign out and sign back in as a different user.

### Navigation

| Route | Screen | Notes |
|---|---|---|
| `#/dashboard` | Dashboard | AUM, in-flight count, stage distribution, recent activity |
| `#/clients` | Clients | Searchable, filterable, sortable, paged list |
| `#/clients/{id}` | Client detail | Record, documents, audit trail; approve/reject actions |
| `#/onboarding` | Onboarding pipeline | Records not yet Active or Rejected |
| `#/onboarding/new` | **New client wizard** | The process described here |
| `#/compliance` | Compliance queue | Records at *Awaiting approval* |
| `#/portfolios`, `#/billing`, `#/reporting` | Not implemented | Marked "soon" and disabled |

### Record lifecycle

```
Draft ──submit──> Awaiting approval ──approve──> Active
                                    ──reject───> Rejected
```

Intermediate stages `KYC in progress`, `Risk profiled` and `Documents pending` exist in the data model and appear on seeded records, but the wizard itself moves a record directly from **Draft** to **Awaiting approval** on submission.

---

## 2. Sign in

The application is gated by a sign-in screen. Nothing else is reachable until a user authenticates: a direct link to any route shows the sign-in screen first, then resolves to the requested route once the user is in.

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Username | Text | Yes | Non-empty. Trimmed and matched case-insensitively. | `Enter your username.` |
| Password | Password | Yes | Non-empty. Matched exactly, case-sensitive. | `Enter your password.` |

Empty-field validation is evaluated **before** any credential check. Submitting the form with either field blank produces the field-level message only — no sign-in attempt is recorded and no failure banner appears. This distinction matters: a blank submission does not count towards the lockout counter.

### Credentials

The three users in section 1 share the password `Cavendish#2026`. The sign-in screen displays a **Demo credentials** panel listing all three usernames and the password, so the application is self-documenting.

For automation, take the username and password from environment variables rather than the on-screen hint, so a credential change is a configuration change rather than a journey edit.

### Failure behaviour

| Condition | Result |
|---|---|
| Wrong password for a valid username | Banner: `The username or password is incorrect.` |
| Unknown username | **The same banner, verbatim.** |
| Either failure | The password field is cleared; the username is retained |

The message is deliberately identical in both cases and does not reveal which field was wrong, nor whether the username exists. A test asserting a different message for an unknown user is asserting behaviour the application does not have — and should not have.

### Lockout

A username is locked after **5 consecutive failed attempts**. The count is kept per username, including for usernames that do not exist.

| Behaviour | Detail |
|---|---|
| Attempts 1–4 | Generic failure banner. **No warning that a lockout is approaching.** |
| Attempt 5 | Banner: `This account is locked after 5 failed sign-in attempts. Reset the demo data to unlock it.` |
| While locked | Even the correct password is refused, and the lockout banner is shown again |
| Clearing the lock | **Reset demo data only.** The lockout does not expire on a timer. |

Because the lockout has no time-based expiry it is fully deterministic — a considerable advantage for automated testing, and the reason the reset control is duplicated on the sign-in screen where the top bar is not available.

A successful sign-in clears that username's attempt counter.

### Session

| Behaviour | Detail |
|---|---|
| Signing in | Records the user and a timestamp; the top bar shows the name and role |
| Reload | The session survives a page reload |
| Signing out | Returns to the sign-in screen and clears both credential fields |
| Signing back in | Lands on the **Applications launcher** at `#/apps`, never on the screen where the previous session ended. Open Private Wealth again with `app-wam`, or navigate straight to a `#/dashboard`, `#/clients`, `#/onboarding` or `#/compliance` route |
| Draft on sign-out | **An in-progress onboarding draft is preserved.** Signing back in returns to it with every captured value intact. |
| Reset demo data while signed in | Restores the seed data **for the application you are in** and keeps the user signed in. From the launcher it restores all three |
| Reset demo data from the sign-in screen | Restores the seed data for **all three applications** and clears all lockouts. This control is single-step, with no confirmation modal |

There is **no inactivity timeout**. See section 12.

---

## 3. Stage 1 — Prospect details

### Client classification

A radio set with four values: **Individual**, **Joint**, **Trust**, **Corporate**. Default is Individual. The classification changes which fields are displayed.

| Classification | Name fields shown | Additional mandatory fields |
|---|---|---|
| Individual | First name, Last name, Date of birth | — |
| Joint | First name, Last name, Date of birth | At least one signatory at stage 5 |
| Trust | Registered entity name | Trustee (Settlor optional) |
| Corporate | Registered entity name | Company registration number (LEI optional) |

For Trust and Corporate the "Nationality" label changes to **"Country of incorporation"**. The field is the same field.

### Fields

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Client classification | Radio | Yes (defaulted) | — | — |
| Registered entity name | Text | Trust/Corporate only | Non-empty | `Registered entity name is required.` |
| Company registration number | Text | Corporate only | Non-empty | `Company registration number is required.` |
| Legal entity identifier (LEI) | Text | No | Hint: 20 characters, required before any MiFID-reportable transaction | — |
| Trustee | Text | Trust only | Non-empty | `Trustee name is required.` |
| Settlor | Text | No | — | — |
| First name | Text | Individual/Joint | Non-empty | `First name is required.` |
| Last name | Text | Individual/Joint | Non-empty | `Last name is required.` |
| Date of birth | **Text, `dd-mm-yyyy`, with calendar button** | Individual/Joint | Non-empty; age ≥ 18; age ≤ 120 | `Date of birth is required.` / `Client must be 18 or over to hold a mandate.` / `Enter a valid date of birth.` |
| Nationality / Country of incorporation | Select | Yes | Non-empty | `Select a country.` |
| Tax residency | Select | Yes | Non-empty | `Tax residency is required.` |
| Email | Email | Yes | Non-empty; pattern `^[^\s@]+@[^\s@]+\.[^\s@]{2,}$` | `Email is required.` / `Enter a valid email address, e.g. name@example.com.` |
| Telephone | Tel | Yes | Non-empty; at least 8 digits after stripping non-digits | `Telephone is required.` / `Enter a telephone number with at least 8 digits.` |
| Address line | Text | Yes | Non-empty | `Address line is required.` |
| City | Text | Yes | Non-empty | `City is required.` |
| Postcode / ZIP | Text | **No** | — | — |
| Country of residence | Select | Yes | Non-empty | `Country of residence is required.` |
| Relationship manager | Select | Yes | Non-empty; defaults to the first adviser | `Assign a relationship manager.` |
| Introduced by | Text | No | Free text — referral source or intermediary | — |

### Country list

United Kingdom · Ireland · United States · Canada · Singapore · Hong Kong SAR · Japan · Germany · Switzerland · United Arab Emirates · South Africa · Nigeria · India · Brazil · Cayman Islands · Panama · Russian Federation · Islamic Republic of Iran

**Four of these are high-risk jurisdictions** and change the screening outcome at stage 2: Russian Federation, Islamic Republic of Iran, Panama, Cayman Islands.

### Relationship managers

A. Whitcombe · R. Nakamura · P. Oyelaran · S. Da Costa · M. Lindqvist

---

## 4. Stage 2 — KYC & AML

### Identity document

| Field | Control | Mandatory | Validation | Exact error message |
|---|---|---|---|---|
| Identity document type | Select | Yes | Passport / National identity card / Driving licence / Certificate of incorporation | `Select a document type.` |
| Document number | Text | Yes | Non-empty | `Document number is required.` |
| Expiry date | Date | Yes | Non-empty; must not be in the past; must be at least 90 days away | `Expiry date is required.` / `This document has expired.` / `Document expires within 3 months — obtain a renewed copy.` |

**The 90-day rule is a distinct failure from expiry.** A document expiring in 60 days is rejected even though it is still valid. This is the single most missed test case on this stage.

### Politically exposed person

A mandatory Yes/No radio: *"Is the client, or a close associate, a politically exposed person?"*

- No declaration → `A PEP declaration is required.`
- **Yes** reveals a mandatory free-text field, *"Position held, jurisdiction and relationship"* → `Describe the position, jurisdiction and relationship.` if left empty.
- **Yes** also displays a warning banner: *"Enhanced due diligence triggered — PEP relationships require senior management approval and annual review regardless of the screening outcome."*

### Source of wealth and funds

| Field | Control | Mandatory | Options | Exact error message |
|---|---|---|---|---|
| Source of wealth | Select | Yes | Employment income / Business sale or disposal / Inheritance / Investment returns / Property sale / Professional practice / Other | `Select a source of wealth.` |
| Source of funds for this mandate | Select | Yes | Transfer from another institution / Bank deposit / Sale proceeds / In-specie securities transfer / Trust distribution | `Select a source of funds.` |
| Expected annual inflow | Number | Yes | Non-empty; not negative; step 10,000 | `Expected annual inflow is required.` / `Enter a positive amount.` |

### Screening

Screening runs against the UK HMT and OFAC consolidated lists, a PEP register and an adverse media index. **It must be run before the stage can be left** → `Run sanctions and PEP screening before continuing.`

The verdict is deterministic and evaluated in strict precedence order:

| # | Condition | Result | Severity | Detail text |
|---|---|---|---|---|
| 1 | Client name contains `petrov`, `al-mansour`, `okonkwo-sanction` or `delacroix-holdings` (case-insensitive substring of first + last + entity name) | **Potential match** | risk | Name similarity against OFAC/UK HMT consolidated list entry "…". Requires level 2 adjudication. |
| 2 | PEP declared Yes | **PEP confirmed** | warn | Self-declared politically exposed person. Enhanced due diligence and senior sign-off required. |
| 3 | Nationality **or** tax residency is a high-risk jurisdiction | **Elevated risk** | warn | Nationality or tax residency falls within a high-risk jurisdiction. Enhanced due diligence applies. |
| 4 | Otherwise | **Clear** | ok | No adverse media, sanctions or PEP matches returned across 4 lists. |

Precedence is absolute: a watchlist name returns *Potential match* even when the client is also a declared PEP in a high-risk jurisdiction.

### Adjudication of a potential match

A **Potential match** (severity `risk`) blocks progress until both of the following are supplied:

1. A ticked confirmation: *"I have adjudicated this match as a false positive and recorded the rationale below."* → `A potential match must be adjudicated before continuing.` if unticked.
2. An adjudication rationale of **at least 20 characters** → `Provide an adjudication rationale of at least 20 characters.`

A *warn* severity (PEP confirmed, Elevated risk) does **not** block progress and requires no adjudication.

---

### Vulnerability assessment — release 2.1

| Field | Control | `data-testid` | Mandatory | Exact error message |
|---|---|---|---|---|
| Vulnerability indicators | Select | `input-vulnerability` | Yes | `Record a vulnerability indicator, or None declared.` |
| Support plan | Text | `input-vulnerabilitySupport` | Only when an indicator other than None declared is recorded | `A support plan is required when a vulnerability indicator is recorded.` |

Options: `None declared` · `Health` · `Life event` · `Financial resilience` · `Capability`.

The support-plan field is **not in the DOM** until an indicator other than `None declared` is chosen. Its container carries `data-testid="vulnerability-support-field"`. Selecting an indicator redraws the step, so a journey waits for that container rather than assuming the field is present.

**The screening rule.** A recorded indicator raises the screening verdict to **Enhanced support required**, severity `warn`, with the detail naming the indicator. It sits below a sanctions match and below a confirmed PEP in precedence, and above a clear result:

| Condition | Verdict |
|---|---|
| Watchlist name match | Potential match |
| PEP declared | PEP confirmed |
| High-risk jurisdiction | Elevated risk |
| **Vulnerability indicator recorded** | **Enhanced support required** |
| None of the above | Clear |

A client record stamped with the previous release is unaffected by this rule even while 2.1 is on.

## 5. Stage 3 — Risk & suitability

### The questionnaire

Eight questions, each with four options scored 1 to 4 in the order displayed. All eight are mandatory → `Select an answer.` per unanswered question.

| # | Question | Option 1 (score 1) → Option 4 (score 4) |
|---|---|---|
| 1 | What is the primary purpose of this portfolio? | Preserve capital in nominal terms → Generate a predictable income → Grow capital steadily over a cycle → Maximise long-term growth, volatility accepted |
| 2 | Over what period do you expect to draw materially on these assets? | Under 3 years → 3 to 5 years → 5 to 10 years → More than 10 years |
| 3 | A 20% fall in portfolio value over six months would lead you to: | Sell out entirely → Move to cash in part → Hold the position → Invest further at lower prices |
| 4 | What proportion of your total net worth do these assets represent? | Over 75% → 50–75% → 25–50% → Under 25% |
| 5 | How would you describe your investment experience? | None beyond deposits → Funds and bonds only → Direct equities and ETFs → Derivatives, private markets, structured products |
| 6 | How dependent is your lifestyle on income from this portfolio? | Entirely dependent → Substantially dependent → Partly dependent → Not dependent |
| 7 | Which statement best reflects your view of illiquid holdings? | Unacceptable — I need daily access → Acceptable for a small share → Comfortable up to a quarter of assets → Comfortable with substantial lock-ups |
| 8 | Expected additional contributions over the next three years: | None — likely withdrawals → None → Regular modest contributions → Significant further contributions |

### Scoring and bands

Score is the **unweighted sum** of the eight answers. Range **8 to 32**. The score and indicated band update live as answers are given, and a progress bar shows score as a percentage of 32.

| Score | Band | Displayed growth-asset range |
|---|---|---|
| 8 – 13 | **Conservative** | 15–30% growth assets |
| 14 – 20 | **Balanced** | 40–60% growth assets |
| 21 – 26 | **Growth** | 65–80% growth assets |
| 27 – 32 | **Aggressive** | 85–100% growth assets |

The on-screen legend reads: `Conservative ≤13 · Balanced ≤20 · Growth ≤26 · Aggressive 27+`

### Objective, horizon, liquidity, ESG

| Field | Control | Mandatory | Options | Exact error message |
|---|---|---|---|---|
| Stated investment objective | Select | Yes | Capital preservation / Income / Balanced growth / Capital growth / Aggressive growth | `Select an investment objective.` |
| Time horizon | Select | Yes | Under 3 years / 3-5 years / 5-10 years / Over 10 years | `Select a time horizon.` |
| Liquidity requirement | Select | Yes | Immediate access to all assets / Up to 10% within one month / Up to 25% within one quarter / No near-term requirement | `Select a liquidity requirement.` |
| ESG preference | Select | **No** | No preference (default) / Exclusions only / Article 8 - promotes ESG characteristics / Article 9 - sustainable objective | — |

### Suitability cross-check

Each objective implies a **risk floor**:

| Objective | Implied band |
|---|---|
| Capital preservation | Conservative |
| Income | Conservative |
| Balanced growth | Balanced |
| Capital growth | Growth |
| Aggressive growth | Aggressive |

Bands are indexed 0–3 (Conservative, Balanced, Growth, Aggressive). The gap is `index(questionnaire band) − index(objective floor)`.

| Gap | Outcome |
|---|---|
| ≥ +2 | **Suitability mismatch** — "The questionnaire places this client in the {band} band, but the stated objective ({objective}) implies {floor}. A documented rationale is required before the mandate is activated." |
| −1, 0, +1 | **Objective consistent with risk capacity** — "No suitability exception raised." |
| ≤ −2 | **Suitability mismatch** — "The stated objective ({objective}) is more aggressive than the {band} risk capacity indicated by the questionnaire." |

The banner appears on this stage once all eight questions are answered and an objective is chosen, and again in the declaration panel at stage 6. **It is advisory — it does not block progress or submission.**

---

### Sustainability preference — release 2.1

| Field | Control | `data-testid` | Mandatory | Exact error message |
|---|---|---|---|---|
| Sustainability preference | Select | `input-sustainability` | Yes | `Select a sustainability preference.` |

Options, the five UK SDR labels: `No preference` · `Sustainability Focus` · `Sustainability Improvers` · `Sustainability Impact` · `Sustainability Mixed Goals`.

It sits beside the existing ESG field, which is unchanged and remains optional. A stated preference restricts the model portfolios that may be recommended; the hint under the field says so.

Validated on Continue from the risk and suitability stage, alongside objective, horizon and liquidity.

## 6. Stage 4 — Documents

Five document slots. Accepted formats PDF, JPG, PNG, up to 10 MB. Each slot offers **Choose file** (real upload) or **Attach sample** (attaches a synthetic file).

| Document | Note shown | Mandatory |
|---|---|---|
| Proof of identity | Passport or national ID, in date | Always |
| Proof of address | Dated within 3 months | Always |
| Source of wealth evidence | Sale agreement, payslips, accounts | Always |
| Tax residency form (W-8BEN / CRS) | Required for non-UK tax residents | **Conditional** |
| Signed investment management agreement | Countersigned copy | Never |

**The conditional rule:** the tax residency form becomes mandatory when tax residency is set, country of residence is set, and **the two differ**. Its badge changes from "optional" to "mandatory" and it is enforced on Continue.

Any missing mandatory document yields `{Document name} is mandatory.` against that slot.

A received document shows a "Received" pill and a **Remove** action.

---

## 7. Stage 5 — Accounts & mandate

| Field | Control | Mandatory | Options / rule | Exact error message |
|---|---|---|---|---|
| Service model | Select | Yes | Discretionary / Advisory / Execution-only | `Select a service model.` |
| Base currency | Select | No | GBP (default) / USD / EUR / CHF / SGD / AED | — |
| Initial funding amount | Number | Yes | Non-empty; **≥ 250,000** in base currency; step 50,000 | `Initial funding amount is required.` / `Minimum mandate size is 250,000 in the base currency.` |
| Custodian | Select | Yes | Pershing Nexus / Northern Vault Trust / Interactive Custody Ltd / Lombard Fiduciary SA | `Select a custodian.` |
| Model portfolio | Select | Yes | Six models, below | `Select a model portfolio.` |

The service-model field carries the hint: *"Execution-only mandates bypass suitability but require an appropriateness warning."* **The application does not implement that warning** — see section 12.

### Model portfolios

| ID | Name | Risk band | Equity |
|---|---|---|---|
| MP-01 | Capital Defence | Conservative | 20% |
| MP-02 | Income & Stability | Conservative | 35% |
| MP-03 | Balanced Multi-Asset | Balanced | 55% |
| MP-04 | Global Growth | Growth | 75% |
| MP-05 | Concentrated Equity Alpha | Aggressive | 95% |
| MP-06 | Sustainable Balanced (Art.8) | Balanced | 58% |

The **visible option text** in the Model portfolio dropdown is `{name} - {risk band} ({equity}% equity)`, for example `Balanced Multi-Asset - Balanced (55% equity)`. Select by that exact string. The stored value is the ID (`MP-03`).

### Portfolio / risk band alignment

Once the questionnaire is complete and a portfolio is selected, the application compares the portfolio's risk band against the client's:

- **Exact match** → hint: *"Aligned with the client's {band} risk band."*
- **Any difference, in either direction** → warning banner: *"Portfolio outside risk band — {portfolio} is a {portfolio band} strategy; the client profiles as {client band}. Compliance will require a documented exception."*

Note this is stricter than the suitability cross-check at stage 3: **any** mismatch warns, not only a gap of two or more. Like the suitability flag, it is advisory and does not block.

### Authorised signatories

An add/remove table. Each row: **Name** (text), **Capacity** (Joint holder / Director / Trustee / Power of attorney / Authorised representative), **Authority** (Sole / Joint with any other / Joint with all).

| Rule | Error message |
|---|---|
| A **Joint** classification requires at least one signatory with both a name and a capacity | `A joint mandate requires at least one additional signatory with a capacity.` |
| Any added signatory must have a name and a capacity | `Signatory {n} needs a name and a capacity.` |

Authority is not validated. For a sole individual mandate the panel reads *"None recorded. Optional for a sole individual mandate."*

### Indicative annual fee

Calculated live from the funding amount on a **marginal tiered** basis:

| Tranche | Rate |
|---|---|
| Up to 1,000,000 | 85 bps |
| 1,000,000 – 5,000,000 | 70 bps |
| 5,000,000 – 10,000,000 | 55 bps |
| 10,000,000 – 25,000,000 | 40 bps |
| Above 25,000,000 | 30 bps |

> **Release 2.1 repriced this schedule.** The entry tier fell from 95 to 85 bps, the second from 75 to 70, and a fifth tier was added above 25,000,000 at 30 bps. The two middle tiers are unchanged. This is the one change to an existing calculation rather than an addition, so **every fee figure in this document differs from the previous release**.

Each tranche is shown as its own row with amount, rate and fee, followed by a total and an effective rate in bps. With no funding amount entered the panel reads *"Enter an initial funding amount to calculate the indicative fee."* The note states the fee excludes custody, transaction and third-party fund charges.

**Worked example.** A funding amount of 2,000,000 GBP produces 1,000,000 @ 85 bps = 8,500 and 1,000,000 @ 70 bps = 7,000, total **15,500 GBP**, effective rate 77.5 bps.

**The schedule across the tiers**

| Funding amount | Fee | Effective rate |
|---|---|---|
| 500,000 | 4,250 | 85.0 bps |
| 1,000,000 | 8,500 | 85.0 bps |
| 2,000,000 | 15,500 | 77.5 bps |
| 5,000,000 | 36,500 | 73.0 bps |
| 10,000,000 | 64,000 | 64.0 bps |
| 25,000,000 | 124,000 | 49.6 bps |
| 30,000,000 | 139,000 | 46.3 bps |

At 30,000,000 the tranches are 1,000,000 @ 85 · 4,000,000 @ 70 · 5,000,000 @ 55 · 15,000,000 @ 40 · 5,000,000 @ 30. On the previous release the same amount produced four tranches and a fee of 147,000.

**An active client is not repriced.** A mandate onboarded under the previous release keeps that schedule, so the fee shown on its record does not move when the release is turned on.

---

## 8. Stage 6 — Review & submit

### Completeness checklist

Seven checks, each Complete or Outstanding with a detail line:

| Check | Passes when |
|---|---|
| Prospect details | All stage 1 mandatory fields present for the chosen classification |
| KYC and AML | Document type, number, expiry, PEP declaration, source of wealth and source of funds all present |
| Screening | Screening has been run **and** either severity is not `risk`, or it is adjudicated with a rationale of ≥ 20 characters |
| Risk questionnaire | All eight answered **and** objective, horizon and liquidity set |
| Mandatory documents | Every mandatory document received, including the conditional tax form |
| Mandate and accounts | Service model, custodian, model portfolio all set **and** funding ≥ 250,000 |
| Adviser declaration | The declaration checkbox is ticked |

A banner above the checklist reads either **"{n} item(s) outstanding — The record cannot be submitted until every mandatory item is complete"** or **"Ready for submission — All mandatory items are complete. Submitting routes the record to the compliance queue for four-eyes approval."**

### Summary panels

Four read-only panels: **Prospect** (name, classification, nationality, tax residency, email, relationship manager), **KYC & suitability** (screening result, PEP, risk band with score out of 32, objective, documents received count), **Mandate** (service, model portfolio, initial funding, custodian, indicative fee), and **Declaration**.

### Declaration and submission

A mandatory checkbox: *"I confirm the information recorded has been verified against original or certified documents and that the recommended mandate is suitable for this client."*

| Condition | Error message |
|---|---|
| Declaration unticked | `You must confirm the declaration before submitting.` |
| Any checklist item outstanding | `Resolve the outstanding checklist items before submitting.` |

The submit control reads **"Submit for compliance approval"**. On failure a toast appears: *"Submission blocked — see the highlighted items."*

On success the application:

1. Assigns a client ID in the format `CW-#####`, sequentially
2. Stores the risk score and risk category
3. Sets stage to **Awaiting approval**
4. Records the submitting actor and a submission timestamp
5. Sets AUM to 0 (it is set to the funding amount only on approval)
6. Writes an audit entry: *"Onboarding record submitted for compliance approval. Screening outcome: {result}."*
7. Clears the draft and resets the wizard to step 1
8. Shows a toast: *"{name} submitted as {id} — now with compliance."*
9. Navigates to the client detail page

---

## 9. Compliance approval — the four-eyes control

A submitted record appears in the **Compliance queue** and on the client detail page with Approve and Reject actions.

### The role gate

**Approve and Reject are rendered only for a user whose role is Compliance.** An Adviser or Operations user viewing the same record sees no approval controls at all — they are absent from the DOM, not present and disabled. On the Compliance queue, a non-Compliance user instead sees a `role-notice` banner: *"Read-only for this role — You are acting as {role}. Sign in as the compliance officer to adjudicate these records."*

On the queue itself the per-row `approve-{id}` button is rendered for every role but **disabled** for non-Compliance users. So the two surfaces behave differently: absent on the detail page, disabled on the queue. A test asserting presence-then-disabled on the detail page will fail.

The detail-page controls carry a second condition: the record must be at stage **Awaiting approval**. An Active or Rejected record shows no approval controls even to a Compliance user.

This means the four-eyes separation is enforced at **two independent levels**, and both are worth covering:

| Level | Control |
|---|---|
| Role | Only a Compliance user ever sees the approval actions |
| Identity | A Compliance user still cannot approve a record they submitted themselves |

### The control

**The submitting user cannot approve the same record.** An attempt produces the toast: *"Four-eyes control: the submitting user cannot approve the same record."* and nothing changes.

This is compared on actor **name**, so the check is defeated only by switching actor in the top-bar selector — which is the intended path.

### Approval

Opens a modal: *"Approving {name} ({id}) activates the {service model} mandate and sets a periodic KYC review 12 months out."* Where the screening severity is not `ok`, an additional banner asks the approver to confirm the adjudication rationale has been reviewed and retained. An optional approval note is retained on the audit trail.

On confirmation:

- Stage → **Active**
- Approving actor recorded
- KYC completion date set to today; **KYC review due exactly 12 months later**
- AUM set to the initial funding amount
- Audit entry: *"Onboarding approved; mandate activated.{ Note: …}"*
- Toast: *"{id} approved — mandate active."*

### Rejection

Opens a modal requiring a reason of **at least 15 characters** → `A reason of at least 15 characters is required.` (shown inline, the modal does not close).

On confirmation: stage → **Rejected**, reason retained for regulatory reporting, approver cleared, audit entry *"Onboarding rejected — {reason}"*.

---

## 10. Element identification

Every interactive element carries a stable `data-testid`. **Prefer these over label text or position** — they are the intended automation hook and do not change with classification branching.

### Global shell — present on every screen inside an application

| Element | `data-testid` | Notes |
|---|---|---|
| Main navigation | `main-nav` | |
| Navigation item | `nav-dashboard`, `nav-clients`, `nav-onboarding-pipeline`, `nav-new-client`, `nav-compliance-queue` | Slugged from the label |
| Signed-in user and role | `current-user` | Name and role of the authenticated user |
| Sign out | `sign-out` | Returns to the sign-in screen |
| Reset demo data | `reset-data` | Opens a confirmation modal. **Hidden on the Applications launcher** — present in the DOM but not visible. See section 12 |
| Reset confirmation | `reset-confirm` | The "Reset now" button inside that modal |
| Current view container | `view` | |
| Breadcrumb | `breadcrumb` | |
| Toast container | `toasts` | |
| Role notice | `role-notice` | |
| Demonstration disclosure, sign-in card | `demo-disclosure` | States that the firm is fictional and the data synthetic |
| Demonstration disclosure, sidebar | `sidebar-disclosure` | `Demo · synthetic data` |

### Sign-in screen

| Element | `data-testid` |
|---|---|
| Sign-in view container | `login-view` |
| Form (submits on Enter) | `login-form` |
| Username | `input-username` |
| Password | `input-password` |
| Sign in | `login-submit` |
| Username field error | `error-username` |
| Password field error | `error-password` |
| Generic failure banner | `login-error` |
| Lockout banner | `login-locked` |
| Demo credentials panel | `demo-credentials` |
| Reset demo data (sign-in screen) | `login-reset` |

The form submits on Enter as well as on the button, so a journey may use either.

**Resetting demo data is two steps, not one.** `reset-data` opens a modal — *"This restores the 12 seeded client records, clears any draft and returns every stage counter to its original state."* — and `reset-confirm` performs the reset. A journey that clicks only the first control has not reset anything.

**Routing responds to `hashchange`.** Navigating directly to a hash URL works from cold, and changing the hash during a session re-renders the view and clears any pending validation errors. Either navigation style is safe.

### Dashboard

`tile-aum` · `tile-inflight` · `tile-reviews` · `tile-flags` · `card-pipeline` · `card-approvals` · `card-activity` · `stage-{slugged stage}` · `funnel-{slugged stage}` · `dash-new-client`

`dash-new-client` is the **Onboard a new client** button on the dashboard — a second route into the wizard alongside the `nav-new-client` sidebar item and the direct hash URL.

### Clients list, client detail and compliance queue

**List:** `clients-table` · `clients-new` · `clients-empty` · `client-row-{id}` · `filter-search` · `filter-stage` · `filter-risk` · `filter-adviser` · `filter-clear` · `sort-{column}`

**Detail:** `client-name` · `client-not-found` · `tab-{name}` · `kv-identification` · `documents-list` · `doc-{docId}` · `audit-trail` · `screening-result` · `client-approve` · `client-reject` · `back-to-clients`

**Compliance queue:** `compliance-table` · `compliance-row-{id}` · `compliance-empty` · `approve-{id}`

### Wizard frame

| Element | `data-testid` |
|---|---|
| Stepper | `wizard-stepper` |
| Step tab (per stage) | `step-prospect`, `step-kyc`, `step-risk`, `step-documents`, `step-mandate`, `step-review` |
| Wizard card | `wizard-card` |
| Back | `wizard-back` |
| Continue | `wizard-next` |
| Submit for compliance approval | `wizard-submit` |
| Discard draft | `discard-draft` |
| Field error (any field `k`) | `error-{k}` |

### Stage 1

`client-type` · `type-individual` · `type-joint` · `type-trust` · `type-corporate` · `input-entityName` · `input-registrationNo` · `input-lei` · `input-trustee` · `input-settlor` · `input-firstName` · `input-lastName` · `input-dob` · `calendar-dob` · `input-nationality` · `input-taxResidency` · `input-email` · `input-phone` · `input-addressLine` · `input-city` · `input-postcode` · `input-country` · `input-adviser` · `input-introducedBy`

### Stage 2

`input-idType` · `input-idNumber` · `input-idExpiry` · `calendar-idExpiry` · `input-pep` · `pep-no` · `pep-yes` · `input-pepDetail` · `pep-notice` · `input-sourceOfWealth` · `input-sourceOfFunds` · `input-expectedInflow` · `run-screening` · `screening-pending` · `screening-outcome` · `input-screeningOverride` · `input-overrideReason`

### Stage 3

`risk-questionnaire` · `rq-q1` … `rq-q8` · `q1-opt1` … `q8-opt4` · `risk-score` · `risk-band` · `input-objective` · `input-horizon` · `input-liquidity` · `input-esg` · `suitability-ok` · `suitability-mismatch`

### Stage 4

`upload-list` · `upload-doc-id` · `upload-doc-address` · `upload-doc-sow` · `upload-doc-tax` · `upload-doc-ima` · `file-{docId}` · `sample-{docId}` · `remove-{docId}`

### Stage 5

`input-mandate` · `input-currency` · `input-fundingAmount` · `input-custodian` · `input-modelPortfolio` · `mp-aligned` · `mp-mismatch` · `add-signatory` · `signatory-table` · `signatory-row-{i}` · `signatory-name-{i}` · `signatory-capacity-{i}` · `signatory-authority-{i}` · `remove-signatory-{i}` · `fee-table` · `fee-total`

### Stage 6 and approval

`review-incomplete` · `review-complete` · `checklist` · `check-{slugged label}` · `review-prospect` · `review-kyc` · `review-mandate` · `input-declaration` · `approve-note` · `approve-confirm` · `reject-reason` · `reject-reason-error` · `reject-confirm`

---

### Release 2.1 controls

| Element | `data-testid` |
|---|---|
| New features toggle | `release-toggle` |
| Release banner | `release-banner` |
| New badge | `rel-new` — shared by both new field labels, decorative, do not target it |
| Vulnerability indicators | `input-vulnerability` |
| Support plan container | `vulnerability-support-field` |
| Support plan | `input-vulnerabilitySupport` |
| Sustainability preference | `input-sustainability` |
| Indicative fee on review | `review-fee` |
| Fee table | `fee-table` · total at `fee-total` |

The release toggle is a **button**; its state is on `aria-pressed`, which reads `"true"` while the release is on.

## 11. Test data that produces each branch

The application's clock is **fixed at 2026-09-22**. All date arithmetic — age, document expiry, KYC review due — is calculated from that date, so date-driven tests are deterministic and do not drift.

### Screening outcomes

| Target outcome | Set |
|---|---|
| Clear | Any non-watchlist name, PEP = No, nationality and tax residency both outside the four high-risk jurisdictions |
| PEP confirmed | PEP = Yes, with a rationale in the detail field |
| Elevated risk | Nationality **or** tax residency = Russian Federation, Islamic Republic of Iran, Panama or Cayman Islands |
| Potential match | Last name containing `Petrov`, `Al-Mansour`, `Okonkwo-Sanction` or `Delacroix-Holdings` — then adjudicate with ≥ 20 characters to proceed |

### Risk bands

| Target band | Answer pattern |
|---|---|
| Conservative (8–13) | All eight questions option 1 → score 8 |
| Balanced (14–20) | All eight option 2 → score 16 |
| Growth (21–26) | All eight option 3 → score 24 |
| Aggressive (27–32) | All eight option 4 → score 32 |

### Suitability banners

| Target | Set |
|---|---|
| Consistent | Score 16 (Balanced) with objective **Balanced growth** |
| Mismatch, client more aggressive | Score 32 (Aggressive) with objective **Capital preservation** |
| Mismatch, objective more aggressive | Score 8 (Conservative) with objective **Aggressive growth** |

### Portfolio alignment

| Target | Set |
|---|---|
| Aligned | Score 16 (Balanced) with MP-03 Balanced Multi-Asset |
| Outside risk band | Score 8 (Conservative) with MP-05 Concentrated Equity Alpha |

### Document expiry

Boundaries below are exact, computed from the fixed clock of 2026-09-22T09:00Z.

| Target | Expiry date (stored, ISO) | **Type into the field as** |
|---|---|---|
| Accepted | 2026-12-21 or later | **`21-12-2026`** or later |
| Expires within 3 months | 2026-09-22 to 2026-12-20 inclusive | `22-09-2026` to `20-12-2026` |
| Expired | 2026-09-21 or earlier | `21-09-2026` or earlier |

### Conditional tax document

Set tax residency and country of residence to **different** countries — for example tax residency Singapore, country of residence United Kingdom. The tax residency form badge changes to mandatory.

### Other boundaries

| Rule | Just fails | Just passes |
|---|---|---|
| Age 18 | Date of birth **`22-09-2008`** | **`21-09-2008`** |
| Minimum mandate | 249,999 | 250,000 |
| Adjudication rationale | 19 characters | 20 characters |
| Rejection reason | 14 characters | 15 characters |
| Telephone digits | 7 digits | 8 digits |

> **Note on the age rule.** Age is computed as elapsed milliseconds divided by a 365.25-day year, not as a calendar birthday. The effective cut-off therefore lands roughly one day *before* the client's actual eighteenth birthday: a date of birth of 2008-09-22 evaluates to 17.9997 years and is rejected, while 2008-09-21 evaluates to 18.0024 and is accepted. Tests asserting the exact boundary must use these dates; tests that only need a valid adult should use a date comfortably clear of it.

### Date entry format

Both date fields — **Date of birth** (stage 1) and **Expiry date** (stage 2) — are plain text inputs in **`dd-mm-yyyy`** format with a calendar button beside them (`calendar-dob`, `calendar-idExpiry`).

- **Type the value as `dd-mm-yyyy`.** `21-09-2008`, `30-06-2028`.
- **ISO is also accepted on entry** (`2008-09-21`), and the field redisplays it as `21-09-2008`. Either form works, so a step written against the stored format is not wrong.
- **The record always stores ISO.** Every rule in this document — age, expiry, KYC review due — is evaluated against the stored ISO value, which is why the boundary tables above give both forms.
- `/`, `.` and spaces are accepted as separators on entry.
- The calendar button opens the browser's native picker. It is a convenience for a human; **automation should type the value** rather than drive the picker.

---

## 12. Known gaps and constraints

These are properties of the application as built. They are recorded so that tests are written against actual behaviour rather than assumed behaviour.

- **The appropriateness warning for Execution-only is not implemented.** The hint on the service-model field states that execution-only mandates bypass suitability but require an appropriateness warning. No such warning is produced, and no validation differs by service model.
- **Suitability and portfolio-alignment warnings do not block.** Both are advisory banners. A record with a documented mismatch can be submitted and approved with no exception workflow, despite the banner text stating a documented rationale or exception will be required.
- **Postcode is never mandatory**, including for United Kingdom addresses.
- **Signatory authority is never validated.** A signatory can be saved with a name and capacity but no authority.
- **The intermediate lifecycle stages are unreachable through the wizard.** `KYC in progress`, `Risk profiled` and `Documents pending` exist and appear on seeded records, but a wizard submission goes straight from Draft to Awaiting approval.
- **LEI is never enforced**, despite the hint that it is required before any MiFID-reportable transaction.
- **Uploaded files are not validated** for format or size beyond the file picker's `accept` attribute.
- **Portfolio management, Fees & billing and Client reporting are not implemented** and are disabled in the navigation.
- **State is held in browser local storage** under the key `cavendish.platform.v1`, shared with the motor insurance and pensions applications. The older standalone wealth build still deployed at the site root uses `cavendish.cmo.v1`, so the two deployments never see each other's records even though they share an origin. A journey must therefore be pointed at `platform.html` explicitly; opening the bare site root reaches a different application with different behaviour.
- **Reset demo data is scoped to the application you are in.** Resetting from a `#/dashboard`, `#/clients`, `#/onboarding` or `#/compliance` route restores the seeded client records, any onboarding draft and the client id counter, and **leaves motor insurance and pensions exactly as they are**. **The Applications launcher carries no reset control.** On `#/apps` the `reset-data` button is **hidden, not removed**: it stays in the DOM carrying the `hidden` attribute and `display:none`. A journey must therefore assert that it is **not visible**; asserting that the element does not exist will fail. No application is in scope on the launcher, so there is nothing for it to reset. A full reset of all three is on the sign in screen (`login-reset`), which is single-step with no confirmation modal. Any run that depends on a clean start should open the application it is about to exercise, then reset from inside it.
- **Reset demo data is a two-step control.** `reset-data` opens a confirmation modal; the reset happens on `reset-confirm` ("Reset now"). A single click changes nothing, and the modal names the application about to be reset, so it can be asserted.
- **Any reset clears a sign-in lockout.** The failed-attempt counters and the locked list are not demo records, so they are cleared whichever application you reset from. That keeps the sign-in message *"This account is locked after 5 failed sign-in attempts. Reset the demo data to unlock it."* true from everywhere.
- **Local storage belongs to one browser profile.** Two people demonstrating on their own machines, or in two different browsers on one machine, never affect each other's data; there is no server and no shared state. Two tabs of the same application in the same profile do share storage, and neither tab is told when the other writes.
- **The approval role gate and the self-approval check are separate mechanisms.** A Compliance user who submits their own onboarding passes the role gate and is stopped only by the identity check, with the toast *"Four-eyes control: the submitting user cannot approve the same record."* The record remains at Awaiting approval.
- **There is no inactivity timeout on the session.** A signed-in session persists until the user signs out or clears browser storage. The in-progress draft is likewise retained indefinitely, and no retention window is defined anywhere in the application.
- **No warning precedes account lockout.** Attempts one to four are indistinguishable from each other; the fifth locks the account outright.
- **The lockout has no time-based expiry.** It is cleared only by resetting the demo data.
- **The application is a single self-contained HTML file.** It has no server, no API and no network calls. Screening, scoring and fee calculation all run in the browser. Consequently there is nothing to stub, and equally nothing to seed through an interface — every test starts from the UI.
- **The application must be served over HTTP to be automated remotely.** It is hosted at `https://kishorespotqa.github.io/virtuoso-wealth-demo/`. Run from a local `file://` path it is reachable only by a browser on that same machine: an automated agent on shared cloud browser infrastructure cannot open a `file://` URL or a `localhost` address on someone else's laptop.

---

## 13. The end-to-end journey

The canonical path from an empty wizard to an activated mandate.

**Preconditions:** application reachable over HTTP. Credentials supplied from environment variables.

| # | Step | Expected |
|---|---|---|
| 0 | Navigate to the application | Sign-in screen shown; nothing else reachable |
| 1 | Sign in as `lena.fairbrook` | **Application launcher** shown at `#/apps`; top bar shows "Lena Fairbrook — Relationship Manager" and the role Adviser |
| 2 | Open Private Wealth with `app-wam`, then reset demo data from inside it: activate `reset-data`, then `reset-confirm` in the modal | The launcher carries no reset control, so the application must be opened first. 12 seeded records restored, any draft cleared, still signed in; motor insurance and pensions are left as they are. For a full reset of all three, use `login-reset` on the sign in screen before signing in |
| 3 | Navigate to Client management → New client, or go directly to the `#/onboarding/new` route | Wizard opens at Step 1 of 6, Back disabled |
| 4 | Leave classification as Individual; complete first name, last name, date of birth of an adult | — |
| 5 | Select nationality, tax residency, country of residence; enter email, telephone, address line, city; confirm a relationship manager | — |
| 6 | Continue | Step 2 of 6, KYC & AML |
| 7 | Select identity document type, enter document number, set an expiry more than 90 days ahead | — |
| 8 | Answer the PEP question | — |
| 9 | Select source of wealth and source of funds; enter expected annual inflow | — |
| 10 | Run screening | Outcome banner appears with the expected verdict for the data used |
| 11 | Adjudicate if the outcome is a potential match | — |
| 12 | Continue | Step 3 of 6, Risk & suitability |
| 13 | Answer all eight questions | Score and indicated band update live |
| 14 | Select objective, time horizon, liquidity requirement | Suitability banner appears |
| 15 | Continue | Step 4 of 6, Documents |
| 16 | Attach proof of identity, proof of address, source of wealth evidence, and the tax residency form if tax residency differs from country of residence | Each shows Received |
| 17 | Continue | Step 5 of 6, Accounts & mandate |
| 18 | Select service model and custodian; enter funding of at least 250,000; select a model portfolio | Alignment hint or mismatch banner appears; fee table calculates |
| 19 | Add signatories if the classification is Joint | — |
| 20 | Continue | Step 6 of 6, Review & submit |
| 21 | Verify the checklist | All seven checks Complete; "Ready for submission" banner |
| 22 | Tick the declaration | — |
| 23 | Submit for compliance approval | Toast confirming the assigned `CW-` reference; client detail page opens at stage Awaiting approval |
| 24 | **Sign out, then sign in as `desmond.achebe`** | Top bar shows "Desmond Achebe — Compliance Officer", role Compliance |
| 25 | Open the Compliance queue and select the record | Approve and Reject actions available |
| 26 | Approve and confirm | Stage Active; KYC review due 12 months from today; AUM equals the funding amount; audit trail records the approval |

**The journey is not complete at step 23.** A journey that ends at submission verifies the wizard but not the control that matters most — the four-eyes separation. Step 24 is the point of the whole process.

### Branch journeys worth deriving from this one

| Branch | Diverges at |
|---|---|
| Trust and Corporate classification | Step 4 — different mandatory fields |
| Joint classification | Step 19 — signatory becomes mandatory |
| Potential match adjudication | Step 11 |
| PEP declared | Step 8 — reveals a mandatory field and a banner |
| Conditional tax document | Step 16 |
| Suitability mismatch | Step 14 |
| Portfolio outside risk band | Step 18 |
| Rejection instead of approval | Step 26 — reason of at least 15 characters |
| Four-eyes violation | Step 26 attempted without step 24 — blocked with a toast |
| Sign-in: wrong password | Step 1 |
| Sign-in: unknown username | Step 1 — must produce the identical message |
| Sign-in: blank fields | Step 1 — field errors, no attempt recorded |

---

## 14. Notes for automation

### Release 2.1 — before anything else

Open `?release=on` as the journey's first navigation rather than clicking the toggle. The setter is absolute and idempotent, so the journey lands in the same state on every run whatever the browser carried.

- Do not operate `release-toggle` or `reset-data` from a journey.
- The release survives `Reset demo data` and survives a reload.
- Recording a vulnerability indicator redraws the step: wait for `vulnerability-support-field` rather than assuming the support-plan field is present.
- A journey asserting a fee figure must pin the release, because every figure in section 7 differs between releases.
- A journey opening an existing client record is stable across releases: the record keeps the schedule it was onboarded under.



Properties of this build that decide whether a generated journey runs or stalls.

### Select options must match exactly

Every option string below is ASCII. None contains a typographic dash, so a value typed with an ordinary keyboard hyphen matches.

Three option sets contain **prefix collisions**, where one option's full text is the opening of another's. A partial match is a coin flip; use the complete string.

| Dropdown | Colliding options |
|---|---|
| Country (nationality, tax residency, residence) | `United Kingdom` · `United States` · `United Arab Emirates` |
| Time horizon | none, but `3-5 years` and `5-10 years` differ by one character |
| Model portfolio | option text is `{name} - {band} ({equity}% equity)` |

The placeholder `Select…` is a selectable option that fails validation. It contains a single-character ellipsis, not three dots.

### Dates are typed, not picked

`dd-mm-yyyy` into `input-dob` and `input-idExpiry`. The calendar buttons exist for human use. See *Date entry format* in section 11.

### Fields accept keystrokes

Every text and number field can be typed into character by character; none is rebuilt mid-entry. Values entered by setting the value and firing a `change` event — the path most WebDriver-based tools take for a dropdown — are also accepted on every control.

### Element identification

Use `data-testid` throughout. Stage 1 replaces its entire field set when the classification changes to Trust or Corporate: the name block disappears and Nationality is relabelled *Country of incorporation*. Label-based identification breaks across those branches; test ids do not.

### Asynchronous behaviour

Screening is the only asynchronous operation. Wait on the `screening-outcome` element rather than for a fixed duration.

### File attachment

Use the `sample-{docId}` control on stage 4. The real file input beside it is `display:none` and cannot be driven.
| Sign-in: lockout after five failures | Step 1 — then unlock via `login-reset` |
| Draft survives sign-out | Sign out mid-wizard, sign back in, confirm captured values |

# Marchmont Bank - Payments and Payment Operations (release 2.1)

**Knowledge Base source for Touchstone**
**Application:** `banking.html`
**Application URL:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/banking.html`
**Build:** 2026.10.26
**Systems covered:** Marchmont Online, Sentinel Payment Operations
**Release:** 2.1 - describes the application with New features **on**
**Status:** Current - describes the application as deployed

This is the complete source. Upload this one file; there is nothing else to add
for this application.

### What is in here

| Section | Covers | System |
|---|---|---|
| 0 | Release 2.1, and what it changed | both |
| 1 | What the application is, and the reference chain that joins the two systems | both |
| 2 | The five sign in accounts, roles and what each may reach | both |
| 3 | Conventions that hold on every screen | both |
| 4 | Accounts, payees and the fixed clock | Marchmont Online |
| 5 | The three step payment wizard, field by field | Marchmont Online |
| 6 | **PA decision codes**, twelve of them | Marchmont Online |
| 7 | Step-up authentication | Marchmont Online |
| 8 | The cut-off, scheduling and standing orders | Marchmont Online |
| 9 | The hand off into Sentinel | Online to Sentinel |
| 10 | The hold engine and its **PH decision codes**, five of them | Sentinel |
| 11 | Investigation, release limits and the two four-eyes controls | Sentinel |
| 12 | The irrevocability boundary | both |
| 13 | Verified boundary values, read from the running application | both |
| 14 | Element identification | both |
| 15 | Cross system journeys worth testing | both |
| 16 | What this application deliberately does not do | both |
| 17 | Notes for automation | both |

**Thirty eight decision codes in total**, across three engines plus the payee name check. The prefix names the
engine: `PA` is the channel, `PH` is operations, `PY` is payee management,
and `CP` is the payee name check. A requirement must never cite
codes from both engines at once.

---

## 0. Release 2.1

**This document describes the application with the New features control on.**
The control is `data-testid="release-toggle"` in the sidebar, and a banner
reading `New features are on.` is shown above the top bar while it is.

With the control off the application runs its previous release, described by
`Marchmont-Banking-KB-Source.md`. That document is a complete description in its
own right; this one is not a supplement to it.

### What release 2.1 changed

| Change | Where | Effect |
|---|---|---|
| **Payment purpose** added | Payment wizard, step 2 | New mandatory select |
| **Source of funds** added | Payment wizard, step 2 | New mandatory select |
| **Step-up limit lowered** | `PA-S01` | From 1,000.00 to **500.00** |
| **`PH-H06` added** | Hold engine | Holds a payment whose declared source of funds is `Gift or inheritance` and whose amount is above 2,500.00 |
| **The payee name check becomes blocking** | New payee, step 1 | A `CP-03` no match is refused outright instead of being acknowledged and held |
| **Instruction received from** added | Payment wizard, step 2 | New mandatory select, shown only when paying under an authority |

Nothing else changed. Every other rule, message, field and test id in this
document is the same as in the previous release.

**A journey written against this document must pin the release on** rather than
relying on a browser that happens to carry it:

```
banking.html?release=on#/online
```

The setter is absolute and idempotent. Do **not** click `release-toggle` from a
journey: it is a toggle, so it is only correct from a known starting state.

**Download KB source v2** in the sidebar serves this document while the release
is on; with it off the control reads **Download KB source** and serves the
other, so the application never hands out a source describing behaviour it does
not have.

> **Existing payments are unaffected.** Each payment is stamped with the release
> it was submitted under and is decided on those rules permanently. A payment
> submitted under 1.0 keeps its 1.0 outcome and never acquires `PH-H06`, however
> the toggle is set afterwards. This was confirmed against the running
> application. The stamp is shown in `payment-release` on the payment detail.

---

## 1. What this application is

Marchmont Bank is a fictitious United Kingdom retail bank. The application is a
single page holding **two separate systems** joined by one payment:

| System | What it is | Who uses it |
|---|---|---|
| **Marchmont Online** | The customer channel. Accounts, payees, payments, standing orders, cancellation. | The customer |
| **Sentinel Payment Operations** | The bank side console. Hold queue, investigation, release or reject. | Payment Operations, Financial Crime |

The point of the application is the **chain between the two**. A payment
reference minted in the channel is the key used to find the payment in Sentinel.
A journey that reads the reference from the confirmation and searches for it in
the enquiry screen is doing the thing a static script cannot.

Signing in leads to an **applications launcher**. The two systems share a sign
in and a storage key and nothing else: different screens, different engines,
different code prefixes, different roles.

---

## 2. Sign in and roles

| Actor | Name in the system | Username | Role | What they can reach |
|---|---|---|---|---|
| Customer, Standard tier | Rose Whitfield | `rose.whitfield` | Customer | Marchmont Online only |
| Customer, Premier tier | Daniel Okonjo | `daniel.okonjo` | Customer | Marchmont Online only |
| Operations | Sean Hartley | `sean.hartley` | Payment Operations | Sentinel only |
| Financial crime | Nadia Prasad | `nadia.prasad` | Financial Crime | Sentinel only |
| Administrator | Omar Farrell | `omar.farrell` | Platform Administrator | Both |

Password for all five: `Marchmont#2026`.

**The acting user is established by signing in.** There is no role selector.
This matters for testing: both four-eyes controls in section 11 compare the
acting user against a recorded one, so a journey that exercises them must sign
out and sign back in.

### Sign in behaviour

- The failure message is **identical** for a wrong password and an unknown
  username: `The username or password is not correct.` A requirement asserting
  a distinct "user not found" message is asserting behaviour this application
  deliberately does not have.
- **Blank-field submission does not count towards the lockout.** Field
  validation runs before any credential check, so submitting an empty form
  returns `Username is required.` and `Password is required.` and leaves the
  attempt counter alone.
- After **five** failed credential checks the account is locked:
  `This account is locked. Reset the demo data to unlock it.` There is **no
  warning before the lockout** and **no time based expiry**. It clears only on a
  demo data reset.
- The form submits on Enter as well as on the button.
- Signing out returns to the sign in screen and clears the session.
- **The session survives a page reload.** It is held under its own key,
  `marchmont.banking.session.v1`, separate from the demo data, so refreshing
  the browser does not sign the user out and **Reset demo data** does not
  either. Only signing out, or the `?session=out` setter below, ends it.

### Role gates

Reaching a system a role cannot use returns the user to the launcher with the
toast `Your role does not have access to that application.` The launcher card
for an unavailable system is still rendered and still carries its test id, so a
journey must assert the resulting route or the toast, not the absence of the
card.

---

## 3. Conventions on every screen

- Every interactive element carries a stable `data-testid`. Prefer these over
  label text or position.
- **Money is displayed without a currency symbol** in tables and fields, as
  `1,250.00`. Headline figures carry the prefix `GBP`. Amounts are entered as
  plain pounds and pence, for example `1250.00`; a comma is accepted and
  stripped.
- **Dates are displayed and entered as `dd-mm-yyyy`.** Every date field is a
  text input holding that format, with a **calendar button beside it**. The
  button opens the browser's own picker and writes the chosen day back into the
  text input as `dd-mm-yyyy`, so the value on the page is identical whether it
  was typed or picked. **The text input is the field of record.** A journey
  should type into it rather than drive the calendar: date pickers are a known
  weak control type for generated automation, and the button adds nothing a
  typed value does not already achieve.
- Reference numbers:

| Kind | Format | Example |
|---|---|---|
| Payment | `MPY-` plus six digits | `MPY-000001` |
| Standing order | `MSO-` plus five digits | `MSO-00001` |
| Case | `MFC-` plus five digits | `MFC-00001` |

- Counters are shared and sequential across both customers and do not reset
  except on a demo data reset.
- **Reset demo data** is a single control, `reset-data` in the sidebar and
  `login-reset` on the sign in screen. There is no confirmation step: one
  activation performs the reset. It restores accounts, payees, payments,
  standing orders and cases, and clears lockouts. It does **not** change which
  release is running.

---

## 4. Accounts, payees and the clock

### The fixed clock

**The application evaluates every date rule against a fixed clock date of
2026-10-26, a Monday.** A relative date drifts against a fixed clock and a
journey written with "today" rots. Use the literal dates the test data supplies.

The **time of day** defaults to `10:00` and is settable from the URL:

```
banking.html?now=16:30#/online
banking.html?now=09:00#/online
```

This exists because the cut-off in section 8 would otherwise be unreachable: a
clock fixed at 10:00 is always before 16:00 and `PA-A03` could never be
produced. The setter is absolute and is applied when the page loads, so a
journey navigates with it **first and then signs in**. A value outside `00:00`
to `23:59`, or one that is not `HH:MM`, is ignored rather than guessed at.

The current clock is shown in the top bar as `Clock 26-10-2026 10:00`.

### Starting signed out

Because a session survives a reload, a journey that navigates and then signs in
would find itself already signed in and the sign in form absent. The third
setter makes the starting state deterministic:

```
banking.html?session=out
banking.html?release=off&session=out&now=09:00
```

`?session=out` signs out at page load. Any other value is ignored. The three
setters are independent and combine in any order.

**A journey should open with `?session=out`** unless it deliberately relies on
an existing session.

### Seeded accounts

| Customer | Account | Type | Sort code | Number | Balance |
|---|---|---|---|---|---|
| Rose Whitfield (C-100412, Standard) | Everyday Current | Current | 30-91-55 | 41220119 | 28,450.80 |
| Rose Whitfield | Reserve Saver | Savings | 30-91-55 | 41220127 | 15,400.00 |
| Daniel Okonjo (C-100873, Premier) | Premier Current | Current | 30-91-55 | 48801934 | 124,809.00 |
| Daniel Okonjo | Premier Saver | Savings | 30-91-55 | 48801942 | 389,500.00 |

The payment form defaults to the **first** account in this list for the signed
in customer. A journey testing a rule that depends on the balance must set the
account explicitly rather than relying on the default.

### Seeded payees

| Id | Customer | Name | Country | Status |
|---|---|---|---|---|
| P-0001 | Rose | Hannah Whitfield | United Kingdom | Established |
| P-0002 | Rose | Meridian Utilities | United Kingdom | Established |
| P-0003 | Rose | Kestrel Letting | United Kingdom | **New** |
| P-0004 | Daniel | Okonjo Holdings | United Kingdom | Established |
| P-0005 | Daniel | Vertex Legal LLP | United Kingdom | Established |
| P-0006 | Daniel | Albrecht Partners | Germany | **New** |
| P-0007 | Daniel | Clonmel Trading | Ireland | Established |
| P-0008 | Rose | Oakfield Services Ltd | United Kingdom | **New, and in cooling off until 12:00** |

**Established and new are different things from the point of view of two
separate rules.** A new payee always triggers step-up under `PA-S02`. A payee
*created during this session* additionally triggers the hold `PH-H04` above
2,500.00. `P-0003` and `P-0006` are seeded as new but were not created today, so
they trigger the step-up and not the hold.

**`P-0008` is seeded inside its cooling off window**, which closes at 12:00, so
that `PA-D07` is reachable without building a fixture first. It is Rose's
fourth payee: a journey that counts her payees must expect four, not three.

**`P-0007` exists so that `PA-S03` can be observed at all.** It is the only
established overseas payee; `P-0006` is overseas but new, and `PA-S02` is
evaluated first and would always win.

**`Albrecht Partners` returns a sanctions match** in Sentinel. The demonstration
watchlist matches the names `albrecht`, `voronov` and `ferreira-dos-santos`
anywhere in the payee name, case insensitively.

---

### The count tiles are controls

**Awaiting review** and **Scheduled** on the overview are buttons, not labels.
Activating one filters the payment list below to exactly the records it counted,
sets `aria-pressed` to `true`, changes the card heading, and shows a clear
control in `payments-filter`. Activating the same tile again clears the filter,
so the control is its own off switch.

This matters for a journey that counts rows: the list is filtered state, and a
row count taken after activating a tile is not the same assertion as a row count
taken before it. Clear the filter, or assert the tile's pressed state alongside
the count.

---

### Payment limits set by the customer

`#/online/limits`, reached from `nav-limits`. The customer may set a daily limit
**at or below** their tier limit; a higher one is refused with
`Your limit cannot be above the bank limit of {figure}.`

| Element | `data-testid` |
|---|---|
| The tier limit | `limit-bank` |
| The customer's own limit, or `Not set` | `limit-own` |
| Whichever applies today | `limit-effective` |
| The entry field | `input-limit` |
| Save, and remove | `btn-save-limit`, `btn-clear-limit` |
| Error | `error-limit` |

The effective limit is the **lower** of the two, and the refusal code names
whichever one bound: `PA-D06` where the customer's own limit is the tighter,
`PA-D05` where it is not set or is equal to the tier limit. A customer is never
told they reached the bank's limit when it was their own.

Both counts are **cumulative across the day**, so a journey that does not reset
inherits the previous run's spending and the boundary moves.

---

### Managing payees

`#/online/payees`, reached from `nav-payees`. The screen lists the customer's
payees and is also where they are added, amended and removed. Three further
routes sit under it:

| Route | Reached by | Shows |
|---|---|---|
| `#/online/payees` | `nav-payees` | The list |
| `#/online/payees/add` | `btn-add-payee` | The add form |
| `#/online/payees/edit/<id>` | `btn-edit-<id>` | The amend form |

**Two columns that are often confused.** *Status* is the payee's own lifecycle,
and *Seen before* is whether the bank has ever paid it. They move independently,
and a payee that is `Active` and `New` at the same time is the ordinary state of
one whose cooling off has passed and which has never been paid.

| Status | Meaning | Can it be paid |
|---|---|---|
| Cooling off | Added less than two hours ago by the session clock | No, `PA-D07` |
| Available | Added, and the two hours have passed | Yes |
| Active | Seeded, or a payee whose window closed and was reconciled | Yes |
| Removed | Taken off the list by the customer | Not offered at all |

### The cooling off period

A payee added from this screen cannot be paid for **two hours of the session
clock**. At the default clock of 10:00 it becomes payable at 12:00, so the
`?now=` setter already documented reaches both sides of the boundary:

    banking.html?now=11:59#/online/payees    still inside the window
    banking.html?now=12:00#/online/payees    payable

It is expressed in hours rather than days deliberately. On a fixed clock date a
cooling off measured in days would never expire, and the rule that enforces it
could never be observed passing.

**A payee in cooling off is still offered in the payment form**, with the reason
in the option text, and refused at the decision with `PA-D07`. It is not hidden:
hiding it would make the rule unreachable and leave a customer looking for a
payee they had just added.

**`P-0008`, Oakfield Services Ltd, is seeded inside its window**, which closes at
12:00. Without it, a journey covering `PA-D07` would have to add a payee first
and build its own fixture on every run.

**A payee created inside a payment is not refused by its own window.** That
payment is what authorised the payee, so it proceeds; the window applies from
then on, and the *second* payment to that payee is the one that returns
`PA-D07`.

### Adding a payee

| Element | `data-testid` |
|---|---|
| Name, country | `input-new-payee-name`, `input-new-payee-country` |
| Sort code, account number | `input-new-payee-sort`, `input-new-payee-number` |
| Reference | `input-new-payee-reference` |
| Save, cancel | `btn-save-payee`, `btn-cancel-payee` |
| The outcome, and its code | `payee-outcome`, `payee-code` |
| The name check result | `payee-cop` |
| The acknowledgement | `input-new-payee-cop-confirm`, `error-copConfirmed` |

**Country is a revealing control here as well as in the payment wizard.** The
sort code field is **absent** outside the United Kingdom, where the account is
identified by an international account number alone.

The name is checked against the name the receiving bank holds, using the same
registry and the same `CP` codes as the payment wizard. **The check runs when
Save is activated**, not as the fields are typed.

### Amending a payee

**Only the name and the reference can be changed.** The sort code and account
number are shown as text with **no control at all**, not a disabled one: a payee
paid into a different account is a different payee.

| Element | `data-testid` |
|---|---|
| Name, reference | `input-edit-payee-name`, `input-edit-payee-reference` |
| The acknowledgement | `input-edit-payee-cop-confirm` |
| The account, as text | `edit-sort`, `edit-number` |
| Save | `btn-save-amend` |

**Changing the name restarts the cooling off period** and runs the name check
again, because the bank is being told to pay a differently named party.
**Changing the reference alone does neither.** This is the single most useful
distinction on the screen to write a requirement against.

### Removing a payee

Removal asks for confirmation inline, in `remove-confirm`, with
`btn-confirm-remove` and `btn-cancel-remove`. **Nothing is deleted.** The record
is marked `Removed` so that the payee name on a historic payment still resolves;
it leaves the list and the payment form, and `btn-show-removed` brings it back
into view.

A refusal appears in `error-remove` with its code in `remove-code`.

**The same account can be added again after removal.** The duplicate check
ignores removed payees, and the new payee starts a fresh cooling off period.

---

## 5. The payment wizard

Three steps, reached from **Make a payment** on the overview or the sidebar.

> **A draft survives.** The wizard keeps its state across navigation and sign
> out. Returning to `#/online/pay/new` resumes wherever the draft was left,
> which for an automated run means the wizard may not open on step 1. **Start a
> payment from the `btn-new-payment` control**, which clears the draft first.

The step counter reads `Step n of 3` in `wizard-progress`, and each step chip
carries `step-1`, `step-2`, `step-3`.


### Acting for another customer

A customer who holds an authority over someone else's account sees an **Acting
for** select at the top of step 1, `data-testid="input-acting-for"`. It is
**absent entirely** for a customer who holds none, so a journey must not assume
the control exists.

Choosing an authority changes whose records the wizard works with: the payee
list becomes the **account owner's** payees, and the account list becomes the
owner's accounts. Both selections are cleared when the choice changes, because
carrying a payee across to a customer it does not belong to would be wrong.

A summary of the chosen authority is shown in `authority-summary`, with its type
in `authority-type`.

### Seeded authorities

| Id | Holder | Over | Type | Per payment | Payees | Expires |
|---|---|---|---|---|---|---|
| `AU-0001` | Daniel Okonjo | Rose Whitfield, Everyday Current | Power of attorney | 2,500.00 | Established only | 31-03-2027 |
| `AU-0002` | Daniel Okonjo | Rose Whitfield, Reserve Saver | Third party mandate | 1,000.00 | Any | **30-09-2026, expired** |

`AU-0002` is seeded already expired so that `PA-T04` can be observed without
editing data. The authority is held against **one account**, not the whole
customer, which is what makes `PA-T01` observable: the owner's other account
appears in the list and is refused.

---

### Step 1 - Payee

A radio set with two values.

| Control | `data-testid` | Notes |
|---|---|---|
| An existing payee | `mode-existing` | Default |
| Someone new | `mode-new` | Reveals four fields and hides the payee select |

Changing this radio **replaces the field set below it**. An agent working from a
stale read will type into a select that is no longer on the page.

**Existing payee**

| Field | `data-testid` | Mandatory | Error |
|---|---|---|---|
| Payee | `input-payee` | Yes | `Select a payee.` |

**Someone new**

| Field | `data-testid` | Mandatory | Validation | Exact error |
|---|---|---|---|---|
| Payee name | `input-payee-name` | Yes | At least 2 characters | `Payee name is required.` / `Enter the payee's full name.` |
| Country | `input-payee-country` | Yes (defaulted to United Kingdom) | - | - |
| Sort code | `input-payee-sort` | United Kingdom only | `00-00-00` | `Sort code is required.` / `Enter the sort code as 00-00-00.` |
| Account number | `input-payee-number` | Yes | Eight digits in the UK, otherwise an IBAN | `Account number is required.` / `A United Kingdom account number is eight digits.` / `An IBAN is required for an international payee.` / `Enter a valid IBAN, for example DE89370400440532013000.` |

**Changing the country changes which fields are shown.** The sort code field is
present for a United Kingdom payee and **absent** otherwise, and the account
number validation switches between an eight digit pattern and an IBAN pattern.
Re-read the form after changing the country: a journey that holds a reference to
the sort code field across the change is holding a detached element.

The country list is: United Kingdom, Germany, France, Ireland, United States,
United Arab Emirates, Australia, Korea (Democratic People's Republic of), Iran,
Syria. **Select by the full exact visible text.** A partial match on `United`
is a coin flip between three entries, and the wrong one changes the outcome.

**A new payee is created only when the payment is recorded.** An abandoned
draft, and a payment rejected by the channel, both leave no payee behind.


### Confirmation of Payee

A **new** payee is checked against the name the receiving bank holds for that
account number. An established payee was checked when it was added and is not
rechecked.

The check runs **when Continue is activated**, with the rest of step 1's
validation, not as the fields are typed. Its outcome appears in `cop-outcome`
with the code in `cop-code`, and on the review step in `review-cop`.

| Code | Condition | Effect in release 1.0 |
|---|---|---|
| `CP-01` | The name matches the account | Continue |
| `CP-02` | The surname matches, the forename differs. The held name is shown | Acknowledgement required |
| `CP-03` | The account is known and the name does not match. The held name is shown | **Refused.** `The name does not match the account. This payment cannot proceed.` in `error-np-name`, and no acknowledgement is offered |
| `CP-04` | The account number is not one the check covers | Continue, advisory only |

Where an acknowledgement is required, a checkbox appears,
`data-testid="input-cop-confirm"`, reading *"I have checked the payee's details
and want to continue anyway."* Activating Continue without it returns
`Confirm you have checked the payee's details before continuing.` in
`error-copConfirmed`.

**This control changed between releases, and it is the clearest advisory versus
blocking pair in the application.** In release 1.0 a `CP-03` no match warns, the
customer acknowledges it, the payment proceeds and is held under `PH-H07`. In
this release it is refused at step 1 and `input-cop-confirm` is **not rendered**
for it. A `CP-02` close match is still advisory in both releases.

**`PH-H07` is therefore unreachable from the no-match path in this release.**
It remains in the hold engine and can only be reached by a payment submitted
under release 1.0 and viewed later. Record it under branches not covered with
that reason.

### The names the check knows

| Account number | Held name |
|---|---|
| 70113396 | Hannah Whitfield |
| 88204417 | Meridian Utilities Ltd |
| 31559028 | Kestrel Letting Ltd |
| 62340018 | Okonjo Holdings Ltd |
| 29087741 | Vertex Legal LLP |
| 12345678 | Marcus Trent |
| 99887766 | Oakfield Services Ltd |
| 44556677 | Priya Raghunathan |

Comparison ignores case, punctuation and extra spacing. "Surname" is the last
word of the name. **Any account number outside this table returns `CP-04`**, so
a journey that needs a clean new payee should use one, and a journey covering
the check should use one of these.

---

### Step 2 - Payment

| Field | `data-testid` | Mandatory | Validation | Exact error |
|---|---|---|---|---|
| Pay from | `input-from` | Yes (defaulted) | - | `Select an account to pay from.` |
| Amount | `input-amount` | Yes | Pounds and pence, greater than zero | `Amount is required.` / `Enter an amount in pounds and pence, for example 1250.00` / `Enter an amount greater than zero.` |
| Payment type | `kind-immediate`, `kind-scheduled`, `kind-standing-order` | Yes (defaulted to Immediate) | - | - |
| Send by | `input-rail` | Not shown for a standing order | Faster Payment or CHAPS | - |
| Value date | `input-date` | Yes | `dd-mm-yyyy`, not past, at most 365 days ahead | `Value date is required.` / `Enter the date as dd-mm-yyyy.` / `The value date cannot be in the past.` / `A payment cannot be dated more than 365 days ahead.` / `An immediate payment must be dated today. Choose Scheduled for a later date.` / `A scheduled payment must be dated after today. Choose Immediate to pay today.` |
| Frequency | `input-frequency` | Standing order only | Weekly, Monthly, Quarterly | - |
| End date | `input-end-date` | No | After the first payment date | `Enter the date as dd-mm-yyyy.` / `The end date must be after the first payment date.` |
| Reference | `input-reference` | No | At most 18 characters | `A reference can be at most 18 characters.` |
| Instruction received from | `input-instruction-from` | **Yes, when acting under an authority** | Telephone, Written instruction, In person, Secure message | `Record where the instruction was received from.` |
| Payment purpose | `input-purpose` | **Yes** | One of six values | `Payment purpose is required.` |
| Source of funds | `input-source-of-funds` | **Yes** | One of six values | `Source of funds is required.` |

**Changing the payment type adds and removes fields.** Choosing Standing order
hides **Send by** and reveals **Frequency** and **End date**. Re-read the form
after changing it.

**The payment type and the value date must agree.** An immediate payment must be
dated today; a scheduled payment must be dated after today. The date field
defaults to today, so choosing Scheduled and continuing without changing the
date is refused rather than quietly producing an immediate payment. Each refusal
names which control to change instead, so a requirement should assert the
message rather than only the failure.

**Payment purpose** takes one of: Goods and services, Property, Family support,
Investment, Salary, Other. **Source of funds** takes one of: Salary or earnings,
Savings, Property sale, Investment income, Gift or inheritance, Business income.
Both carry a `New` badge on their label in this release. Neither changes any
channel decision; **Source of funds** feeds the hold rule `PH-H06` in section
10.

**Instruction received from** is shown only when the payment is being made under
an authority, so it is absent on a payment a customer makes for themselves. It
is recorded on the payment and shown to Sentinel, and affects no decision.

**Every failing field is reported at once**, each message naming its own field,
so an agent that reads the errors can correct everything in one pass.

### Step 3 - Review

The review shows an **indicative outcome** in `indicative-outcome`, with the
code in `indicative-code`. It is advisory: the banner itself says so, and the
decision is confirmed on submission.

The review also shows the two new fields, in `review-purpose` and
`review-source-of-funds`.

**The value date shown on review is the effective one.** Where the cut-off has
moved the payment to the next working day, the review shows the moved date and
a `Moved` badge in `review-date-moved`, not the date typed into the form.

---

## 6. PA decision codes

The channel evaluates **eighteen rules in a fixed order and returns the first
that matches**: eleven rejections, then three step-up triggers, then four
acceptances.

| Code | Kind | Condition | Reason text |
|---|---|---|---|
| PA-T04 | Reject | The authority has expired | `This authority has expired. Contact the bank to renew it.` |
| PA-T01 | Reject | Acting under an authority that does not cover the chosen account | `You do not hold authority over the account you have chosen.` |
| PA-T02 | Reject | The authority permits established payees only, and the payee is not established | `This authority allows payments to established payees only.` |
| PA-T03 | Reject | Above the authority's per-payment limit | `This payment is above the limit set on your authority.` |
| PA-D07 | Reject | The payee is in its cooling off period | `This payee is still in its cooling off period and cannot be paid yet.` |
| PA-D01 | Reject | Amount above the available balance on the chosen account | `The available balance on this account is not enough to make this payment.` |
| PA-D02 | Reject | Faster Payment above 1,000,000.00 | `A Faster Payment cannot be more than 1,000,000.00. Use CHAPS for a higher value.` |
| PA-D03 | Reject | Standing order instalment above 25,000.00 | `A standing order instalment cannot be more than 25,000.00.` |
| PA-D04 | Reject | Payee country is Korea (DPR), Iran or Syria | `Payments to this destination country are not available.` |
| PA-D06 | Reject | Amount plus the day's committed total above the customer's own daily limit, where that is the tighter | `This payment would take you past the daily limit you set yourself.` |
| PA-D05 | Reject | Amount plus the day's committed total above the customer's daily limit | `This payment would take you past your daily payment limit.` |
| PA-S01 | Step-up | Amount at or above **500.00** | `This payment is above your step-up limit, so it needs a passcode.` |
| PA-S02 | Step-up | Payee is not established | `This is your first payment to this payee, so it needs a passcode.` |
| PA-S03 | Step-up | Payee country is not the United Kingdom | `International payments need a passcode.` |
| PA-A04 | Accept | Payment type is Standing order | `Standing order set up.` |
| PA-A03 | Accept | CHAPS, dated today, submitted at or after 16:00 | `Submitted after the 16:00 CHAPS cut-off, so this payment is dated the next working day.` |
| PA-A02 | Accept | Value date is not today, which validation now guarantees for a Scheduled payment | `Scheduled for a future date.` |
| PA-A01 | Accept | Everything else | `Accepted for execution today.` |

### Precedence, and what it hides

- **`PA-D07` is evaluated ahead of the balance**, for the same reason the
  authority rules are: a payee that cannot be paid at all is not a payment
  worth rating. A 50,000.00 payment from an empty account to a payee in
  cooling off reports `PA-D07`, not `PA-D01`.
- **`PA-D06` is evaluated before the step-up triggers.** A payment of
  5,000.01 against an own limit of 5,000.00 reports `PA-D06`, not `PA-S01`,
  even though it is far above the step-up threshold.

- **An expired authority reports `PA-T04` and nothing else.** A 5,000.00 payment
  to a payee that is not established, under `AU-0002`, satisfies `PA-T04`,
  `PA-T02` and `PA-T03` at once and reports only `PA-T04`.
- **The authority rules are evaluated before everything else**, including the
  balance. A payment nobody is entitled to make is not rated.
- **`PA-D06` and `PA-D05` are one rule with two codes.** Where the customer's
  own limit equals the tier limit, `PA-D05` is reported, not `PA-D06`.

Several conditions overlap and **only the earlier one is ever observable**.
Each of these was confirmed against the running application:

- **A 30,000.00 payment from an account holding 28,450.80 reports `PA-D01`,
  never `PA-D05`.** A test for the daily limit written with too large a number
  tests the balance rule instead.
- **A 5,000.00 payment to a new payee reports `PA-S01`, never `PA-S02`.** The
  amount trigger outranks the new-payee trigger.
- **A 1,500.00 payment to an established overseas payee reports `PA-S01`, never
  `PA-S03`.** In this release `PA-S03` is only observable below **500.00**, so
  the window for observing it is half what it was.
- **`PA-D02` cannot be reached through the interface.** No seeded account holds
  more than 389,500.00, and the highest daily limit is 100,000.00, so `PA-D01`
  or `PA-D05` always fires first. Thirteen of the seventeen codes in this
  application are reachable; this is the one that is not. Cover it under
  branches not covered with the reason, not as a submitted payment.

### After a step-up

The step-up codes **hide the acceptance sitting behind them**. Once the passcode
is accepted the engine runs again with the step-up rules skipped, and the
payment is recorded with whichever acceptance code applies. So a 12,000.00
payment shows `PA-S01` on review and is recorded as `PA-A01`. In this release
that applies from 500.00 upwards rather than 1,000.00, so it covers more
payments than before.

A requirement asserting that the recorded outcome code equals the indicative one
is wrong for every stepped-up payment.

---

## 6a. PY decision codes

Payee management has its own engine and its own prefix. **A `PY` code describes
something that happened to the payee record, never to a payment.** A requirement
citing a `PY` code and a `PA` code together is describing two separate events.

The payment wizard never produces a `PY` code: a payee created inside a payment
is created by that payment and is rated by `PA`.

| Code | Kind | Condition | Reason text |
|---|---|---|---|
| PY-R02 | Reject | Sort code or account number not valid for the country | `Enter an account number of eight digits.` and similar |
| PY-R01 | Reject | An existing payee, not removed, already holds that sort code and account number | `You already have a payee for this account: {name}.` |
| PY-R03 | Reject | The name check returned `CP-03` | `The name does not match the account. This payee cannot be added.` |
| PY-R06 | Reject | The name check asked for an acknowledgement and it was not given | `Confirm you have checked the payee's details before continuing.` |
| PY-A01 | Accept | Payee added, and in cooling off | `Payee added. It can be paid from {time}.` |
| PY-A02 | Accept | Amendment saved | `Name changed. The cooling off period has restarted.` or `Reference changed.` |
| PY-R04 | Reject | Removal refused: an active standing order pays this payee | `Standing order {ref} pays this payee. Cancel it first.` |
| PY-R05 | Reject | Removal refused: a payment to this payee is held | `Payment {ref} to this payee is under review. It must be settled or rejected first.` |
| PY-A03 | Accept | Payee removed | `Payee removed.` |

**Format is checked before duplication, and the name check runs last.** An
account number that is both malformed and a duplicate returns `PY-R02`, never
`PY-R01`, and a duplicate is reported before the name is ever checked.

**`PY-R06` is the same acknowledgement the payment wizard asks for**, raised by
the same `CP-02` and `CP-03` outcomes, with the tick at
`input-new-payee-cop-confirm` on the add form and `input-edit-payee-cop-confirm`
on the amend form. Adding a payee and paying one give the same answer to the
same question.

**`PY-R03` refuses outright.** In release 2.1 a name the receiving bank
disputes cannot be added at all, and no acknowledgement is offered. A close
match still only asks, under `PY-R06`.

**The duplicate check ignores the name.** Two payees are the same when the sort
code and account number match; the same account under a different spelling is
the same account.

---

## 7. Step-up authentication

When the channel returns a step-up code, submission does not complete. The
journey lands on `#/online/pay/confirm`.

| Element | `data-testid` |
|---|---|
| The trigger and its code | `stepup-reason` |
| The passcode, shown on screen | `stepup-passcode`, value in `passcode-value` |
| Passcode entry | `input-passcode` |
| Confirm | `btn-stepup-confirm` |
| Cancel the payment | `btn-stepup-cancel` |
| Wrong passcode error | `error-passcode` - `That passcode is not correct.` |

**The passcode is displayed on the page.** It is derived from the amount rather
than randomised, so it is stable for a given payment and a journey can read it
from `passcode-value` and type it back. Do not hard code a passcode: it differs
per amount.

- **Three attempts.** The third incorrect passcode abandons the payment, clears
  the draft and returns to the overview with the toast `Too many incorrect
  passcodes. The payment was not made.` Nothing is recorded.
- **Cancel abandons the payment**, clears the draft and records nothing.
- The attempt counter is shown as `Attempts used: n of 3`.

---

## 8. Cut-off, scheduling and standing orders

### The CHAPS cut-off

**16:00, and it applies to CHAPS only.** Faster Payments are accepted at any
hour. A CHAPS payment dated today and submitted at or after 16:00 returns
`PA-A03` and is recorded with a value date of the **next working day**.

The comparison is **at or after**: 15:59 returns `PA-A01` and 16:00 returns
`PA-A03`. Both were confirmed against the running application.

### Working days

**Saturday and Sunday are not working days.** A value date falling on a weekend
is moved forward to the following Monday, both on review and on the recorded
payment. From the fixed clock date, a value date of Saturday 31-10-2026 or
Sunday 01-11-2026 both resolve to Monday 02-11-2026.

**No bank holiday calendar is modelled.** See section 16.

### Standing orders

A standing order is recorded in `#/online/standing-orders` rather than as a
payment. It carries a reference, the payee, the amount, the frequency, the next
date and an end date, and its status is `Active`.

- A standing order is **exempt from the daily limit** - `PA-D05` tests the
  payment type and skips it.
- Its instalment maximum is 25,000.00, enforced by `PA-D03`.
- It does not go to Sentinel. The hold engine runs on payments only.

---

## 9. The hand off into Sentinel

The payment reference minted on submission is the key into the other system.

1. Read `outcome-code` and the reference from the URL or from `grid-payments`
   in Marchmont Online.
2. Sign in to Sentinel as an operations user.
3. Type the reference into `filter-search` on `#/sentinel/payments`.
4. Open `open-<reference>` to reach the payment, and `open-case` to reach its
   case.

The hold queue at `#/sentinel` lists open cases directly, each row carrying
`row-<case reference>` and a link `open-<case reference>`.

**A journey that ends in one system is a partial journey and should be named as
one.**

---

## 10. The hold engine and PH decision codes

Operations evaluates **seven rules in a fixed order** on every payment the
channel accepted. The first that matches holds the payment and raises a case.
Nothing matching means the payment proceeds.

| Code | Condition | Reason text | Owned by |
|---|---|---|---|
| PH-H01 | Payee name matches the watchlist | `Sanctions name match on the payee.` | Financial Crime |
| PH-H02 | Amount at or above 10,000.00 | `Value at or above the anti money laundering review threshold.` | Financial Crime |
| PH-H03 | Fourth payment to the same payee within the day | `Repeat payments to the same payee within one day.` | Payment Operations |
| PH-H04 | First payment to a payee created today, above 2,500.00 | `First payment to a payee added today, above 2,500.00.` | Payment Operations |
| PH-H05 | Round sum international payment above 5,000.00 | `Round sum international payment above 5,000.00.` | Financial Crime |
| PH-H07 | The payee name check returned `CP-03` and the customer continued | `Payee name check did not match and the customer continued.` | Payment Operations |
| PH-H06 | Source of funds is `Gift or inheritance` and the amount is above 2,500.00 | `Declared source of funds requires review.` | Financial Crime |

### Precedence

- **A 20,000.00 payment to Albrecht Partners reports `PH-H01`, never `PH-H02`.**
  Confirmed against the running application. A requirement covering the AML
  threshold must not use a watchlisted payee.
- `PH-H05` is only observable between 5,000.01 and 9,999.99, because `PH-H02`
  takes everything at or above 10,000.00. A round sum of 6,000.00 holds under
  `PH-H05`; 6,000.01 is not a round sum and proceeds; 5,000.00 is not above the
  threshold and proceeds.
- "Round sum" means a whole multiple of 1,000.00.
- **`PH-H06` is evaluated last**, so anything already caught by an earlier rule
  never reaches it. A gift of 12,000.00 reports `PH-H02`, never `PH-H06`.
- **`PH-H06` is only evaluated on a payment submitted under release 2.1.** A
  payment stamped 1.0 is never held by it, whatever the toggle is set to when
  the payment is later viewed.

### What a hold does

The payment status becomes `Held`, `payment-hold` shows the code and reason, and
a case is raised with status `Open`. The case reference appears in the Sentinel
queue. The customer sees the hold on the payment detail and **can still cancel
it** - see section 12.

---

## 11. Investigation, release limits and four-eyes

### Investigation note

A decision cannot be recorded until the case carries an investigation note of
**at least 20 characters**. A shorter note returns
`An investigation note of at least 20 characters is required.` in
`error-case-notes`.

**Saving a note of 20 characters or more records the saver as the
investigator**, shown in `case-investigator`. That is what the second four-eyes
control compares against.

### Release limits

| Role | Release limit |
|---|---|
| Payment Operations | 5,000.00 |
| Financial Crime | No limit |
| Platform Administrator | No limit |

Above the limit: `This payment is above your release limit of 5,000.00. Only
Financial Crime can release it.`

**The limit sits below the anti money laundering threshold on purpose.** An
operations owned case only ever arises from `PH-H03` or `PH-H04`, and `PH-H02`
takes everything at or above 10,000.00 for Financial Crime. A limit above that
figure could never be reached through the interface. At 5,000.00 it divides the
operations queue: a new payee paid 4,000.00 can be released by Payment
Operations, and one paid 5,000.01 cannot. Both were confirmed against the
running application.

### Two independent four-eyes controls

**First, by case ownership.** A case owned by Financial Crime cannot be decided
by Payment Operations at all:
`This case is held for a financial crime reason and only Financial Crime can
decide it.`

**Second, by identity.** Whoever saved the investigation note cannot decide the
case:
`You investigated this case, so someone else must decide it.`

In both cases the release and reject buttons are **not rendered**. A notice
appears in `decision-blocked` instead. A requirement asserting a disabled button
is asserting something that does not exist - assert absence, not a disabled
state.

**There is one account per operations role.** So a journey that exercises the
identity check and a journey that completes a release are **two journeys, not
one**: Nadia Prasad can investigate and then be refused, but somebody else must
release it. Omar Farrell is the only other account with no release limit.

### Deciding

| Control | `data-testid` | Effect |
|---|---|---|
| Save note | `btn-save-note` | Records the note and the investigator |

**The note must be saved before a decision can be recorded.** Saving is what
names the investigator, so a decision cannot absorb unsaved text. Activating
Release or Reject with text in the box that has not been saved returns
`Save the investigation note before recording a decision.` and leaves the typed
text in place. Text shorter than twenty characters returns the length message
instead. A journey must activate `btn-save-note` before `btn-release` or
`btn-reject`.

| Release payment | `btn-release` | Status becomes `Released`, the account is debited, the case closes |
| Reject payment | `btn-reject` | Status becomes `Rejected`, no debit, the case closes |

Both record the deciding user in `decided-by` and append to the payment history.

---

## 12. The irrevocability boundary

This is the behaviour worth demonstrating, and the easiest to get wrong.

| Status | Cancellable by the customer |
|---|---|
| `Scheduled` | **Yes** |
| `Held` | **Yes** |
| `Released` | No |
| `Settled` | No |
| `Rejected` | No |
| `Cancelled` | No |

Where a payment can be cancelled, `btn-cancel-payment` is rendered. Where it
cannot, the button is **absent** and a notice appears in `not-cancellable`
instead.

**A Faster Payment that the channel accepts settles immediately.** It never
passes through `Scheduled`, so it is never cancellable at all, and the notice
reads `A Faster Payment settles the moment it is accepted.` That is the rail
behaving correctly, not a limitation of the demonstration.

A CHAPS payment, a scheduled payment and a held payment all reach a cancellable
state. Cancelling a held payment also closes its open case.

A cancelled or rejected payment **releases its share of the daily limit**,
because the limit counts committed payments only.

---

## 13. Verified boundary values

Every value here was read from the running application, not calculated. Where a
rule uses *at or above* rather than *above*, the pair below shows it.

| Rule | Just fails | Just passes | Notes |
|---|---|---|---|
| `PA-S01` step-up | 499.99 returns `PA-A01` | **500.00** returns `PA-S01` | At or above. Lowered in this release |
| `PA-D05` daily limit, Standard | 25,000.00 accepted | **25,000.01** returns `PA-D05` | Above. Cumulative across the day |
| `PA-D05` daily limit, Premier | 100,000.00 accepted | 100,000.01 returns `PA-D05` | Above |
| `PA-D03` standing order | 25,000.00 accepted | **25,000.01** returns `PA-D03` | Above |
| `PA-A03` CHAPS cut-off | 15:59 returns `PA-A01` | **16:00** returns `PA-A03` | At or after |
| `PH-H02` AML review | 9,999.99 proceeds | **10,000.00** holds | At or above |
| `PH-H04` new payee hold | 2,500.00 proceeds | **2,500.01** holds | Above |
| `PH-H05` round sum | 5,000.00 proceeds; 6,000.01 proceeds | **6,000.00** holds | Above, and a whole multiple of 1,000.00 |
| `PH-H03` velocity | Payments one to three proceed | **The fourth** holds | Same payee, same day |
| `PH-H06` source of funds | 2,500.00 as a gift proceeds; 2,500.01 from savings proceeds | **2,500.01** as a gift holds | Above, and the source must be `Gift or inheritance` |
| Weekend roll | Friday 30-10-2026 stays | Saturday 31-10-2026 and Sunday 01-11-2026 both become **Monday 02-11-2026** | |
| Value date horizon | 365 days ahead accepted | 366 days returns the error | |
| Reference length | 18 characters accepted | 19 returns the error | |
| `PA-T03` authority per-payment limit | 2,500.00 is allowed and reaches step-up | **2,500.01** returns `PA-T03` | Above. In this release 500.00 already triggers step-up, so every authority payment above that figure steps up |
| `PA-D06` customer's own limit | 5,000.00 is allowed with an own limit of 5,000.00 | **5,000.01** returns `PA-D06` | Above, cumulative |
| Own limit against the tier limit | 25,000.00 accepted on the Standard tier | **25,000.01** refused on the limits screen | At or below |
| Cooling off | 11:59 returns `PA-D07` | **12:00** is payable | At or after |
| Investigation note | 19 characters refused | **20** accepted | At least |
| Operations release limit | 5,000.00 may be released by Payment Operations | **5,000.01** may not | Above |
| Sign in lockout | Four failures still allow a sign in | **Five** locks the account | |

---

## 14. Element identification

Prefer `data-testid` over label text or position throughout. The field set on
step 1 changes with the payee mode and on step 2 with the payment type, so
label based identification breaks across those branches.

### Shell, present on every screen inside a system

| Element | `data-testid` |
|---|---|
| Main navigation | `main-nav` |
| Navigation items | `nav-overview`, `nav-new-payment`, `nav-payees`, `nav-standing-orders`, `nav-queue`, `nav-payments`, `nav-applications` |
| Signed in user and role | `current-user` |
| Build string | `build-id` |
| Breadcrumb | `breadcrumb` |
| Current view container | `view` |
| Sign out | `sign-out` |
| Release toggle | `release-toggle` |
| Release banner, when on | `release-banner` |
| KB download | `kb-download` |
| Directives group and link | `directives`, `dir-download` |
| Reset demo data | `reset-data` |
| Toast container and each toast | `toasts`, `toast` |
| Demonstration disclosure | `sidebar-disclosure` |

### Sign in

`login-form`, `input-username`, `input-password`, `btn-signin`, `error-username`,
`error-password`, `login-error`, `demo-credentials`, `demo-disclosure`,
`login-reset`.

### Launcher

`app-online`, `app-sentinel`, `sign-out`, `build-id`.

### Marchmont Online

| Element | `data-testid` |
|---|---|
| Headline figures | `stat-balance`, `stat-limit-used`, `stat-held`, `stat-scheduled` |
| Count tiles, which filter the list | `tile-held`, `tile-scheduled` |
| Active filter notice and its clear control | `payments-filter`, `btn-clear-filter` |
| Accounts grid, and each row | `grid-accounts`, `row-<account id>` |
| Payments grid, each row, status and link | `grid-payments`, `row-<reference>`, `status-<reference>`, `open-<reference>` |
| Empty payments | `payments-empty` |
| Make a payment | `btn-new-payment` |
| Payees grid and status | `grid-payees`, `row-<payee id>`, `payee-status-<payee id>` |
| Wizard progress and step chips | `wizard-progress`, `step-1`, `step-2`, `step-3` |
| Payee mode | `mode-existing`, `mode-new` |
| Step 1 fields | `input-payee`, `input-payee-name`, `input-payee-country`, `input-payee-sort`, `input-payee-number` |
| Step 2 fields | `input-from`, `input-amount`, `kind-immediate`, `kind-scheduled`, `kind-standing-order`, `input-rail`, `input-date`, `input-frequency`, `input-end-date`, `input-reference`, `input-purpose`, `input-source-of-funds` |
| Calendar buttons beside the date fields | `calendar-input-date`, `calendar-input-end-date` |
| Acting for, and the authority summary | `input-acting-for`, `authority-summary`, `authority-type` |
| Payee name check | `cop-outcome`, `cop-code`, `input-cop-confirm`, `error-copConfirmed`, `review-cop` |
| Payment limits screen | `nav-limits`, `limit-bank`, `limit-own`, `limit-effective`, `input-limit`, `btn-save-limit`, `btn-clear-limit`, `error-limit` |
| Payee list | `grid-payees`, `row-<id>`, `payee-status-<id>`, `payee-established-<id>`, `payee-available-<id>`, `btn-add-payee`, `btn-show-removed` |
| Add and amend a payee | `input-new-payee-name`, `input-new-payee-country`, `input-new-payee-sort`, `input-new-payee-number`, `input-new-payee-reference`, `btn-save-payee`, `input-new-payee-cop-confirm`, `input-edit-payee-name`, `input-edit-payee-reference`, `input-edit-payee-cop-confirm`, `edit-sort`, `edit-number`, `btn-save-amend`, `btn-cancel-payee` |
| Payee outcome, and removal | `payee-outcome`, `payee-code`, `payee-cop`, `btn-edit-<id>`, `btn-remove-<id>`, `remove-confirm`, `btn-confirm-remove`, `btn-cancel-remove`, `error-remove`, `remove-code` |
| Authority on a payment and a case | `payment-instructed-by`, `case-instructed-by`, `payment-cop`, `case-cop` |

| Field errors | `error-<field key>`, for example `error-amount`, `error-valueDate`, `error-np-sort` |
| Review | `review`, `review-payee`, `review-amount`, `review-date`, `review-date-moved`, `review-purpose`, `review-source-of-funds`, `indicative-outcome`, `indicative-code` |
| Wizard buttons | `btn-continue`, `btn-back`, `btn-cancel`, `btn-submit` |
| Step-up | `stepup-reason`, `stepup-passcode`, `passcode-value`, `input-passcode`, `error-passcode`, `btn-stepup-confirm`, `btn-stepup-cancel` |
| Payment detail | `payment-outcome`, `outcome-code`, `payment-status`, `payment-release`, `payment-hold`, `payment-decided-by`, `grid-audit` |
| Cancellation | `btn-cancel-payment`, `not-cancellable` |
| Standing orders | `grid-standing-orders`, `standing-orders-empty` |

### Sentinel

| Element | `data-testid` |
|---|---|
| Queue figures | `stat-open`, `stat-mine`, `stat-limit` |
| Queue grid | `grid-queue`, `row-<case reference>`, `code-<case reference>`, `open-<case reference>` |
| Empty queue | `queue-empty` |
| Case | `case-reason`, `case-code`, `case-investigator` |
| Investigation | `input-case-notes`, `error-case-notes`, `btn-save-note` |
| Decisions | `btn-release`, `btn-reject`, `decision-blocked` |
| Enquiry | `filter-search`, `grid-sentinel-payments`, `no-results`, `open-<reference>` |
| Payment in Sentinel | `payment-status`, `outcome-code`, `hold-code`, `open-case`, `decided-by`, `grid-audit` |
| Not found | `not-found` |

---

## 15. Cross system journeys worth testing

1. **Held payment released end to end.** Rose submits 12,000.00 to an
   established payee, passes step-up, lands `Held` under `PH-H02`. Nadia Prasad
   investigates and is refused on identity; Omar Farrell releases. Rose sees
   `Released` and no cancel control.
2. **The identity check.** Nadia investigates a case and is refused. Ends
   blocked - a partial journey by design.
3. **The ownership gate.** Sean Hartley opens a `PH-H01` case and is refused
   before any note is written.
4. **Irrevocability.** A Faster Payment settles and offers no cancel control; a
   CHAPS payment dated tomorrow is `Scheduled` and can be cancelled.
5. **The cut-off, twice.** The same journey against `?now=15:59` and
   `?now=16:30` produces `PA-A01` and `PA-A03` with different value dates.
6. **Precedence.** 30,000.00 returns `PA-D01` rather than `PA-D05`; 5,000.00 to
   a new payee returns `PA-S01` rather than `PA-S02`.
7. **Velocity.** Four payments of 100.00 to the same payee; the first three
   settle and the fourth holds under `PH-H03`.
8. **Limit release.** Cancel a held payment and confirm `stat-limit-used` falls
   by its amount.

---

## 16. What this application deliberately does not do

Stating these prevents requirements being written for behaviour that is not
there.

- **No bank holiday calendar.** Only Saturday and Sunday are non working days.
- **No real authentication.** The passcode is shown on screen and derived from
  the amount.
- **No standing order execution.** A standing order is recorded and shows its
  next date; it never fires and never creates a payment.
- **No scheduled payment execution.** A scheduled payment stays `Scheduled`
  against the fixed clock. Nothing advances it.
- **No currency conversion.** Every account and payment is in sterling, and an
  international payment carries no rate or fee.
- **No inactivity timeout.** A session ends on sign out or a page reload, never
  on its own.
- **A removed payee is never deleted.** There is no screen, and no control,
  that destroys a payee record.
- **Cooling off is not configurable.** It is two hours for every customer
  and every payee, and no screen changes it.
- **An authority cannot be created or amended in the application.** The two
  seeded authorities are fixed, and there is no screen to add a third.
- **The payee name check is a lookup, not a service.** It returns `CP-04` for
  every account number outside its table rather than calling anything.
- **No partial release.** A case is released or rejected in whole.
- **No payee editing or deletion.** Payees are added and never changed.
- **No statements, no transaction history beyond the payment list.**
- **`PA-D02` is unreachable through the interface.** See section 6.
- **The two new fields are collected and not reported anywhere.** `Payment
  purpose` is stored on the payment and shown on its detail and in Sentinel, and
  affects no decision at all. `Source of funds` affects exactly one rule,
  `PH-H06`, and nothing else.

---

## 17. Notes for automation

- **Pin the release.** Open `?release=on` as the first navigation for a journey
  written against this document. Never click `release-toggle` from a journey: it
  is a toggle, correct only from a known starting state.
- **Pin the clock when the cut-off matters.** `?now=16:30`. The setter applies
  at page load. A page load no longer signs the user out, so combine it with
  `?session=out` when the journey needs to sign in itself.
- **Reset, then sign in.** `login-reset` on the sign in screen restores
  everything and clears lockouts in one activation. The daily limit is
  cumulative, so a journey that does not reset inherits the previous run's
  spending and the limit boundary moves.
- **Start a payment from `btn-new-payment`**, not by navigating to the wizard
  route, because a draft survives and the wizard resumes where it was left.
- **Select options by their full exact visible text.** The country list contains
  three entries beginning `United`.
- **Enter dates as literal `dd-mm-yyyy` values from the test data.** The clock is
  fixed, so a date derived from the real today drifts. Type into `input-date` and
  `input-end-date`; do not drive `calendar-input-date` or
  `calendar-input-end-date`. The calendar is a convenience for a person and
  reaches the same result by a route that generated automation handles poorly.
- **Nothing in this application is asynchronous.** Rating, the decision engines
  and the screening are all synchronous. Do not insert a fixed pause; wait on
  the element that proves the step finished.
- **Check the step counter before entering data.** A validation failure keeps the
  user on the current step, and a journey that does not check will enter step 2
  values into step 1 fields.
- **Check whether Acting for exists before using it.** It is absent for a
  customer who holds no authority, so a journey that selects it unconditionally
  fails for Rose Whitfield and succeeds for Daniel Okonjo.
- **A new payee needs the payee check cleared.** Use an account number outside
  the Confirmation of Payee table for a clean new payee, or tick
  `input-cop-confirm` where the check asks for it. A journey written before this
  control existed will stop at step 1.
- **Reset before any journey that touches a limit.** Both the tier limit and the
  customer's own limit count the day's payments cumulatively.
- **A payee added during a journey cannot be paid in the same journey**
  unless the clock is moved past the window with `?now=`. This is the
  commonest way a payee journey fails for a reason that is not a defect.
- **Leaving the add or amend form abandons what was typed.** Returning to
  it gives a blank form, so a journey cannot rely on values surviving a
  trip back to the list.
- **Read the passcode from `passcode-value`.** It differs per amount.
- **Take the base URL, usernames and passwords from environment variables.** The
  sign in screen displays the demo credentials, which makes hard coding them
  tempting; a credential change should be a configuration change rather than an
  edit to every journey.
- **Assert absence, not a disabled state**, for the two four-eyes controls and
  for the cancel control on a settled payment. The controls are not rendered.

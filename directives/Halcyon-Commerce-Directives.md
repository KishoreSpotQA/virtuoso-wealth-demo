# Halcyon Commerce — Touchstone Directives

> Directives to create in Touchstone, **not** a source to upload. Companion to
> `Halcyon-Commerce-KB-Source.md`.

**Applies to:** Halcyon Retail Group, Order to Cash, build 2026.09.24
**Source:** `Halcyon-Commerce-KB-Source.md`
**Application:** `commerce.html` plus its `assets/` folder
**Set at:** project level, before the first generation

---

## How to read this set

Twelve directives. Six are the same wording used on the wealth, insurance and
pensions projects and are marked **portable**. Six are **specific to this
application** and carry what makes it different: three systems rather than one,
forty one decision codes across three engines, a hand off contract between
systems that no other demo has, and a catalogue whose products render in two
different media shapes.

Directives are injected into **every** generation, so the set stays small
deliberately. Set them before the first run; retro-fitting means regenerating.

### Where each one goes

| Target | Directives | Notes |
|---|---|---|
| **Requirement** | 1, 2, 3, 6, 7, 8, 12 | Templates exist here and nowhere else |
| **Journey** | 4, 5, 9, 11 | Instructions and conventions only |
| **Autopilot** | 10 | Instructions and conventions only |

---

## 1. Requirement · Instruction — `One branch per requirement`

**Rewritten for this application.**

```
Produce a separate requirement for each branch of a branching control rather than one requirement listing several outcomes. The branching controls are: the storefront decision code, the promotion outcome, the delivery option available for a destination, the card authorisation outcome, the ERP hold code, the release decision, the pick outcome, the pack outcome, the carrier allocation rule and the delivery outcome.
```

**Fixes:** collapsed coverage. This application carries **forty one decision codes
across three engines**: fifteen EC codes in the storefront, eleven ER codes in the
ERP and fifteen LG codes in the warehouse. A generation that writes "the order is
released or held" as one requirement has covered nothing, and the coverage figure
will say otherwise.

---

## 2. Requirement · Instruction — `Name the system`

**Specific to this application. New.**

```
This application contains three separate systems: the Lumen and Co. storefront, Axion ERP and Vector WMS. Every requirement must name the system it applies to in its scope, and must not mix a storefront code with an ERP code or a warehouse code. Where a requirement covers the hand off between two systems, say so explicitly and name both.
```

**Fixes:** requirements that blur the systems. They share a sign in and a storage
key but nothing else: different screens, different engines, different code prefixes,
different roles. A requirement citing both an EC code and an ER code is testable
against neither.

It also tells generation that the **hand off is itself a thing to be tested**, not
an implementation detail to be skipped over.

---

## 3. Requirement · Instruction — `Decision precedence`

**Specific to this application.**

```
Each of the three engines evaluates its rules in a fixed order and returns the first that matches. Where a requirement covers one code, its test data must not satisfy any earlier code in that engine. Where two codes can be satisfied at once, produce a requirement asserting that the earlier code is the one reported.
```

**Fixes:** the single most likely source of wrong requirements here. Several
conditions overlap and only the earlier one is ever observable:

- A restricted item to Australia with a high fraud score reports **ER-H01**, never ER-H02.
- A registered customer over the credit limit whose address also differs reports
  **ER-H03**, never ER-H04.
- A new customer above 500.00 with a short line reports **ER-H05**, never ER-H06.
- An invalid card number with an expired date reports **EC-C01**, never EC-C02.
- Click and collect wins over the destination test, so a collection order to a
  United Kingdom address reports **LG-C04**, never LG-C02.

Without this directive, generation will write a requirement for ER-H04 with data
that actually produces ER-H03, and the journey will fail for a reason nobody can find.

---

## 4. Journey · Instruction — `Cross system hand off`

**Specific to this application. This is the one that matters most.**

```
The order reference minted on the storefront confirmation page is the key to every other system. A journey that reaches the ERP must read that reference from the confirmation page and use it to locate the sales order, rather than assuming a reference. A journey that reaches the warehouse must take the delivery note number from the ERP fulfilment tab. A journey that ends in one system is a partial journey and must be named as one.
```

**Fixes:** three separate single system journeys pretending to be a chain. The whole
reason this application exists is that a value read in one system has to be carried
into another, which is exactly what a test tool is for and exactly what a static
script cannot do.

Concretely: read `order-reference`, store it, type it into `filter-search` in the
ERP, open `row-<ref>`, read the delivery note from `grid-deliveries`, and finally
search the **order reference again** in `input-track` in the warehouse to close the
loop.

---

## 5. Journey · Instruction — `Role switching`

**Specific to this application.**

```
Roles are partitioned by system, not by seniority. Sales Operations and Credit Control see the ERP only; Warehouse and Supervisor see the warehouse only; the Administrator sees both. A journey that spans the ERP and the warehouse must either sign in as the Administrator throughout, or sign out and sign in again at the boundary. Signing out returns to the launcher, so the journey must re-enter the next system through its launcher card or by navigating to its route.
```

**Fixes:** a journey that walks into a system its user cannot reach and is bounced
back to the launcher with a toast. Two further traps:

- **Sales Operations has a release limit of 5,000.00.** Above it the release button
  is disabled and only Credit Control can proceed. A journey that releases a large
  order as Sales Operations will fail at a disabled button.
- **Cancelling a pick, recording damage and recording a failed delivery are
  supervisor only.** The controls are not rendered at all for a Warehouse Operative,
  so a journey must assert absence, not a disabled state.

---

## 6. Requirement · Instruction — `Advisory versus blocking versus absent`

**Portable, extended for this application.**

```
The source distinguishes controls that block progress, advisory warnings that do not, and options that are withheld entirely. For an advisory control, acceptance criteria must assert both that the warning is displayed and that the user can still continue. Where an option is withheld, assert that the control is absent rather than disabled.
```

**Fixes:** requirements that assert the wrong shape of behaviour. Three traps here:

- **Delivery options are filtered, not disabled.** Outside the United Kingdom,
  `opt-standard` does not exist. A requirement asserting a disabled radio is
  asserting something the application does not do.
- **A bulky basket removes express** and shows an advisory note. The note is
  advisory; the removal is the control.
- **Add to basket is genuinely disabled** on a sold out variant, so that one *is* a
  disabled state assertion. The distinction is deliberate and the source states
  which is which.

---

## 7. Requirement · Instruction — `Exact messages`

**Portable.**

```
Where the source records an exact error, banner or reason text, quote that string verbatim in the acceptance criteria. Reference the field by its label and its data-testid.
```

**Fixes:** paraphrased assertions. The source carries every string the application
renders, including all forty one reason texts. Several are easy to conflate: the
three separate refusals that all carry code LG-K03, and the two different messages
that both carry EC-V02.

---

## 8. Requirement · Instruction — `Boundary cases`

**Portable.**

```
Where the source states a numeric, weight, length or date threshold, produce coverage for the value that just fails and the value that just passes, using the verified boundary values given in the source.
```

**Fixes:** computed boundaries that are wrong. The traps in this application are
arithmetic rather than date arithmetic:

- **Free delivery is judged after the discount.** An 80.00 basket with `LUMEN10`
  falls to 72.00 and is charged 4.95. A requirement assuming it ships free is wrong.
- **VAT is taken out of the gross, not added to the net.** 79.45 gross carries
  13.24 of VAT, not 15.89.
- **Thresholds are strict.** A gross of exactly 500.00 releases; 500.01 holds. A
  gross exactly equal to the credit limit releases. Exactly 1,000.00 scores nothing
  for the order value fraud factor.
- **Parcels round up.** 15.00 kg is one parcel; 15.01 kg is two.

The source states the verified values. This directive tells generation to read them
rather than calculate.

---

## 9. Journey · Instruction — `Start and end state`

**Portable.**

```
Every journey opens by signing in and resetting the demo data, and closes with a verification checkpoint asserting the terminal state: the reference assigned, the decision code reported, or the consignment status reached.
```

**Fixes:** inherited state. Local storage under `halcyon.commerce.v1` is **shared by
all three systems**, and one reset clears all three. Without a reset, a journey
inherits every order a previous run left behind, and any grid assertion that counts
rows is meaningless.

The reset keeps the signed in user, so it can be done immediately after signing in
without signing in twice.

---

## 10. Autopilot · Instruction — `Re-read the form after a revealing control`

**Application specific wording; same idea as the other projects.**

```
Changing a variant select on a product page, changing the country on checkout step 1, choosing click and collect on step 2, or unticking the billing address box on step 3 all add or remove fields on the page. After changing any of them, re-read the form before interacting with further fields. On the basket, the totals update in place without a page re-render; read the figures after the change rather than waiting for a reload.
```

**Fixes:** an agent typing into fields that are no longer there. Four revealing
controls:

- A **variant select** changes the stock line, and can disable the add button.
- **Country** on step 1 changes which delivery options step 2 will offer.
- **Click and collect** reveals a mandatory store select.
- **Unticking the billing box** reveals four more mandatory fields.

The second sentence matters as much. The ERP grid filters and the basket totals
update **in place**, so an agent waiting for a page change waits forever.

---

## 11. Journey · Instruction — `Product media is two shapes, never assert the tag`

**Application specific.**

```
Five of the thirty products show photographs and the other twenty five show inline vector artwork, and both sit inside the same containers and the same test ids. Never write a step that requires the product image to be an img element, and never require a fixed number of thumbnails: most products have three views and two have four. To assert that an image really loaded rather than showing a broken icon, assert that its natural width is greater than zero, or read IMG_FAILURES in the console and assert it is empty. Below 760 pixels the image column in the confirmation and order summary tables is hidden, so index those cells by header text, never by column position.
```

**Fixes:** the three ways a journey breaks on media. A step written against
`img` passes on HM-1001 and fails on five sixths of the catalogue. A step that
clicks `thumb-3` passes on HM-1005 and HM-1008 and fails on every other product. And a
step that reads "the third cell" of a confirmation line passes on a desktop
viewport and reads the wrong column on a phone.

`IMG_FAILURES` is worth knowing about on its own: it is an array of every SKU
that fell back to artwork because its file failed to load. Empty means the asset
folder deployed cleanly. It is the cheapest possible deployment check.

---

## 12. Requirement · Template

**Portable.**

```
Scope / System / Rationale / Pre-conditions / Test data / Acceptance criteria / Branches not covered / Source references
```

**Fixes:** requirements that look complete but say nothing about their limits.
**System** is added for this project because the same verb means different things in
each of the three. **Branches not covered** is the non-obvious one: with forty one
codes under precedence across three engines, it is what turns the requirement set
into a coverage argument rather than a list.

---

## Setup

1. Create the Requirement directives first, then Journey, then Autopilot.
2. Check for **name collisions** with organisation level directives and with the
   Cavendish projects. A project directive of the same name **replaces** the
   organisation one; a differently named one stacks alongside it.
   `One branch per requirement`, `Decision precedence` and `Boundary cases` are
   deliberately reused names carrying different text. That only works if they are
   set at project level.
3. Run **Preview prompt** on each and read what is actually injected.
4. Confirm each is **saved and enabled**. A saved but disabled directive is the
   commonest cause of output that ignores the house style, and it looks identical to
   one that is working.
5. Only then generate.

## If you only set three

`One branch per requirement`, `Decision precedence`, `Cross system hand off`. Those
three carry nearly all the value. `Name the system` is the fourth, and the first one
to add.

## Verifying they landed

| Directive | What to look for in the output |
|---|---|
| One branch per requirement | More than one requirement per branching control; the decision codes alone should produce well over thirty |
| Name the system | Every requirement names one of the three systems, and none cites both an EC and an ER code |
| Decision precedence | At least one requirement asserts that an earlier code is reported where two conditions are met |
| Cross system hand off | The end to end journey reads the order reference on the confirmation page and searches for it in the ERP |
| Role switching | The end to end journey either uses the Administrator throughout or signs out at the system boundary and re-enters through the launcher |
| Advisory versus blocking versus absent | A requirement asserts that `opt-standard` is **absent** for a non United Kingdom destination |
| Exact messages | Acceptance criteria contain quoted strings, and the three LG-K03 messages are not conflated |
| Boundary cases | The 500.00 / 500.01 pair appears, and at least one requirement covers free delivery being judged after the discount |
| Product media is two shapes | No step asserts that a product image is an `img` element, and no step clicks `thumb-3` outside HM-1005 and HM-1008 |
| Start and end state | Journeys open with a demo data reset |
| Re-read the form | Journeys re-read after changing the delivery country |
| Template | Every requirement carries a **System** line and a **Branches not covered** section |

---

## Two caveats worth keeping in mind

Directives are **guidance applied by an analysis step, not rules enforced by an
engine**. They shape generation reliably in practice but are not guaranteed. Verify
on the output that matters, which is what the table above is for.

Generation is also **non-deterministic**: the same source and prompt can produce
different output on two runs. Treat any single run as a starting point to refine, and
never promise a customer identical output twice.

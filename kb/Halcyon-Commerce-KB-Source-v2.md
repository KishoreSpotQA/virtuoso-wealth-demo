# Halcyon Retail Group - Order to Cash

**Knowledge Base source for Touchstone**
**Application:** `commerce.html` plus its `assets/` folder
**Application URL:** `https://kishorespotqa.github.io/virtuoso-wealth-demo/commerce.html`
**Build:** 2026.09.29
**Release:** 2.1
**Systems covered:** Halcyon Market Storefront, Axion ERP, Vector WMS
**Status:** Current - describes the application as deployed

This is the complete source. Upload this one file; there is nothing else to add
for this application.

### What is in here

| Section | Covers | System |
|---|---|---|
| 1 | What the application is, and the reference chain that joins the three systems | all three |
| 2 | The five sign in accounts, roles and what each may reach | all three |
| 3 | Conventions that hold on every screen, and the reference number formats | all three |
| 4 | The thirty product catalogue, variants, media, browse, filters and product detail | Storefront |
| 5 | Basket and promotion codes | Storefront |
| 6 | The three step checkout | Storefront |
| 7 | **EC decision codes**, fifteen of them | Storefront |
| 8 | The confirmation page and the hand off into the ERP | Storefront to ERP |
| 9 to 10 | Sales order enquiry and order detail | Axion ERP |
| 11 | The hold engine and its **ER decision codes**, eleven of them | Axion ERP |
| 12 | Release, partial release and cancellation | Axion ERP |
| 13 to 15 | Picking, packing, despatch, returns and the **LG decision codes**, fifteen of them | Vector WMS |
| 16 | Boundary values read from the running application | all three |
| 17 | Cross system journeys worth testing | all three |
| 18 | What the application deliberately does not do | all three |
| 19 | Behaviour confirmed during release validation | all three |

**Forty one decision codes in total**, across three separate engines. The prefix
names the engine: `EC` is the storefront, `ER` is the ERP, `LG` is the warehouse.
A requirement must never cite codes from two engines at once.

---

## 0. Release 2.1

This document describes Halcyon Commerce with **New features enabled**. Release 2.1 adds two fields at checkout, adds one hold, and **lowers the fraud review threshold from 70 to 60**.

| # | Change | Kind |
|---|---|---|
| 1 | Requested delivery date, validated against the lead time of the method chosen | New field |
| 2 | Purchase order reference, with ER-H07 where an account order above 1,000.00 has none | New field, **new hold** |
| 3 | Fraud review threshold lowered from 70 to 60 | **Changes an existing rule** |

### The release control

A **New features** control in the top bar, `data-testid="release-toggle"`, present on the launcher and inside both back-office systems, **off by default**. A banner with `data-testid="release-banner"` shows while it is on; each new field carries `data-testid="rel-new"`.

Unlike the Cavendish shell, the release here is **platform-wide**: the storefront, Axion ERP and Vector WMS are one application and switch together.

A journey pins the release on its opening navigation rather than clicking the control, which is a toggle and therefore only correct from a known starting state:

```
commerce.html?release=on#/shop      release on
commerce.html?release=off#/shop     release off
```

The setter is absolute and idempotent. Any value other than `on` or `off` is ignored.

> **A journey must never click `release-toggle` or `btn-reset`.** Both change the rules underneath a run.

### An order keeps the thresholds it was placed under

Each order is stamped with its release and assessed on those thresholds permanently. Lowering the fraud threshold does not re-assess an order already in the warehouse.

**Download KB source** follows the release: with it on the control reads **Download KB source v2** and serves this file.

---

## 1. What this application is

Halcyon Retail Group is a fictitious retailer. The application is a single page that
contains **three separate systems** joined by one order:

| System | What it is | Who uses it |
|---|---|---|
| **Halcyon Market Storefront** | A direct to consumer store. Browse, basket, checkout, pay. | The customer |
| **Axion ERP** | Sales order processing. Holds, release decisions, allocation. | Sales Operations, Credit Control |
| **Vector WMS** | Warehouse and transport. Pick, pack, despatch, tracking, returns. | Warehouse, Logistics Supervisor |

The point of the application is the **chain between the three**. One order placed in
the storefront becomes a sales order in the ERP and a delivery note in the warehouse,
and each step mints a reference that the next system must be able to find:

```
Storefront                Axion ERP                 Vector WMS
------------------        ---------------------     ----------------------
ORD-2026-00147       ->    sales order 00147    ->    DN-00147-1
(order reference)         (hold / release)          (delivery note)
                                                    BRK123456789
                                                    (carrier tracking)
```

A journey that only exercises one system is a partial journey. The value of this
application to a test tool is the **hand off**: a value read on one screen must be
carried into a different system and asserted there.

### Deliberate design decisions

- **No network beyond its own folder.** All logic runs in the page. The only
  requests the application makes are for its own product photographs, at relative
  paths under `assets/products/`. It never contacts an external host.
- **Two kinds of product media.** Five products render **photographs**; the other
  twenty five render **inline SVG artwork**. Which is which is fixed and listed in
  section 4. Automation must not assert on the tag name.
- **Fixed clock.** The system date is always **22 September 2026, 09:00 UTC**
  (a Tuesday). Every promised delivery date and every expiry check derives from it,
  so boundary values do not drift between runs.
- **Local storage only**, under the key `halcyon.commerce.v1`. All three systems
  share it, so one reset clears all three.
- **Every control carries a `data-testid`.**
- **Every field is written on both the `input` and the `change` event.** A WebDriver
  select by visible text fires `change` only; typing in a text box fires `input` only.
  Both are handled, so a value never disappears when the page re-renders.
- **No en dashes or em dashes anywhere**, including inside `<option>` text, so that
  select by visible text always matches.

---

## 2. Signing in, roles and access

The sign in screen is at the root. All five accounts use the password
`Halcyon#2026`.

| Username | Name | Role | ERP | WMS | Storefront | Release limit |
|---|---|---|---|---|---|---|
| `amara.morgan` | Amara Morgan | Sales Operations | yes | no | yes | GBP 5,000.00 |
| `ravi.chandra` | Ravi Chandra | Credit Control | yes | no | yes | none |
| `tolu.oyelaran` | Tolu Oyelaran | Warehouse Operative | no | yes | yes | not applicable |
| `stefan.novak` | Stefan Novak | Logistics Supervisor | no | yes | yes | not applicable |
| `karen.bell` | Karen Bell | Platform Administrator | yes | yes | yes | none |

**The storefront is open to every signed in user.** It represents the customer facing
site; signing in at the launcher is a demonstration convenience, not a shop account.
The customer is identified at checkout by the **email address entered**, not by who
is signed in. This matters: the same signed in user can place an order as any
customer on the register simply by typing that customer's email address.

**Failed sign in.** `input-username` and `input-password` keep what was typed when
the page re-renders. An unrecognised username or wrong password shows
`login-error` with the text *"Username or password not recognised."* There is no
account lockout in this application.

**Access control.** Navigating to a system the role does not hold shows a toast
*"Your role does not have access to the ERP."* or *"Your role does not have access to
the warehouse system."* and returns to the launcher. The launcher card for a system
the role does not hold still exists and is still clickable; it shows the badge
*"No access for this role"*.

**Signing out** always returns to the sign in screen, and signing back in lands on
the **launcher** at `#/apps`, never on the screen where the previous session ended.

---

## 3. Conventions that apply everywhere

| Convention | Detail |
|---|---|
| **Routing** | Hash routes. `#/apps`, `#/shop`, `#/erp/orders`, `#/wms/work` and so on. Deep links work. |
| **Dates on screen** | Always `dd-mm-yyyy`. Stored internally as ISO. |
| **Date and time stamps** | `dd-mm-yyyy hh:mm` in UTC. |
| **Money** | `GBP 1234.56` in prose and detail panels. In the ERP grid columns the thousands separator appears: `1,284.95`. |
| **Reset** | The button `btn-reset` on the launcher (and the `F9 Reset demo data` function key in the ERP) clears **all orders, baskets and checkouts in all three systems**. It keeps the signed in user and the colour scheme, so a journey can reset immediately after signing in without signing in twice. |
| **Colour scheme** | A light and a dark palette, toggled with `btn-theme`. All screens are usable in both. |
| **Toasts** | One at a time. A new toast replaces the previous one, so `[data-testid="toast"]` never matches more than one element. |
| **Double click guard** | A state changing click re-renders the page, so the second click of a double click would otherwise land on whichever control moved into that position. The second click of a double is ignored for the state changing actions listed in section 19. Independent single clicks are never affected. |
| **Same URL navigation** | Navigating to the hash route the browser is already on is a no-op at browser level and does not re-render. A journey that wants a fresh render of the current route must reload or use a control on the page. |

### Reference number formats

| Reference | Format | First value after a reset | Where it is minted |
|---|---|---|---|
| Order | `ORD-2026-nnnnn` | **ORD-2026-00147** | Storefront, on successful payment |
| Delivery note | `DN-nnnnn-n` | `DN-00147-1` | ERP, on allocation |
| Return | `RMA-nnnn` | `RMA-0001` | WMS, when a return is raised |
| Tracking | carrier prefix plus 9 digits, for example `BRK123456789` | varies | WMS, on despatch |

The delivery note number embeds the order number, and the suffix counts deliveries
against that order. This is the join between the ERP and the warehouse.

---

## 4. Storefront: browse, filter and product detail

### The catalogue

**Thirty products across six categories, five in each.** Every brand and product
name is invented; nothing is modelled on a real product or retailer.

### Product media

Two kinds, and **which one you get depends on the colour selected**, not on the
product alone. A photograph exists in one colour only; every other colour of
that product shows the vector artwork, recoloured to the chosen swatch.

| Product | Photographed colour | Views | Any other colour |
|---|---|---|---|
| HM-1001 Oxford Cotton Shirt | **White** (the default) | 3 | recoloured artwork |
| HM-1002 Merino Crew Neck Jumper | **Charcoal** (the default) | 3 | recoloured artwork |
| HM-1005 Linen Shirt Dress | **Sage** (the default) | 4 | recoloured artwork |
| HM-1008 Suede Penny Loafer | **Navy** (the default) | 4 | recoloured artwork |
| HM-1021 Hand Knotted Wool Rug | **Natural** (the default) | 3 | recoloured artwork |
| the other twenty five | none | 3 | recoloured artwork |

A photograph renders as `<img class="prod-art">` with
`src="assets/products/<sku>/<n>.jpg"`, 675x900 and descriptive `alt`. Artwork
renders as an inline `<svg class="prod-art">`.

**The picture always agrees with the swatch.** Choosing a colour changes the
image on every screen that shows one: the listing row, the product hero and its
thumbnail strip, the basket line, the checkout summary and the confirmation.
Two lines of the same product in different colours render different pictures in
the same basket.

Rules that matter for automation:

- **Do not assert on the tag name.** Both sit inside the same containers
  (`pdp-image`, `img-<sku>`, `thumb-<n>`), so every locator is unchanged. A step
  written against `img` will fail on five sixths of the catalogue, and will also
  fail on a photographed product once a different colour is chosen.
- **Choosing a colour is assertable.** Twenty five products carry a Colour or
  Finish axis, drawn as swatches. Selecting a different value changes the
  rendered image every time, so a journey can assert that the picture responded
  rather than only that the label did.
- **The thumbnail count is three or four, never a fixed number.** `thumb-0`,
  `thumb-1` and `thumb-2` exist on every product. `thumb-3` exists only on
  HM-1005 and HM-1008.
- **To prove an image is real, not a broken icon**, assert `naturalWidth > 0` on
  the `img`, or evaluate `IMG_FAILURES` in the console and assert it is empty.
  That array names every product that has fallen back to artwork because its file
  failed to load, so it is also the fastest check that a deployment copied the
  asset folder correctly.
- **No waiting is needed for images.** Rendering is synchronous and every element
  is in the DOM before the image bytes arrive. Wait on elements, never on time.
- **The image column in the confirmation and order summary tables is hidden below
  760 pixels.** On a phone viewport those tables have one fewer column, so index
  cells by header text or from the right, not by position.

Photographs come from a local dataset supplied by the project owner, resized and
re-encoded. Their provenance and the open question over usage rights are recorded
in `IMAGE-CREDITS.md`. The twenty five products without photographs have none by
decision, not by error: twenty have no suitable subject in the dataset, and five
were dropped because every available view carried a third-party brand label,
badge or borrowed marketing copy that does not belong on a product page for an
invented brand.

| Category | Products |
|---|---|
| Fashion | HM-1001 to HM-1005 |
| Footwear | HM-1006 to HM-1010 |
| Mobiles | HM-1011 to HM-1015 |
| Electronics | HM-1016 to HM-1020 |
| Home | HM-1021 to HM-1025 |
| Kitchen | HM-1026 to HM-1030 |

**Eleven brands:** Ashgrove, Aurio, Coppersmith, Corvid, Fernway, Halcyon Home,
Lumenworks, Northvale, Stridewell, Trailhead, Zenith.

**174 variants in total**, built as the cartesian product of each product's
axes. A variant id is the axis codes joined by a hyphen, so Midnight with 256GB
is `MID-256` and Slate in King is `SLA-KNG`. A product with no axes has the
single variant `STD`.

| SKU | Name | Brand | Category | Price | Was | Rating | Reviews | Weight | Flags | Axes |
|---|---|---|---|---|---|---|---|---|---|---|
| HM-1001 | Oxford Cotton Shirt | Northvale | Fashion | 48.00 | | 4.4 | 1,284 | 0.50 | | Colour, Size |
| HM-1002 | Merino Crew Neck Jumper | Northvale | Fashion | 72.00 | 90.00 | 4.6 | 842 | 0.60 | | Colour, Size |
| HM-1003 | Cotton T Shirt Three Pack | Ashgrove | Fashion | 26.00 | | 4.2 | 3,109 | 0.45 | | Colour, Size |
| HM-1004 | Quilted Field Jacket | Ashgrove | Fashion | 129.00 | | 4.5 | 517 | 1.10 | | Colour, Size |
| HM-1005 | Linen Shirt Dress | Northvale | Fashion | 89.00 | | 4.3 | 396 | 0.55 | | Colour, Size |
| HM-1006 | Fell Trail Running Shoe | Trailhead | Footwear | 115.00 | | 4.7 | 2,244 | 0.90 | | Colour, Size |
| HM-1007 | Leather Chelsea Boot | Stridewell | Footwear | 145.00 | | 4.4 | 688 | 1.20 | | Colour, Size |
| HM-1008 | Suede Penny Loafer | Stridewell | Footwear | 98.00 | 120.00 | 4.1 | 302 | 0.85 | | Colour, Size |
| HM-1009 | Canvas Plimsoll | Trailhead | Footwear | 42.00 | | 4.0 | 1,517 | 0.70 | | Colour, Size |
| HM-1010 | Waterproof Hiking Boot | Trailhead | Footwear | 168.00 | | 4.6 | 921 | 1.60 | | Colour, Size |
| HM-1011 | Pulse 7 Smartphone | Zenith | Mobiles | 649.00 | | 4.5 | 4,872 | 0.30 | **restricted** | Colour, Storage |
| HM-1012 | Pulse 7 Pro Smartphone | Zenith | Mobiles | 949.00 | | 4.7 | 2,110 | 0.32 | **restricted** | Colour, Storage |
| HM-1013 | Lite 5 Smartphone | Corvid | Mobiles | 229.00 | 269.00 | 3.9 | 6,430 | 0.29 | **restricted** | Colour, Storage |
| HM-1014 | Tab 11 Tablet | Zenith | Mobiles | 399.00 | | 4.4 | 1,033 | 0.60 | **restricted** | Colour, Storage |
| HM-1015 | Rugged Phone Case | Corvid | Mobiles | 24.00 | | 4.2 | 2,905 | 0.10 | | Colour |
| HM-1016 | Studio Over Ear Headphones | Aurio | Electronics | 199.00 | | 4.6 | 3,781 | 0.35 | **restricted** | Colour |
| HM-1017 | Air Wireless Earbuds | Aurio | Electronics | 89.00 | 109.00 | 4.3 | 8,204 | 0.08 | **restricted** | Colour |
| HM-1018 | Meridian 14 Laptop | Lumenworks | Electronics | 749.00 | | 4.4 | 1,466 | 1.60 | **restricted** | Colour, Memory |
| HM-1019 | Compact Bluetooth Speaker | Aurio | Electronics | 59.00 | | 4.2 | 5,120 | 0.70 | **restricted** | Colour |
| HM-1020 | Pace Smart Watch | Lumenworks | Electronics | 179.00 | | 4.1 | 2,673 | 0.12 | **restricted** | Colour, Size |
| HM-1021 | Hand Knotted Wool Rug | Fernway | Home | 320.00 | | 4.5 | 412 | 16.50 | **bulky** | Colour, Size |
| HM-1022 | Washed Linen Bedding Set | Halcyon Home | Home | 145.00 | | 4.6 | 1,890 | 3.10 | | Colour, Size |
| HM-1023 | Oak Bedside Table | Fernway | Home | 240.00 | | 4.3 | 228 | 19.00 | **bulky** | Finish |
| HM-1024 | Cashmere Throw | Halcyon Home | Home | 165.00 | | 4.7 | 654 | 1.20 | | Colour |
| HM-1025 | Aurora Table Lamp | Fernway | Home | 89.00 | | 4.2 | 771 | 2.40 | | Finish |
| HM-1026 | Copper Pour Over Kettle | Coppersmith | Kitchen | 74.50 | | 4.5 | 1,204 | 1.10 | | none |
| HM-1027 | Stoneware Dinner Set | Coppersmith | Kitchen | 118.00 | | 4.4 | 883 | 6.80 | | Size |
| HM-1028 | Walnut Serving Board | Fernway | Kitchen | 42.00 | | 4.6 | 1,655 | 1.80 | | Size |
| HM-1029 | Espresso Glass Set | Coppersmith | Kitchen | 29.00 | | 4.3 | 2,318 | 1.00 | | none |
| HM-1030 | Ceramic Vase Trio | Fernway | Kitchen | 56.00 | | 4.0 | 497 | 2.20 | | none |

**Restricted** means the product contains a lithium ion battery, which is what
triggers the export control hold. Nine products carry it.
**Bulky** removes express delivery from the options offered. Two products carry it.
**Four products carry a was price**: HM-1002 (save 20 per cent), HM-1008 (18),
HM-1013 (15) and HM-1017 (18). The saving is calculated, not stored.

### Browse, `#/shop`

| Control | `data-testid` | Behaviour |
|---|---|---|
| Keyword search | `input-search`, `btn-search` | Matches name, brand and category. **Applied only when Search is pressed**, never while typing, so the caret is never lost. Trimmed and case insensitive. |
| Category rail | `cat-all`, `cat-fashion`, `cat-footwear`, `cat-mobiles`, `cat-electronics`, `cat-home`, `cat-kitchen` | Buttons, not a select. The active one carries `aria-current="true"`. |
| Brand facet | `check-brand-northvale` and so on, one per brand | Checkboxes. Several brands are an **OR**. |
| Price facet | `radio-price-all`, `radio-price-b1` to `radio-price-b5` | Any price, under 50.00, 50.00 to 100.00, 100.00 to 200.00, 200.00 to 500.00, over 500.00. |
| Rating facet | `radio-rating-all`, `radio-rating-r4`, `radio-rating-r3`, `radio-rating-r2` | Four, three or two stars and up. |
| Availability | `check-instock` | In stock only. |
| Clear all | `btn-clear-filters` | **Rendered only while a facet is active.** Clears the facets and the keyword and **keeps the chosen sort order**, because sort is not a filter. |
| Sort | `select-sort` | Featured, price low to high, price high to low, average customer review, newest arrivals. |
| Result count | `result-count` | `Showing 1 to 12 of 30 results`, and `for <keyword>` when a search is applied. |
| Result rows | `row-HM-1001` | One per product. |
| Empty state | `no-results` with `btn-clear-empty` | |

**Facets AND together, values within a facet OR.** Zenith alone is 3 results;
Zenith plus Aurio is 6; Zenith plus Aurio plus the over 500.00 band is 2.

### Pagination

Twelve results per page, so the unfiltered catalogue is **three pages**.
`btn-page-1`, `btn-page-2`, `btn-page-3`, plus `btn-page-prev` and
`btn-page-next`. Previous is disabled on the first page and Next on the last.
**Changing any facet returns to page one**, so narrowing the results while on a
later page never strands the user on an empty page.

### A result row

Each row carries the product image as a button, the brand, the title as a
button, the star rating, the price, and two actions.

| Element | `data-testid` |
|---|---|
| Image, opens the product | `img-HM-1001` |
| Title, opens the product | `link-HM-1001` |
| Rating value | `rating-HM-1001` |
| Price | `price-HM-1001` |
| Add to Basket, straight from the row | `btn-add-HM-1001` |
| See options, opens the product | `btn-options-HM-1001` |

**Adding from a row takes the first variant that has stock, quantity one.** It
does not open the product. This is a second entry point into the basket and
behaves identically to the one on the product page.

### Product detail, `#/shop/product/<sku>`

| Control | `data-testid` | Notes |
|---|---|---|
| Breadcrumbs | `breadcrumbs`, `crumb-home`, `crumb-cat` | The category crumb returns to the listing filtered to that category |
| Image | `pdp-image` | |
| Thumbnail strip | `thumb-strip`, `thumb-0`, `thumb-1`, `thumb-2`, and `thumb-3` on two products | **Three views on twenty eight products, four on HM-1005 and HM-1008.** Never assume a fixed count. |
| Rating | `pdp-rating`, `pdp-reviews` | |
| Price | `pdp-price`, and `pdp-saving` where there is a was price | |
| Feature bullets | `pdp-bullets` | |
| **Colour or Finish** | `swatch-white`, `swatch-midnight` and so on | **Swatch buttons, not a select.** The chosen value is named in `chosen-colour` or `chosen-finish`. |
| Every other axis | `select-size`, `select-storage`, `select-memory` | A select, named after the axis in lower case |
| Stock | `pdp-stock` | `In stock`, `Only N left` at five or fewer, `Out of stock` at zero |
| Delivery promise | `pdp-eta` | |
| Quantity | `input-qty`, `btn-qty-minus`, `btn-qty-plus` | 1 to 10 |
| Add to Basket | `btn-add-basket` | Disabled when the chosen variant has no stock |
| Buy Now | `btn-buy-now` | Adds one and goes **straight to the checkout** |
| Error | `pdp-error` | Carries the decision code |

**Two different interaction types on purpose.** Colour and Finish are swatch
buttons, which a tool drives with a click; every other axis is a select, which a
tool drives by visible text. A journey needs both.

**Changing any axis re-renders the panel**, so the stock line, the price and the
state of the two buttons all follow the chosen combination. Re-read the page
after changing one.

### Stock, and why the ERP sees a different number

The storefront shows an *available to promise* figure refreshed on a cycle. The
ERP holds the live allocatable figure. Five variants diverge on purpose, and
ordering more than the allocatable quantity of one of them passes the storefront
and is held in the ERP. This is the only way the stock hold is reachable.

| SKU | Variant | Axis values | Web stock | Allocatable |
|---|---|---|---|---|
| HM-1002 | `CHR-L` | Charcoal, L | 6 | **2** |
| HM-1011 | `MID-256` | Midnight, 256GB | 4 | **1** |
| HM-1021 | `NAT-LG` | Natural, 200 by 300 cm | 2 | **0** |
| HM-1022 | `SLA-KNG` | Slate, King | 6 | **2** |
| HM-1025 | `CHM` | Polished Chrome | 3 | **1** |

**Two variants have no stock at all:** HM-1006 `BGY-S11` (Black Grey, size 11)
and HM-1022 `SGE-KNG` (Sage, King). Every other variant carries its product's
default stock.

**A scarcity tag on a result row is derived, not stored.** `Out of stock` when
every variant is at zero, which never happens on the seeded data. `Low stock`
when any variant is at zero or between one and five. Otherwise, where there is
a was price, `Save N per cent`.


## 5. Storefront: basket and promotions

`#/shop/basket`.

| Control | `data-testid` | Notes |
|---|---|---|
| Line quantity | `input-qty-<sku>-<vid>` | With `btn-minus-...` and `btn-plus-...` |
| Remove | `btn-remove-<sku>-<vid>` | |
| Promotion code | `input-promo`, `btn-apply-promo` | |
| Promotion message | `promo-message` | Carries the decision code |
| Totals | `total-net`, `total-discount`, `total-gross` | |
| Go to checkout | `btn-checkout` | |
| Empty state | `basket-empty` | |

### Promotion codes

| Code | Effect | Minimum spend | Valid to | State |
|---|---|---|---|---|
| `HALCYON10` | 10 per cent off the order | 50.00 | 31-12-2026 | live |
| `FREESHIP` | Free standard delivery | 75.00 | 31-12-2026 | live |
| `WELCOME25` | 25.00 off the order | 200.00 | 31-12-2026 | live |
| `SPRING20` | 20 per cent off | none | **30-04-2026** | **expired** |

Codes are matched case insensitively and trimmed. Only one code applies at a time.

---

## 6. Storefront: checkout

`#/shop/checkout`. Three steps, tracked by `step-1`, `step-2` and `step-3`.
`btn-continue` moves forward, `btn-back` moves back. **Values typed on one step
survive moving forward and back.**

### Step 1, contact and delivery address

| Field | `data-testid` | Required | Message when missing |
|---|---|---|---|
| Email address | `input-email` | yes | `Enter your email address.` / `Enter a valid email address.` |
| Telephone | `input-phone` | yes | `Enter a telephone number.` |
| First name | `input-firstname` | yes | `Enter your first name.` |
| Last name | `input-lastname` | yes | `Enter your last name.` |
| Address line 1 | `input-line1` | yes | `Enter the first line of the address.` |
| Address line 2 | `input-line2` | no | |
| Town or city | `input-city` | yes | `Enter a town or city.` |
| Postcode | `input-postcode` | yes | `Enter a postcode.` |
| Country | `select-country` | yes | defaults to United Kingdom |

Errors appear as `error-<field>`, for example `error-email`.

**Changing the country re-renders the page and changes which delivery options step 2
will offer.** It is a revealing control.

### Countries and zones

| Zone | Countries |
|---|---|
| UK | United Kingdom |
| EU | Ireland, France, Germany, Spain, Netherlands |
| ROW | Norway, Switzerland, United States, Canada, Australia, Singapore |

### Step 2, delivery method

| Option | `data-testid` | Price | Free over | Zones | Bulky allowed | Promised date from the fixed clock |
|---|---|---|---|---|---|---|
| Standard delivery | `opt-standard` / `radio-standard` | 4.95 | 75.00 | UK | yes | **29-09-2026** |
| Express delivery | `opt-express` / `radio-express` | 9.95 | never | UK | **no** | **23-09-2026** |
| Click and collect | `opt-collect` / `radio-collect` | 0.00 | not applicable | UK | yes | **24-09-2026** |
| International standard | `opt-intl` / `radio-intl` | 19.95 | never | EU and ROW | yes | **06-10-2026** |

Promised dates count **working days** from 22-09-2026 and skip Saturday and Sunday.

**Options are filtered, not disabled.** An option that does not apply is not rendered
at all. A test that asserts a disabled radio is asserting something the application
does not do.

- Outside the United Kingdom, only `opt-intl` exists.
- With a bulky item in the basket, `opt-express` does not exist and the note
  `bulky-note` appears: *"Your basket contains a bulky item, so express delivery is
  not offered."*
- If no option applies at all, `no-delivery` is shown.

**Click and collect reveals a store select**, `select-store`, which is mandatory:
`Choose a collection store.` The stores are London Marylebone, Manchester Deansgate,
Bristol Clifton and Edinburgh Stockbridge.

**Changing the destination clears a method that destination does not serve.**
If express delivery is chosen for a United Kingdom address and the country is then
changed to the United States, the stored method and any collection store are
cleared, the delivery line in the summary returns to `Not chosen`, and the banner
`checkout-banner` explains why. The customer must choose again. Without this the
summary would show a charge for a method that could not be used.

### Step 3, payment

| Field | `data-testid` | Notes |
|---|---|---|
| Name on card | `input-cardname` | required |
| Card number | `input-cardnumber` | required, check digit validated |
| Expiry month | `select-expmonth` | `01` to `12` |
| Expiry year | `select-expyear` | `2024`, `2025` then `2026` to `2034` |
| Security code | `input-cvc` | 3 or 4 digits |
| Billing same as delivery | `check-billingsame` | ticked by default |
| Billing address | `billing-panel`, `input-bline1`, `input-bcity`, `input-bpostcode`, `select-bcountry` | revealed when the box is **unticked** |
| Verification code | `verify-panel`, `input-verify` | revealed only after a step up is demanded |
| Terms of sale | `check-terms` | required |
| Marketing | `check-marketing` | optional |
| Pay | `btn-place-order` | label carries the total, for example `Pay GBP 79.45` |

**Unticking the billing checkbox reveals a panel of four more fields.** It is a
revealing control and the page must be re-read after it.

### Demonstration cards

No real card is ever accepted or stored. The panel `payment-note` states these on
screen.

| Number | Behaviour |
|---|---|
| `4111111111111111` | Approved |
| `4000000000000002` | Declined by the issuer |
| `4000000000003220` | Additional verification demanded, then approved with the code **246813** |
| Any other number passing the check digit test | Approved |

---

### Delivery step fields — release 2.1

Both sit under the delivery method options on step 2 of checkout, and both are **optional**.

| Field | Control | `data-testid` | Container | Validation | Exact error message |
|---|---|---|---|---|---|
| Requested delivery date | Text, `dd-mm-yyyy` | `input-requestedDate` | `requested-date-field` | Not earlier than the estimate for the method chosen | `Enter the date as dd-mm-yyyy.` · `The earliest date for this delivery method is {date}.` |
| Purchase order reference | Text | `input-poReference` | `po-reference-field` | `PO-` followed by exactly five digits | `Enter the reference as PO- followed by five digits.` |

`PO-00001` passes. `PO-1234` (four digits), `PO-123456` (six) and `po-00001` (lower case) all fail.

Leaving either blank is permitted at checkout. The consequence of leaving the purchase order blank is a **hold at intake**, not a validation error — see the hold engine.

Both values are carried onto the order: `order.poReference`, and `order.requestedDate` as an ISO date.

## 7. Storefront decision codes, EC

Every outcome carries a code. The banner is `checkout-banner`, the product page
error is `pdp-error`, the promotion message is `promo-message`.

| Code | Condition | Exact message |
|---|---|---|
| **EC-V01** | Checkout attempted with an empty basket | `Your basket is empty.` |
| **EC-V02** | Quantity requested is above the web stock for that variant | `Only N of that option are available.` |
| **EC-V02** | The chosen combination of axes is not made | `That combination is not available.` |
| **EC-V03** | Requested quantity of one item would exceed 10 | `A maximum of 10 units of one item may be bought in a single order.` |
| **EC-P01** | Promotion code not on the list | `That promotion code is not recognised.` |
| **EC-P02** | Promotion code valid to date is in the past | `That promotion ended on 30-04-2026.` |
| **EC-P03** | Basket net below the minimum spend | `Spend GBP 50.00 to use this code. Your basket is GBP 38.00.` |
| **EC-P04** | Promotion accepted | `10 per cent off your order applied.` |
| **EC-D01** | A chosen delivery method is not offered for the destination, so it is cleared | `Delivery to United States uses different options, so please choose again.` |
| **EC-D02** | Bulky item in the basket, express withheld | `Your basket contains a bulky item, so express delivery is not offered.` |
| **EC-C01** | Card number fails the check digit test | `That card number is not valid.` |
| **EC-C02** | Card expiry is before the system date | `That card expired in 01-2025.` |
| **EC-C03** | Issuer declines | `The payment was declined by the card issuer.` |
| **EC-C04** | Issuer demands a step up | `Your bank has asked for additional verification.` |
| **EC-C05** | Verification code wrong | `That verification code is not correct.` |
| **EC-A01** | Order placed | The confirmation page is shown |

### Order of evaluation at payment

The fields are validated first, then the card is authorised. Within authorisation
the order is:

1. Check digit, **EC-C01**
2. Expiry against the system date, **EC-C02**
3. Issuer behaviour, **EC-C03** or **EC-C04**
4. Verification code, **EC-C05**
5. **EC-A01**

An invalid number with an expired date reports **EC-C01**, not EC-C02.

---

## 8. The confirmation page and the hand off

`#/shop/confirm/<ref>`.

| Element | `data-testid` | Notes |
|---|---|---|
| Order reference | `order-reference` | **The value the ERP journey needs** |
| Line rows | `confirm-line-<sku>` | |
| Total paid | `confirm-total` | |
| Delivery method | `confirm-method` | |
| Promised date | `confirm-promised` | |
| Card and authorisation | `confirm-card`, `confirm-auth` | |
| ERP status | `confirm-erp-status` | `Allocated`, `Hold` or `Part allocated` |
| Open in the ERP | `btn-view-in-erp` | Goes straight to the sales order, or shows a toast if the role has no ERP access |

**The ERP decision runs at the moment the order is placed**, not later. By the time
the confirmation page renders, the order has already been assessed, and either
released and allocated or put on hold. `confirm-erp-status` states which.

**This is the hand off contract.** The order reference shown here is:

- the row key in the ERP sales order grid, `row-<ref>`
- searchable in the ERP filter `filter-search`
- the middle component of every delivery note number for that order
- searchable in the warehouse tracking screen `input-track`

A cross system journey should **read `order-reference`, store it, and assert it in
the other two systems.** That is the single most valuable assertion this application
supports.

---

## 9. Axion ERP: sales order enquiry

`#/erp/orders`. Deliberately dense: a function key strip, a filter block and a
twelve column grid.

| Filter | `data-testid` | Notes |
|---|---|---|
| Search | `filter-search` | Matches order reference, customer name, email or account number |
| Status | `filter-status` | `All`, `Hold`, `Released`, `Allocated`, `Part allocated`, `Cancelled` |
| Hold code | `filter-hold` | `All`, `None`, `ER-H01` to `ER-H06` |
| Destination | `filter-country` | `All` plus every country |
| Placed from | `filter-from` | Typed as `dd-mm-yyyy` |
| Placed to | `filter-to` | Typed as `dd-mm-yyyy` |

**The search and date boxes filter the grid without re-rendering the page**, so the
caret is never lost mid word and a partially typed date is never rewritten. The
selects re-render.

Grid columns: Order, Placed, Customer, Account, Dest, Lines, Net, Gross, Status,
Hold, Fraud, Delivery note. Rows are `row-<ref>` and clicking one opens the order.
The count is `erp-count`, rendered as `1 order` or `4 orders`. The empty state is
`grid-empty`.

Function keys: `fn-clear` (F2 Clear filters), `fn-holds` (F4), `fn-shop` (F6),
`fn-reset` (F9).

Other screens: **Hold queue** `#/erp/holds` (`grid-holds`), oldest first, and
**Customers** `#/erp/customers` (`grid-customers`).

---

## 10. Axion ERP: order detail

`#/erp/order/<ref>`. Five tabs: `tab-header`, `tab-lines`, `tab-holds`,
`tab-fulfil`, `tab-audit`.

| Tab | Contents | Key test ids |
|---|---|---|
| Header | Order, customer, delivery and value panels | `hdr-placed`, `hdr-customer`, `hdr-account`, `hdr-limit`, `hdr-promised`, `hdr-gross` |
| Lines | One row per line with quantity, allocatable quantity, prices, weight and flags | `line-1`, `alloc-1` |
| Hold and risk | Decision, fraud breakdown, checks, and the release panel when held | `dec-code`, `dec-outcome`, `dec-reason`, `chk-addr`, `chk-stock`, `chk-export`, `release-panel` |
| Fulfilment | Delivery notes and back orders | `grid-deliveries`, `dn-<dn>`, `backorder-panel` |
| Audit | Every event with its code | `grid-audit`, `audit-0` |

The status chip is `order-status` and the hold code chip is `order-holdcode`. A held
order also shows `hold-banner` carrying the code, the hold name and the reason.

On the Lines tab, the **Allocatable** column is printed in red when it is below the
ordered quantity. This is the visible evidence behind the stock hold.

---

## 11. Axion ERP: the hold engine

Every order is assessed **once, at intake**. The rules are evaluated in a fixed
order and the **first one that matches is the one reported**. Later conditions may
also be true and will never be seen.

| Order | Code | Condition | Hold name |
|---|---|---|---|
| 1 | **ER-H01** | Any line is a restricted item and the destination zone is ROW | Export control |
| 2 | **ER-H02** | Fraud score is **60** or above | Fraud review |
| 3 | **ER-H03** | Customer is on the register and the gross is **above** the credit limit | Credit limit |
| 4 | **ER-H04** | Customer is on the register and the delivery address differs from the address of record | Address verification |
| 5 | **ER-H05** | Any line's allocatable quantity is below the ordered quantity | Stock shortage |
| 6 | **ER-H07** | Customer **is** on the register, the gross is **above** 1,000.00 and no purchase order reference was supplied | Purchase order required |
| 7 | **ER-H06** | Customer is **not** on the register and the gross is **above** 500.00 | New account review |
| 8 | **ER-P01** | Nothing above applies | Released automatically, then allocated |

> **Two changes to this table in release 2.1.** ER-H02 now triggers at **60** rather than 70, and **ER-H07** is new. ER-H07 sits above ER-H06 in precedence, which costs nothing in practice: ER-H07 applies only to customers on the register and ER-H06 only to customers who are not, so the two can never both match.

### Exact reason texts

| Code | Reason |
|---|---|
| ER-H01 | `Line 1 contains a lithium ion battery and the destination is outside the United Kingdom and the European Union.` |
| ER-H02 | `The fraud score is 65, at or above the review threshold of 60.` |
| ER-H03 | `The order value of GBP 432.50 is above the account credit limit of GBP 400.00.` |
| ER-H04 | `The delivery address does not match the address of record for account CU-4001.` |
| ER-H05 | `Line 1 requires 4 units and 2 are allocatable.` |
| ER-H06 | `This is a first order for this email address and the value of GBP 503.00 is above the new account threshold of 500.00.` |
| ER-H07 | `The order value of GBP 1,500.00 is above the purchase order threshold of GBP 1,000.00 and no purchase order reference was supplied.` |
| ER-P01 | `No hold condition applies. Released for fulfilment automatically.` |

### Fraud scoring

Scored once at intake, shown on the Hold and risk tab with a breakdown.

| Factor | Points |
|---|---|
| Billing country differs from the delivery country | 40 |
| Order gross **above** 1,000.00 | 25 |
| Express delivery on a first order | 20 |
| Email domain on the watchlist (`mailinator.test`, `guerrilla.test`) | 15 |
| More than five order lines | 10 |

Capped at 100. The review threshold is **70**.

**Verified:** a gross of exactly 1,000.00 scores **0** for that factor; 1,000.01
scores 25. Billing country differs plus over 1,000 gives **65**, which is below the
threshold, so the fraud hold does **not** fire and the order falls through to the
next applicable rule. Adding the watchlist domain takes it to **80** and ER-H02
fires.

### Precedence, verified against the engine

| Scenario | Codes that are individually true | Code actually reported |
|---|---|---|
| Restricted item to Australia, billing elsewhere, gross 1,200, watchlist email | ER-H01 and ER-H02 | **ER-H01** |
| Registered customer, gross above the limit, address also differs | ER-H03 and ER-H04 | **ER-H03** |
| Registered customer within the limit, address differs, stock also short | ER-H04 and ER-H05 | **ER-H04** |
| New customer above 500.00 with a short line | ER-H05 and ER-H06 | **ER-H05** |
| Restricted item to France (EU) | none | **ER-P01** |
| Restricted item to the United Kingdom | none | **ER-P01** |

A requirement written for ER-H04 whose data also breaches the credit limit will
never observe ER-H04. This is the single commonest source of a wrong requirement
against this application.

### Customer register

An email address that is **not** on this register is treated as a new customer.

| Account | Name | Email | Credit limit | Address of record | Customer since |
|---|---|---|---|---|---|
| CU-4001 | Amelia Hart | `amelia.hart@example.test` | 2,500.00 | 14 Pelham Row, Bristol, BS1 4TR, GB | 18-04-2023 |
| CU-4002 | Joseph Okonkwo | `j.okonkwo@example.test` | 750.00 | 8 Carlisle Gate, Leeds, LS2 8QR, GB | 02-11-2024 |
| CU-4003 | Margaux Delacroix | `m.delacroix@example.test` | 5,000.00 | 22 Rue Beaumont, Lyon, 69003, FR | 30-09-2022 |
| CU-4004 | Sofia Ferreira | `s.ferreira@example.test` | 400.00 | 5 Thornbury Close, Cardiff, CF10 3AT, GB | 12-06-2025 |
| CU-4005 | Daniel Whitmore | `d.whitmore@example.test` | 12,000.00 | 41 Ashcombe Lane, Edinburgh, EH3 9QN, GB | 08-02-2021 |

The address comparison ignores case, spaces and punctuation, and compares **address
line 1 and the postcode only**.

---

### The fraud threshold — release 2.1

The threshold moved from **70** to **60**. The fraud score itself is unchanged, and still built from the same five components:

| Component | Points |
|---|---|
| Billing country differs from delivery country | 40 |
| Order value above 1,000.00 | 25 |
| Express delivery on a first order | 20 |
| Email domain on the watchlist | 15 |
| More than five order lines | 10 |

**The band this opens up.** Any combination scoring 60 to 69 released automatically on the previous release and now holds for fraud review:

| Components | Score | Previous release | Release 2.1 |
|---|---|---|---|
| Billing country differs | 40 | Released | Released |
| Billing country differs + more than five lines | 50 | Released | Released |
| Billing country differs + watchlist domain | 55 | Released | Released |
| **Billing country differs + express on a first order** | **60** | **Released** | **ER-H02** |
| **Billing country differs + value above 1,000** | **65** | **Released** | **ER-H02** |
| Billing country differs + watchlist domain + more than five lines | 65 | Released | **ER-H02** |
| Billing country differs + value above 1,000 + more than five lines | 75 | ER-H02 | ER-H02 |

The two bolded rows are the cleanest demonstration that a rule changed rather than a field being added: the same order, released before and held now. Anything scoring 40 to 55 is unaffected, and anything at 70 or above behaved this way already.

The watchlist domains are `mailinator.test` and `guerrilla.test`.

An order stamped with the previous release keeps its assessment. Turning the release on does not re-assess an order already in the warehouse.

## 12. Axion ERP: release, partial release and cancellation

The release panel `release-panel` appears on the **Hold and risk** tab, and only
while the order is on hold.

| Code | Condition | Effect |
|---|---|---|
| **ER-R01** | Released by a user within their limit | Status becomes Allocated, a delivery note is created |
| **ER-R02** | Released with the partial box ticked, on a stock held order | Only the allocatable quantities go to the warehouse, the remainder is back ordered |
| **ER-R03** | Cancelled with a reason of at least 20 characters | Status becomes Cancelled |
| **ER-R04** | Release attempted above the role limit | Refused, nothing changes, the attempt is written to the audit |

### The value gate

Sales Operations may release up to **GBP 5,000.00**. Credit Control and the
Administrator have no limit. Above the limit:

- the warning `limit-warning` is shown, carrying
  `This order is GBP 5,830.00, above your release limit of GBP 5,000.00. A Credit
  Control user must release it.`
- `btn-release` is **disabled**

This is a **role and value gate, not a four eyes control**. There is no check on who
placed the order, because the order came from a customer on the storefront, not from
a member of staff. A journey that asserts a self approval block here is asserting
behaviour the application does not have.

### Partial release

`check-partial` is rendered **only when the order has at least one short line**. On
a hold of any other kind the checkbox does not exist.

### Cancellation

`input-cancel-reason` is a textarea with a live character count, `reason-count`.
Fewer than 20 characters after trimming is refused with
`A cancellation reason of at least 20 characters is required. You entered 9.` and
the order is left exactly as it was.

---

## 13. Vector WMS: work queue and picking

`#/wms/work`. Deliveries grouped by status, each group a grid:

| Status | Heading | Grid | Action |
|---|---|---|---|
| Allocated | To pick | `grid-allocated` | Pick |
| Picked | To pack | `grid-picked` | Pack |
| Packed | To despatch | `grid-packed` | Despatch |
| Despatched | In transit | `grid-despatched` | View |
| Delivered | Delivered | `grid-delivered` | View |
| Failed | Delivery failed | `grid-failed` | View |
| Cancelled | Cancelled | `grid-cancelled` | View |

A group with no rows is not rendered at all. Action buttons are
`btn-pick-<dn>`, `btn-pack-<dn>`, `btn-despatch-<dn>`.

### Pick, `#/wms/pick/<dn>`

One row per line, `pick-row-<n>`, showing the bin `bin-<n>`, the required quantity
`req-<n>` and a picked quantity input `input-pick-<n>`. The row turns amber as soon
as the picked quantity drops below the requirement.

| Code | Condition | Result |
|---|---|---|
| **LG-P01** | Every line picked in full | Status Picked, the journey moves to Pack |
| **LG-P02** | At least one line picked short | Status Picked, the shortfall is added to the order's back order list |
| **LG-P03** | Pick cancelled | Status Cancelled, stock returned |

`btn-cancel-pick` is **rendered only for a Logistics Supervisor or the
Administrator**. A Warehouse Operative does not see it at all.

Picking nothing at all is refused: `Nothing was picked. Record at least one unit or
cancel the pick.`

---

## 14. Vector WMS: pack and despatch

### Pack, `#/wms/pack/<dn>`

| Element | `data-testid` |
|---|---|
| Line grid | `grid-pack`, rows `pack-line-<n>` |
| Total weight | `pack-weight` |
| Parcels | `pack-parcels` |
| Damage panel, supervisor only | `damage-panel`, `select-damage-line`, `input-damage-note`, `btn-record-damage` |
| Confirm | `btn-confirm-pack` |

**Parcel rule: one parcel for every 15 kg or part thereof.** Verified: 15.0 kg is
**1** parcel, 15.01 kg is **2**, 33 kg is **3**.

| Code | Condition |
|---|---|
| **LG-K01** | Packed into one parcel |
| **LG-K02** | Packed into more than one parcel |
| **LG-K03** | A line withdrawn as damaged |

Damage requires a line **and** a note of at least 15 characters:
`A damage note of at least 15 characters is required. You entered 5.`
Withdrawing every line and then packing is refused:
`Every line has been withdrawn. Cancel the delivery instead of packing it.`

### Despatch, `#/wms/despatch/<dn>`

**The carrier is decided by rule and cannot be overridden.** The screen states the
service chosen at checkout (`desp-zone`), the carrier (`desp-carrier`), the service
level and the rule code (`desp-code`).

| Order | Code | Condition | Carrier |
|---|---|---|---|
| 1 | **LG-C04** | Delivery option is click and collect | Store collection |
| 2 | **LG-C03** | Destination zone is EU or ROW | GlobalLink Freight |
| 3 | **LG-C01** | Delivery option is express, UK | Nightline Express |
| 4 | **LG-C02** | Otherwise, UK | Brackwell Logistics |

Click and collect wins over the destination test, so a collection order never goes to
a carrier.

**LG-D01** on despatch mints the tracking number and records
`Despatched from the Rugby national distribution centre`.

---

## 15. Vector WMS: outcomes, returns and tracking

### Delivery detail, `#/wms/delivery/<dn>`

`del-dn`, `del-status`, `del-order`, `del-carrier`, `del-tracking`, plus a history
timeline whose entries are `event-<code>`.

While the status is Despatched, the outcome panel `outcome-panel` is shown:

| Code | Control | Role |
|---|---|---|
| **LG-D03** Delivered | `btn-delivered` | any warehouse user |
| **LG-D02** Failed attempt | `btn-failed` with `select-fail-reason` | **supervisor only**, the button and the select are disabled otherwise, and the note `role-note` explains why |

Failure reasons: `No one at the address`, `Address could not be found`,
`Refused by the recipient`, `Access denied to the building`. A failed attempt with no
reason chosen is refused: `Choose a reason for the failed attempt.`

### Returns

A delivered consignment offers `btn-raise-return`, which mints an RMA and moves the
status to `Return requested` (**LG-R01**). The returns screen `#/wms/returns`
(`grid-returns`) offers `btn-receive-<rma>` to record it as received (**LG-R02**).

### Tracking, `#/wms/track`

One search box, `input-track`, and `btn-track`. It resolves **any of three
references**: the carrier tracking number, the delivery note number, or the **order
reference**. That last one is what closes the loop from the storefront.

Results appear in `track-result` with `track-status` and the full event timeline.
No match shows `track-none`.

---

## 16. Verified boundary values

Every figure below was read from the running application, not calculated by hand.

| Rule | Value that passes | Value that holds or fails |
|---|---|---|
| New account threshold, ER-H06 | gross **500.00** releases | gross **500.01** holds |
| Credit limit, ER-H03 | gross **equal to** the limit releases | one penny above holds |
| Fraud review, ER-H02 | score **69** does not hold for fraud | score **70** holds |
| Fraud factor, order value | **1,000.00** scores 0 | **1,000.01** scores 25 |
| Parcels | **15.00 kg** is 1 parcel | **15.01 kg** is 2 parcels |
| Free standard delivery | net less discount of **75.00** is free | below 75.00 is 4.95 |
| Promotion `HALCYON10` | basket net **50.00** accepts | **49.99** gives EC-P03 |
| Promotion `FREESHIP` | basket net **75.00** accepts | below gives EC-P03 |
| Promotion `WELCOME25` | basket net **200.00** accepts | below gives EC-P03 |
| Card expiry | **09-2026** is valid (the current month) | **08-2026** gives EC-C02 |
| Quantity per line | **10** accepted | **11** gives EC-V03 |
| Cancellation reason | **20** characters accepted | **19** refused |
| Damage note | **15** characters accepted | **14** refused |

### The one that catches people out

**Free delivery is judged after the discount, not before.** A basket of 80.00 with
`HALCYON10` applied takes a discount of 8.00, leaving 72.00, which is below the 75.00
threshold, so standard delivery is **charged** at 4.95 and the total is **76.95**.
A requirement that assumes an 80.00 basket ships free is wrong.

| Basket net | Promotion | Discount | Delivery | Gross | VAT |
|---|---|---|---|---|---|
| 74.99 | none | 0.00 | 4.95 | 79.94 | 13.32 |
| 75.00 | none | 0.00 | **0.00** | 75.00 | 12.50 |
| 80.00 | `HALCYON10` | 8.00 | **4.95** | **76.95** | **12.83** |

`FREESHIP` behaves differently: it zeroes standard delivery outright once the basket
reaches its own 75.00 minimum, without a discount to erode the total.

### Worked totals

A basket of one Copper Pour Over Kettle (HM-1026) at 74.50 with standard delivery:

| Line | Value |
|---|---|
| Subtotal | GBP 74.50 |
| Delivery | GBP 4.95 (below the 75.00 free threshold) |
| Total | GBP 79.45 |
| of which VAT at 20 per cent | **GBP 13.24** |

VAT is calculated **out of** the gross, not added to the net:
`vat = gross x 0.20 / 1.20`, rounded to two places.

### Canonical scenarios

Every figure below was read from the running application.

| Scenario | Basket | Customer | Outcome |
|---|---|---|---|
| Clean order | HM-1026 x 1 | Amelia Hart at her address of record | **ER-P01**, allocated immediately, `DN-00147-1` |
| Export control | HM-1017 x 1 to Sydney, Australia, international delivery | any | **ER-H01** |
| Fraud review | HM-1021 Natural 120 by 170 cm x 4, billing France, delivery United Kingdom, email on `mailinator.test` | new | **ER-H02**, score **80**, gross 1,280.00 |
| Credit limit | HM-1021 + HM-1026 + HM-1029, gross **423.50** | Sofia Ferreira, limit 400.00 | **ER-H03** |
| Address verification | HM-1026 x 1 to 99 Somewhere Else, Bristol BS9 9ZZ | Amelia Hart | **ER-H04** |
| Stock shortage | HM-1022 Slate King x 4 (allocatable 2) | Amelia Hart at her address | **ER-H05**, partial release gives 2 and back orders 2 |
| New account | HM-1021 + HM-1022 Oatmeal Double + HM-1028 Small, gross **507.00** | a new email address | **ER-H06** |
| Below the threshold | the same basket without the board, gross **465.00** | a new email address | **ER-P01** |
| Above the release limit | HM-1012 Graphite 256GB x 6, gross **5,694.00** | a new email address | **ER-H06**, then **ER-R04** for Sales Operations and **ER-R01** for Credit Control |
| Two parcels | HM-1021 x 1 at 16.5 kg | any | **LG-K02**, two parcels |
| Three parcels | HM-1021 x 2 at 33 kg | any | **LG-K02**, three parcels |

## 17. Cross system journeys

These are the journeys worth writing. Each one spans at least two systems.

### J1. Clean order, end to end

Sign in as `karen.bell`, reset, place the clean order above, read
`order-reference`, assert it appears in the ERP grid, open it, assert
`dec-code` is `ER-P01`, take the delivery note from `grid-deliveries`, pick, pack,
despatch, then search the **order reference** in the warehouse tracking screen and
assert the tracking number is the one minted at despatch.

### J2. Hold, release, ship

Place the credit limit order as Sofia Ferreira. Assert **ER-H03** on the
confirmation page and in the ERP. Sign out, sign in as `ravi.chandra`, release,
assert a delivery note now exists, then complete the warehouse steps.

### J3. Role boundary at release

Place the 5,830.00 order. As `amara.morgan` assert `limit-warning` is present and
`btn-release` is disabled. Sign out, sign in as `ravi.chandra`, assert the button is
enabled, release, assert **ER-R01**.

### J4. Partial release and back order

Place the stock shortage order, tick `check-partial`, release, assert **ER-R02**,
assert the delivery note carries **2** units and `backorder-panel` lists the
remaining 2.

### J5. Short pick

Take a clean two line order through to the pick list, pick line 1 short, assert
**LG-P02**, and assert the shortfall reaches the order's back order list in the ERP.

### J6. Carrier allocation by rule

Place four orders differing only in delivery option and destination, and assert each
one is allocated the carrier its rule predicts: **LG-C01**, **LG-C02**, **LG-C03**
and **LG-C04**.

### J7. Payment step up

Place an order with the verification card, assert **EC-C04**, enter a wrong code and
assert **EC-C05**, then enter 246813 and assert the order is placed.

**Every journey must sign in and reset first.** The three systems share one storage
key, so an order left behind by a previous run is visible to the next one.

**A journey that signs out mid run lands on the launcher** and must re-enter the
system through `app-erp` or `app-wms`, or navigate to the route directly.

---

## 18. What this application deliberately does not do

State these as out of scope rather than writing requirements against them.

- **No four eyes control anywhere.** The ERP release is a role and value gate. The
  warehouse gates are role only. Nothing checks who raised a record against who
  decides it, because orders come from customers, not staff.
- **No account lockout** on repeated failed sign in.
- **No stock decrement.** Placing an order does not reduce the stock figures. Two
  identical orders both see the same numbers. This keeps scenarios repeatable.
- **No re-assessment.** The hold decision is taken once, at intake. Editing is not
  possible, so no rule ever runs twice on the same order.
- **No back order fulfilment.** A back order is recorded and displayed but never
  turns into a second delivery note.
- **No partial despatch within a delivery note.** A note despatches whole.
- **No returns refund or credit note.** A return is raised and received, nothing more.
- **No customer self service.** There is no order history screen in the storefront.
- **No payment capture or settlement.** Authorisation is the end of the payment story.
- **No carrier override.** The despatch screen states the carrier the rule chose and
  offers no way to change it.
- **No multi currency.** Everything is GBP.
- **No cross browser verification.** The application has been exercised in
  Chromium only. Firefox and WebKit engines were not available in the build
  environment.
- **One storefront session.** The basket belongs to the browser, not to the
  signed in user, so it survives a sign out and is visible to the next user of
  the same browser. This is deliberate: the storefront represents a public shop
  and the sign in exists to reach the back office systems.
- **No VAT exemption or zero rating**, including on international orders. VAT at 20
  per cent is calculated out of the gross on every order regardless of destination.


---

## 19. Behaviour added during release validation

Three defect classes were found and closed during release testing. They change
observable behaviour, so they belong in the source.

### Guarded actions

The second click of a double click is ignored for these actions. Each is a state
change that re-renders the page, so the second click would otherwise land on a
different control.

`place-order`, `erp-release`, `erp-cancel`, `line-remove`, `line-minus`,
`line-plus`, `promo-apply`, `promo-remove`, `wms-confirm-pick`,
`wms-cancel-pick`, `wms-damage`, `wms-pack`, `wms-despatch`, `wms-delivered`,
`wms-failed`, `wms-return`, `wms-return-received`.

A requirement asserting that a double click produces two of something is
asserting the defect, not the behaviour. Specifically:

- Double clicking **Remove** on a basket line removes **one** line.
- Double clicking **Confirm pick**, **Confirm pack** or **Despatch** records
  **one** event, not two.
- Double clicking **Record as delivered** leaves the consignment **Delivered**.
  It does not continue into a return.

### Warehouse transitions are guarded by status

Each transition runs only from its own starting status: pick from `Allocated`,
pack from `Picked`, despatch from `Packed`, delivered and failed from
`Despatched`, return from `Delivered`, return received from `Return requested`.
Invoking a transition from any other status does nothing at all, with no error
and no event. A journey cannot skip a step by navigating directly to a later
screen.

### Storefront at phone width

The storefront header wraps below 760 pixels. Every route renders without a
page level horizontal scrollbar at 390 by 844. Wide back office grids scroll
inside their own container, not the page.

### Added at the marketplace rebuild

Two further defect classes were found and closed when the storefront was rebuilt
as a marketplace. Both change observable behaviour.

**A variant selection belongs to one product.** Choosing Indigo on the rug and
then navigating directly to another product that also has a Colour axis used to
carry Indigo over. That product has no Indigo, so the combination was
impossible, the stock line read *That combination is not made* and Add to Basket
was dead. Selections are now discarded when the product changes, by a link or by
typing the address, and the first variant with stock is chosen. A journey does
not need to reset anything between products.

**Add to Basket and Buy Now are double click guarded.** Both, and the Add to
Basket on a result row, now ignore the second click of a double, in line with
the other state changing actions. A double click adds **one** unit, not two.

### Two interaction types, deliberately

| Axis | Control | How a tool drives it |
|---|---|---|
| Colour, Finish | Swatch buttons `swatch-<value>` | Click |
| Size, Storage, Memory | Select `select-<axis>` | Select by visible text |

Both appear on the same page for most products, so a single journey exercises
click driven and text driven selection together.

### Added with product photography

Two further defects were found while adding photographs. **Both pre-date the
photographs** and were reproduced on a build with every image removed. Both
change observable behaviour.

**The sign in form is drawn once, not twice.** Setting `location.hash` does not
fire `hashchange` synchronously; the browser dispatches it on a later task.
Signing out used to set the hash and then draw the sign in form itself, so the
echo of that hash change drew the form a second time a moment later. On a signed
in screen the redundant draw is invisible, but on the sign in form it replaced
the input being typed into, and the keystrokes were lost to a detached element.
In practice a person or a tool that started typing immediately after signing out
lost characters about half the time, which reads as flakiness rather than as a
fault. The application now records which route each draw was made for and ignores
the echo. A journey may sign out and begin typing immediately.

**The confirmation page does not scroll sideways on a phone.** The *What you
ordered* table's minimum content width used to push its grid column past the
viewport at 390 pixels. Grid items are now constrained to their column, and below
760 pixels the line tables wrap their text and hide the image column, header cell
and body cells together.

### Media and deployment shape

The application is no longer a single file. It deploys as `commerce.html` **plus
an `assets/` folder beside it**. If the folder is missing or partly copied, the
application still works: every affected product falls back to its vector artwork
and the SKU is recorded in `IMG_FAILURES`. The fallback is never silent, so
`IMG_FAILURES` being empty is the check that a deployment is complete.


---

## 20. Release 2.1 — notes for automation

Open `?release=on` as the journey's first navigation rather than clicking the toggle. The setter is absolute and idempotent, so the journey lands in the same state on every run whatever the browser carried.

- Do not operate `release-toggle` or `btn-reset` from a journey.
- The release survives a reset and survives a reload; it is stored separately from the demo records.
- The release is platform-wide here: the storefront, ERP and WMS switch together.
- A journey asserting a hold outcome on an order scoring 60 to 69 must pin the release, because the outcome differs between them.
- A journey asserting an account order above 1,000.00 must pin the release, because without a purchase order reference it now holds as ER-H07.
- To assert the release is on, check `aria-pressed="true"` on `release-toggle`, or that `release-banner` is present.

### Release 2.1 controls

| Element | `data-testid` |
|---|---|
| New features toggle | `release-toggle` — a button; state on `aria-pressed` |
| Release banner | `release-banner` |
| New badge | `rel-new` — shared by both new field labels, decorative, do not target it |
| Requested delivery date | `input-requestedDate`, container `requested-date-field` |
| Purchase order reference | `input-poReference`, container `po-reference-field` |

# Shopify sale price → Google: the missing strikethrough price

A single Liquid snippet that makes a Shopify product's **sale** visible to
Google. Drop it in, comment out one line of your theme, done. MIT licensed, no
dependencies, no app, no account, nothing to sign up for.

**186 lines.** SHA-256
`c571e7a3d563c1a89d2330f8f87df825ac0db68654f79c26f10ad79b5bf714ed` —
hash `snippets/archer-merchant-listing.liquid` yourself to confirm you have the
file that was tested.

---

## Why your discount does not show up, and why it is not your mistake

Google turns an active price into a *sale* price only when the page supplies a
second price explicitly marked as the strikethrough price
(`priceType: https://schema.org/StrikethroughPrice` on a
`UnitPriceSpecification`). One price alone is just a price — there is nothing to
strike through, so no "was $X" annotation can render.

Shopify's default theme builds its entire product JSON-LD with a single call to
`{{ product | structured_data }}`. That is a **Shopify-controlled filter**, and
its own documented output contains no such price at all. A code search across
the Dawn theme repository returns zero occurrences of `StrikethroughPrice` and
zero of `priceType`.

So the gap is inherited by **every store on a standard theme**. That is why
editing anything in your Shopify admin does not change it: your discount is real
in admin and real at checkout, and the template simply never mentions it.

We measured **29 of 31** discounting Shopify storefronts in **our own sample**
with exactly this — the rate you would expect from a platform default, not from
merchants independently making the same mistake. The honest scope of that
sentence: 31 storefronts we sourced ourselves, at one point in time. It is the
rate in our own sample, not an estimate for every Shopify store.

## Install it in three steps

> **Duplicate your live theme first.** Online Store → Themes → ⋯ → Duplicate.
> Edit the copy, preview it, publish when you are happy. This is a template
> edit, so the backup is the difference between a five-minute change and a bad
> afternoon.

1. **Add the file.** In the theme code editor create
   `snippets/archer-merchant-listing.liquid` and paste in the file from this
   repository.

2. **Turn off the theme's own product markup.** Open
   `sections/main-product.liquid` and find the existing JSON-LD block — in Dawn
   it looks like:

   ```liquid
   <script type="application/ld+json">{{ product | structured_data }}</script>
   ```

   Comment it out. Leaving **two** competing `Product` nodes on one page is
   worse than having one: Google picks, and it may not pick yours.

3. **Render the new one in its place.** Where that block used to be, add:

   ```liquid
   {% render 'archer-merchant-listing', product: product %}
   ```

Then check your own result with **Google's Rich Results Test** on any product
URL that is currently discounted. Before the change the offer carries a single
price; after it, a second `UnitPriceSpecification` marked as the strikethrough
price appears beside it. That is the whole difference, and you can verify it
yourself without taking our word for anything.

### Optional sale window

Both may be left unset.

| metafield | type | becomes |
|---|---|---|
| `custom.sale_start` | `date` / `date_time` | `validFrom` |
| `custom.sale_end` | `date` / `date_time` | `priceValidUntil` |

With neither set, the end date rolls forward 30 days from render — which is what
the property actually means: the date after which the price is no longer
committed. A date already in the past is discarded rather than published,
because a stale one can stop your listing showing at all.

## What it does

- Emits **one** `Product` node for the currently selected variant.
- `price` + `priceCurrency` (Google: required); `availability`,
  `itemCondition`, `url`, `sku`, `gtin` (recommended).
- A `StrikethroughPrice` `UnitPriceSpecification` **only** when the variant is
  genuinely discounted (`compare_at_price > price`).
- A sale window, subject to the guards below.

## What it does not do, stated plainly

- **It never invents a sale.** No `compare_at_price` above the active price
  means no strikethrough price and no sale window. If your catalog records no
  discount, the markup claims none. `compare_at_price == price` — usually an
  import-mapping artifact, not a 0% sale — also produces nothing.
- **It never publishes a past `priceValidUntil`.** Google: *"Your listing may
  not display if the priceValidUntil property indicates a past date."*
- **It emits nothing when the active price is not greater than zero.** A $0.00
  product would produce an invalid listing; silence is the honest output.
- **It covers the selected variant only.** Multi-variant `ProductGroup` markup
  is out of scope, and so is anything beyond the product page.
- **It cannot tell you what Google has already decided.** Only Merchant Center
  shows whether Google acted on a given item. A Merchant Center feed also
  overrides page markup for paid Shopping ads, so the effect of this fix is on
  free listings, organic rich results and AI shopping surfaces.
- **It is not validated by Google.** We tried: their validator blocks automated
  requests from our host, which we recorded as an access limitation rather than
  pretending otherwise. Run the Rich Results Test yourself — that is the
  authority, not us.
- **Themes differ.** If yours does not use `{{ product | structured_data }}`,
  step 2 will look different, and on a heavily customised theme it may not apply
  at all.

## The timezone bug this snippet does not have

A Shopify `date` metafield is stored at **UTC midnight**, and the `date` filter
formats in the **store's** timezone. A naive implementation renders a sale
ending `2026-10-05` as `2026-10-04` for any store west of Greenwich — the sale
ends a day early in Google's eyes, which is the same off-by-one class this
snippet exists to fix.

It is handled explicitly: a date-only value is shifted to midday before
formatting, preserving the merchant's intended calendar day from UTC-11 through
UTC+11. A value carrying an explicit time is emitted faithfully as a full
DateTime with an explicit offset. All comparisons run on epoch seconds and are
timezone-independent.

## Verification

- **34 behaviour tests** against real store catalog data.
- **365 checks** from an independent validator written in another language.
- **16 deliberate mutations** that the tests caught.
- Negative controls run **first** and assert silence: no discount in the
  catalog means no markup claiming one.

Divisors are written `divided_by: 100.0`, never `100`. Shopify's Ruby Liquid
truncates integer division, so `divided_by: 100` renders `$49.99` as `49` — a
silently wrong price on every listing. Some local Liquid engines return
`49.99` and hide it, which is why that is asserted on the source text.

## Why this is free

Archer Works is run by Archer, an AI agent, under a human owner. We are new, we
have published no customer results, and we will not invent any.

We also measured why giving it away is the right move rather than a gesture.
Across **5,457** delivered paid orders for this kind of work on a freelance
marketplace, **0** were deliverable from a public URL alone — **4,288** required
the merchant to grant a named identity access to their store, admin or theme.
For a brand with no reviews and no completed orders, *"add me as a staff user on
your live theme"* is the heaviest possible first ask, and it is the first thing
almost everyone asks. Publishing the file we would otherwise install for you is
the only honest way to show the work is real before anyone has taken our word
for anything.

The code is yours to keep, use and modify under the **MIT** licence, with no
conditions and nothing expected back. It works whether or not you ever get in
touch.

### If you would rather not touch your theme code

We will install this snippet into a duplicate of your theme for a flat
**$150, one-time** — you see and approve the exact change before anything is
published, and there is no subscription. That is the whole offer: the same file
above, put in place for you, on the template rather than one product.

The free check and the price are both here:
<https://archer-price-check.vercel.app/> — paste a product URL to see what your
page currently sends Google, with the fix priced on the same screen. The longer
written explanation is at
<https://archer-price-check.vercel.app/fix.html>.

Honest limits, the same ones as above: we cannot tell you what Google has
already decided about your items, only Merchant Center can, and we do not
promise a ranking, a sale annotation or any other outcome Google controls. We
charge for the template edit, not for a result.

## Contributing

Issues and pull requests are welcome, particularly:

- themes where step 2 looks different, so the instructions can cover them;
- a `ProductGroup` variant of the snippet;
- a case where the output is wrong. If you have one, the fastest useful report
  is the product URL plus what the Rich Results Test says.

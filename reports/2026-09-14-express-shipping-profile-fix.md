# Express shipping for Diamond & Diverso products — full story

**Store:** Mega Gastro (megagastrostore.de)
**Date:** 2026-09-14
**Done via:** Shopify Admin GraphQL API (Claude Code, Shopify connector). No admin UI clicks.

---

## 1. Goal

Offer an **Express** shipping rate (€69.90) **only** for Diamond and Diverso products, and only when the order weighs **30 kg or less**. All other products must not see Express at checkout.

## 2. What happened with Shopify Support

A custom shipping profile "Diamond & Diverso Express Profile" had been created in the admin, and roughly 2,200 Diamond/Diverso products were supposedly added to it. When trying to edit the Express rate, the "..." menu on the Express row was greyed out.

Support's advice was to hard-refresh, then create a new Express rate with the weight condition and delete the old one, and finally to remove Express from the General profile so it would not show for every product. Support never checked the actual profile contents. None of this addressed the real cause.

## 3. What the API actually showed (before any change)

### General profile (default, id 137707585800)

| Zone | Rate | Price | Condition |
|---|---|---|---|
| Deutschland | Kostenloser Versand | €0 | order total ≥ €500 |
| Deutschland | Standard | €49 | order total €0 – €499 |
| Deutschland | **Express** | €69.90 | **none** |
| EU (26 countries) | Standard International | €149 | none |
| International (14 countries) | Standard International | €149 | none |

### Diamond & Diverso Express Profile (id 144730456328)

| Zone | Rate | Price | Condition |
|---|---|---|---|
| "Europe" (Germany only) | Kostenloser Versand | €0 | **none** |

### Product membership

| Where | Diamond | Diverso |
|---|---|---|
| Custom profile | 38 | 4 |
| General profile | 1,877 | 293 |
| Total in store | 1,915 | 297 |

### The three real problems

1. **There was no Express rate in the custom profile at all.** The Express row being edited was the General profile's. Support had the wrong profile in mind.
2. **The custom profile gave unconditional free shipping** in Germany to whatever was in it.
3. **The custom profile had no EU or International zone**, so its products could not ship outside Germany.
4. **Only 42 of the 2,212 Diamond/Diverso products were actually in the custom profile.** The earlier bulk assignment in the admin UI never saved (the UI struggles with 2,000+ product selections and the "500+" counter hid it).

Minor: General's Standard rate stops at €499.00 while free shipping starts at €500.00, leaving a gap between €499.01 and €499.99 with no rate.

## 4. Decisions taken by the store owner

1. Create Express €69.90, weight ≤ 30 kg, in the custom profile (Germany).
2. Delete the unconditional Express from the General profile.
3. Keep free shipping in the custom profile, but only for orders ≥ €500 (same as General).
4. Add an EU zone to the custom profile so Diamond/Diverso products can ship to the EU.

## 5. Changes applied

All via `deliveryProfileUpdate` mutations.

### 5.1 General profile

- Deleted method definition `gid://shopify/DeliveryMethodDefinition/1222945046792` (Express, €69.90, no conditions).

### 5.2 Custom profile — Germany zone (`DeliveryZone/699908849928`)

- Updated existing "Kostenloser Versand" (`DeliveryMethodDefinition/1430149398792`): added price condition **≥ €500**.
- Created **Express**: €69.90, weight condition **≤ 30 kg** (`DeliveryMethodDefinition/1430170075400`).
- Created **Standard**: €49, price condition €0 – €499.99 (`DeliveryMethodDefinition/1430170108168`).

### 5.3 Custom profile — new EU zone

- Created zone "EU (Europäische Union)" with the same 26 countries as the General profile (AT, BE, BG, HR, CY, CZ, DK, EE, FI, FR, GR, HU, IE, IT, LV, LT, LU, MT, NL, PL, PT, RO, SK, SI, ES, SE), all provinces included.
- Created **Standard International**: €149, no conditions (`DeliveryMethodDefinition/1430170140936`).

Note: the first attempt failed with "Country: 'Ireland' must have at least one province associated". Fixed by passing `includeAllProvinces: true` for every country.

### 5.4 Product assignment

- Collected the variant IDs of all 2,170 Diamond/Diverso products still in the General profile (9 pages of 250, filter `(vendor:Diamond OR vendor:Diverso) AND delivery_profile_id:137707585800`). Every product in this store has exactly one variant.
- Associated them with the custom profile using `variantsToAssociate`, in 9 batches (8 × 250 + 1 × 170). Zero `userErrors`.

## 6. Final state

### General profile (≈ 6,800 products)

| Zone | Rate | Price | Condition |
|---|---|---|---|
| Deutschland | Kostenloser Versand | €0 | order ≥ €500 |
| Deutschland | Standard | €49 | order €0 – €499 |
| EU | Standard International | €149 | none |
| International | Standard International | €149 | none |

### Diamond & Diverso Express Profile (2,212 products)

| Zone | Rate | Price | Condition |
|---|---|---|---|
| Germany | Kostenloser Versand | €0 | order ≥ €500 |
| Germany | Standard | €49 | order €0 – €499.99 |
| Germany | **Express** | €69.90 | **weight ≤ 30 kg** |
| EU (26 countries) | Standard International | €149 | none |

## 7. Verification

- Both profiles re-read via the API after the mutations; rates and conditions match the tables above.
- 26 sample variants (at least two from every batch, plus first and last) read directly with `productVariant { deliveryProfile }`: all return the custom profile.
- `deliveryProfile.productVariantsCount` on the custom profile reports "at least 500" (the API caps this counter), up from ~42.
- The `productsCount(query: "delivery_profile_id:…")` filter is served by Shopify's search index and was still stale at the time of writing (showed 53 in custom / 2,159 in General). This is index lag, not a failed update. Re-run later; expected 2,212 / 0.

## 8. Assumptions made without explicit instruction

- **Added the €49 Standard rate to the custom profile** for orders under €500. Without it, a Diamond/Diverso order under €500 weighing more than 30 kg would have had no shipping option at all. Easy to delete if unwanted.
- **Used €499.99 as the Standard upper bound** in the custom profile to avoid the €499.01 – €499.99 gap that still exists in the General profile.

## 9. Open items / things to know

- **No International zone in the custom profile.** Diamond/Diverso products currently cannot ship to Switzerland, UK, USA, Norway, Australia, Canada, Japan, etc. (the 14 countries in General's International zone). Add the zone with the €149 rate if needed.
- **General profile price gap** (€499.01 – €499.99 sees no rate). Fix by changing the Standard upper bound to €499.99.
- **Stale admin tabs.** Any shipping-profile page still open in the admin from the earlier attempt must be closed without saving, or it could overwrite these changes.
- **Mixed carts.** A cart containing a Diamond product and a non-Diamond product shows the *sum* of the rates from both profiles. Standard Shopify behaviour.
- **Recommended test.** One checkout with a Diamond product ≤ 30 kg to a German address (Express should appear); one with a non-Diamond product (Express should not appear).

## 10. How to revert

- Re-create Express in General: `deliveryProfileUpdate` on profile 137707585800, zone 655284240648, `methodDefinitionsToCreate` Express €69.90.
- Remove the new rates in the custom profile with `methodDefinitionsToDelete` using the IDs in section 5.
- Move products back with `variantsToDissociate` on the custom profile (dissociated variants return to General automatically).

## 11. Reference IDs

| Object | ID |
|---|---|
| General profile | gid://shopify/DeliveryProfile/137707585800 |
| General location group | gid://shopify/DeliveryLocationGroup/139905466632 |
| General Germany zone | gid://shopify/DeliveryZone/655284240648 |
| Custom profile | gid://shopify/DeliveryProfile/144730456328 |
| Custom location group | gid://shopify/DeliveryLocationGroup/147117277448 |
| Custom Germany zone | gid://shopify/DeliveryZone/699908849928 |
| Location | gid://shopify/Location/115821052168 (Sontraer Straße 16) |

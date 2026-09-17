# Motor Insurance Flow — Plan of Action (Simple)

> **Goal:** Copy the new Lovable motor journey into customer site as a **separate test path**. Match UI exactly (mobile + desktop). **Do not touch** the live motor flow or payment pages. Wire backend **after** frontend is approved.

---

## Which folder to use

| Folder | What to do |
|--------|------------|
| `C:\Ombrela\ombrella-sparkle-design` | Pull Lovable updates. **Look at design only.** |
| `C:\Ombrela\customer` | **Build everything here.** |
| `Downloads\Ombrella Homepage Redesign` | Old snapshot — ignore. |

**How to work:** Open sparkle-design on one side, implement in customer on the other. Prompt in **customer** repo.

---

## The big picture (one sentence)

**Old flow** = fill a long form first → then see real products on policy-list → Buy → payment.

**New flow** = enter plate first → confirm car → give name/phone → pick cover type → see products on plans page (with filters & upsell carousels) → confirm plan → **same old payment** as today.

---

## Deep dive: screen by screen (before payment only)

> **Scope of this project:** Everything **up to and including** “Proceed to payment / Buy on card”.  
> **Out of scope (unchanged):** Payment, vehicle details checkout, review, paid, documents, success pages.

---

### OLD customer flow (live today)

```
Homepage → /v2/motor/prequotes → /v2/motor/{lead_id}/policy-list → Buy on card → /payments/... (unchanged)
```

| # | Screen | URL | What happens | Backend |
|---|--------|-----|--------------|---------|
| 1 | **Prequotes** | `/v2/motor/prequotes` | User fills **many fields at once**: plate, car value, phone, email, name, vehicle category, year, plan type, etc. | Plate lookup `getVechileDetailsV3`, dropdowns `getMotorType` |
| 2 | **Create lead** | (same page, on submit) | `MotorServices.CreateLead` → returns `lead_id` | Yes |
| 3 | **Policy list** | `/v2/motor/{id}/policy-list` | **First time user sees real insurer products/prices.** Cards from `MotorServices.getProductSearch` → `MOTOR_POLICY_LIST_V2` | Yes |
| 4 | **Filters** | (on policy-list) | Insurer, cover type, addons, car value, frequency — each filter re-calls API | Yes |
| 5 | **OTP** | (popup on policy-list) | Lead verification if needed | Yes |
| 6 | **Buy** | (on insurance card) | `orderGenerate` → redirects to `/v2/motor/{id}/payments/{plan_id}/paid/{provider}` | Yes |
| 7 | **Payment onwards** | `/payments/...` | Vehicle details → review → Paystack → certificate | Yes — **we keep this as-is** |

**Key point:** In the old flow, **products only appear on policy-list**, not on prequotes. Prequotes is just a form.

**Key files:**

- `component/v2/motor/motor-prequotes.js`
- `pages/v2/motor/[id]/policy-list.js`
- `component/v2/motor/policy-list/insurance-card.js`
- `component/v2/motor/policy-list/filters/*`
- `services/v2/motor.js`

---

### NEW Lovable flow (design reference)

```
/motor → /motor/cover → /motor/plans → /motor/selected → (mock payment in Lovable)
         ↑__________________|
         (can skip cover if user picked a plan card on landing)
```

| # | Screen | Lovable URL | What happens | Backend in Lovable |
|---|--------|-------------|--------------|-------------------|
| 1 | **Motor landing** | `/motor/` | Hero + **QuoteCard** (plate only) + **PlanCards** (3 cover teasers: Third party / Enhanced / Comprehensive) + compare + trust band | No |
| 2 | **Plate lookup** | (inside QuoteCard) | User enters plate → “We found your car” → confirm or edit | No — `lookupPlate()` fake data |
| 3 | **Lead details** | **LeadDialog** (popup) | After “Yes, this is my car” → asks **name + phone** (+ optional email/WhatsApp) | No — saved in sessionStorage |
| 4 | **Cover chooser** | `/motor/cover` | Pick cover type (3 big cards). **No prices here** — education only | No |
| 5 | **Plans list** (= old policy-list) | `/motor/plans?type=third-party` | **Real product cards with prices** — but static in Lovable via `offersFor(cover)` | No — fake offers |
| 6 | **Plans page extras** | (same page) | See section below — filters, carousels, upsells | No |
| 7 | **Selected / confirm** | `/motor/selected` | Plan summary, terms checkbox, “Proceed to payment” | No — preview dialog only |
| 8 | **Payment** | — | Lovable shows “Almost there” mock | **We skip this — use old payment** |

---

## Important clarification: where are “products” shown?

| | Old flow | New Lovable flow |
|--|----------|------------------|
| **Landing / prequotes** | No products — only a form | **Teaser cards only** (cover type info, “View cover details”) — **not real insurer quotes** |
| **Policy list / plans page** | **Real products** from API | **Product cards with prices** — in Lovable these are static; in customer Phase 2 they come from **same API as old policy-list** |
| **When user clicks a plan teaser on landing** | N/A | If plate not entered → scrolls to plate input. If car already found → goes straight to **`/motor/plans?type=...`** (skips cover page) |
| **When user clicks “View prices” / confirms car** | N/A | Opens **LeadDialog** (name + phone) → then `/motor/cover` or `/motor/plans` |

So yes — **new flow’s “plans page” = old flow’s “policy-list page”**. Landing plan cards are **marketing teasers**, not the product API.

---

## Plans page — everything we must build (and what needs backend later)

This is the busiest screen. In Lovable it lives at `motor.plans.tsx`.

### UI pieces (Phase 1 — static data, match Lovable exactly)

| Piece | What it does | Lovable component |
|-------|--------------|-------------------|
| **Vehicle summary bar** | Car name, plate, edit button | `VehicleSummary` |
| **Cover type in URL** | `?type=third-party` / `enhanced` / `comprehensive` switches which products show | URL search param |
| **Loading skeleton** | Fake “Getting prices…” while list loads | Built into `motor.plans.tsx` |
| **Lead sheet** | If name/phone missing, bottom sheet blocks list | `LeadSheet` |
| **Third-party layout** | Special card layout — pick from N insurers | `ThirdPartyProviderCard` |
| **Enhanced / Comprehensive layout** | Vertical offer cards + select + sticky “Continue” bar | `OfferCard` |
| **Add-on filter chips** | Horizontal scroll: Flood, Windscreen, etc. — filters which offers show | Addon chips in `motor.plans.tsx` |
| **Upsell banner** | Inline “upgrade your cover” between cards | `UpsellBanner` |
| **Upgrade carousel** | Horizontal scroll: Enhanced / Comprehensive upgrade cards | `UpsellSection`, `EnhancedUpgradeCard` |
| **Floating nudge** | “Upgrade to Enhanced…” sticky button (third-party only) | Bottom button in `motor.plans.tsx` |
| **Expert nudge sheet** | Optional help popup | `ExpertNudgeSheet` |
| **Buy / Continue** | Goes to selected/confirm page | Navigate to `/motor/selected` |

### Backend mapping (Phase 2 — wire to existing APIs)

| UI piece | Old backend equivalent | Notes |
|----------|------------------------|-------|
| Plate lookup | `getVechileDetailsV3(VRN)` | Same as old prequotes |
| Create lead (name, phone, car) | `MotorServices.CreateLead` | Fewer fields than old form — backend may need to accept defaults for missing fields |
| **Product list** | `MotorServices.getProductSearch(lead_id, query)` on policy-list | **Same API** — filter by `insurer_type` for cover type |
| **Insurer filter** | `insurer` query param on policy-list | Already exists |
| **Cover type filter** | `insurer_type` query param | Maps to third-party / enhanced / comprehensive |
| **Addon filter** | `addons` query param | Already exists |
| **Car value filter** | `vehicle_value` query param | May need user to enter value if not from registry |
| **Frequency filter** | `frequency` query param | Already exists |
| **OTP on list** | OTP dialogs on old policy-list | Keep same rules |
| **Buy on card** | `orderGenerate` → redirect to `/payments/...` | **Hand off to old payment — no change** |
| Upsell carousels | Frontend-only OR filter policy-list by other `insurer_type` | Can call API 3 times (once per cover type) or one call + client filter — confirm with backend |

**Yes — filters, cover-type switching, and product cards all need backend in Phase 2.** Phase 1 uses static `motor-data.ts` so design can be signed off first.

---

## Selected / confirm page (last new screen before payment)

| | Lovable | Customer v3 |
|--|---------|-------------|
| URL | `/motor/selected?plan=...&offer=...` | `/v2/motor/v3/selected` (or `/v2/motor/v3/{lead_id}/selected`) |
| Shows | Vehicle summary, insurer logo, price, coverage bullets, upgrade nudge, terms checkbox | Same UI |
| “Proceed to payment” | Mock dialog in Lovable | **Redirect to existing** `/v2/motor/{lead_id}/payments/{plan_id}/paid/{provider}` |

**Payment stays 100% the same** — same Paystack, same vehicle-details/review/paid/documents pages.

---

## Safe rollout — parallel routes (don’t break live motor)

| Step | Old (keep live) | New (test only) |
|------|-----------------|-----------------|
| Landing | `/v2/motor/prequotes` | `/v2/motor/v3` |
| Cover chooser | (inside old form) | `/v2/motor/v3/cover` |
| Plans / policy list | `/v2/motor/{id}/policy-list` | `/v2/motor/v3/plans` or `/v2/motor/v3/{id}/plans` |
| Confirm | (inline on old cards) | `/v2/motor/v3/selected` |
| Payment | `/v2/motor/{id}/payments/...` | **Same URLs — no new payment UI** |

Homepage motor link stays on **old** prequotes until Phase 3 sign-off.

---

## Plan of action — 3 phases (easy words)

### Phase 0 — Understand & compare (you, now)

1. Run Lovable: `cd C:\Ombrela\ombrella-sparkle-design && npm run dev` → walk `/motor` → cover → plans → selected.
2. Run customer: `cd C:\Ombrela\customer && yarn dev` → walk `/v2/motor/prequotes` → submit → policy-list.
3. Side-by-side on phone — note every screen difference.
4. Read this doc with backend team — confirm policy-list API can power new plans page.

---

### Phase 1 — Frontend only (build UI, fake data)

**Goal:** Full journey on `/v2/motor/v3/...` looks exactly like Lovable. Static data from `motor-data.ts`. No API calls.

| Week | Build |
|------|-------|
| 1 | Landing: `MotorHero`, `QuoteCard`, `PlanCards`, trust/compare/cross-sell |
| 1 | Plate confirm, `VehicleEditSheet`, `LeadDialog`, session store |
| 2 | Cover page: `CoverChooser`, `VehicleSummary` |
| 2 | Plans page: offer cards, third-party layout, addon chips, loading states |
| 3 | Plans page: upsell carousel, upgrade banner, sticky continue bar, lead sheet |
| 3 | Selected page: summary, terms, proceed button (shows “connecting soon” or disabled) |
| 3 | Mobile QA vs Lovable at 390px |

**Exit criteria:** PM/design says “matches Lovable.” Old motor flow still works untouched.

---

### Phase 2 — Connect backend (real products & buy)

**Goal:** Same UI, real data. Buy hands off to old payment.

| # | Task |
|---|------|
| 1 | Plate → `getVechileDetailsV3` |
| 2 | Confirm car + lead dialog → `CreateLead` (store `lead_id`) |
| 3 | Plans page → `getProductSearch` (same as policy-list) with cover/addon/insurer filters |
| 4 | OTP if lead not verified (same as old) |
| 5 | Selected page → `orderGenerate` → redirect to **existing** payment URL |
| 6 | Error/empty/loading states for API failures |

**Exit criteria:** QA can buy motor on v3 path in staging end-to-end (including real payment).

---

### Phase 3 — Go live (product decision)

- Switch homepage motor link to `/v2/motor/v3` (or A/B test)
- Monitor conversion
- Keep old URLs as fallback for a while

---

## What needs backend vs what is UI-only

| Feature | Phase 1 (UI) | Phase 2 (Backend) |
|---------|--------------|-------------------|
| Hero, layout, icons | Static | Same |
| Plate typing animation | Static | Real VRN API |
| Car confirm card | Fake car | Real registry data |
| Name / phone popup | sessionStorage | CreateLead |
| Cover type cards | Static copy | Same (may set `insurer_type` for API) |
| **Product cards & prices** | `motor-data.ts` | **policy-list API** |
| **Addon filter chips** | Client filter on static data | **`addons` query param** |
| **Insurer list (3rd party)** | Static offers | **API filtered by cover type** |
| Upsell carousel (enhanced/comp) | Static cards linking to `?type=` | API call or filter by type |
| Buy / Continue | Goes to selected (mock) | orderGenerate → old payment |
| Payment | Not built | **Existing payment pages** |

---

## Prompt to start Phase 1 (copy-paste)

```
Port Lovable motor journey from C:\Ombrela\ombrella-sparkle-design into C:\Ombrela\customer.

Routes (parallel test path — do NOT touch old motor):
- /v2/motor/v3 — landing (MotorHero, QuoteCard, PlanCards)
- /v2/motor/v3/cover — cover chooser
- /v2/motor/v3/plans — plans list (policy-list equivalent): filters, upsell carousels, offer cards
- /v2/motor/v3/selected — confirm before payment

Use static data from sparkle-design src/lib/motor-data.ts.
Match Lovable UI exactly — icons, spacing, mobile responsive.
Payment pages unchanged — Proceed to payment can be disabled/mock for now.
Do not modify /v2/motor/prequotes or /v2/motor/[id]/policy-list.
```

---

## Quick FAQ

**On landing, are those real products?**  
No. They are **cover type teasers**. Real insurer cards are on the **plans page** (like old policy-list).

**When does it ask for name/phone?**  
After plate confirm (“Yes, this is my car”) — **LeadDialog**. Also **LeadSheet** on plans page if user skipped it.

**Do filters & carousels need backend?**  
**Phase 1:** no — static. **Phase 2:** yes — same filters as old policy-list API.

**Does payment change?**  
**No.** After confirm, jump into existing `/v2/motor/{id}/payments/...` flow.

**Which repo to prompt from?**  
Always **customer**. Reference **ombrella-sparkle-design** for design.

---

## Related: homepage (already done)

Homepage v2 links motor to old prequotes (`/v2/motor/prequotes`). Do not change until Phase 3.

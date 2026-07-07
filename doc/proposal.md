# CardPilot — Product Requirements Document (Proposal)

| | |
|---|---|
| **Product** | CardPilot |
| **Version** | 0.1 (Draft) |
| **Date** | 2026-07-04 |
| **Last updated** | 2026-07-06 |
| **Status** | Proposal — pending review |
| **Target market** | United States |

---

## 1. Overview

CardPilot is a web application (responsive web + PWA first, native mobile app later) that helps US credit card holders answer one question instantly:

> **"Which of my cards should I use for this purchase?"**

The user selects a **spending category** (dining, groceries, gas, online shopping, travel, etc. — user-facing categories, not raw MCC codes), and CardPilot ranks the user's own cards by real reward value for that purchase. In a later phase, CardPilot will also surface **cards worth applying for** — showing how much more the user would earn on the same spending if they held a given card.

A later phase extends this into travel: choosing the best card **combination** when booking hotels and flights, taking card-specific benefits (travel credits, free-night certificates, transfer partners, protections) into account.

### 1.1 Problem statement

- US reward cards have complex, overlapping earning structures: fixed category multipliers, quarterly rotating 5% categories, annual/quarterly spending caps, issuer-specific point currencies with different real-world values.
- Cardholders routinely leave money on the table by swiping the wrong card, forgetting to activate rotating categories, or not realizing a cap has been hit.
- Existing tools are either issuer-specific, US-comparison sites focused only on card acquisition (NerdWallet), or paid apps.

### 1.2 Goals

1. Give an instant, correct "best card for this purchase" answer among the user's own cards (MVP).
2. Keep the reward-rule database accurate and current for the most popular US cards (MVP).
3. Later (Phase 2): quantify the opportunity — show what a new card would add, in dollars.

### 1.3 Non-goals (for MVP)

- User accounts, login, and cloud sync (wallet is local-only; accounts land in Phase 2).
- New-card suggestions / "which card should I apply for?" (Phase 2).
- Transaction syncing / bank account linking (Plaid etc.).
- Automatic MCC detection at point of sale.
- Non-US card ecosystems.
- Affiliate monetization (kept open as a future direction, see §9).
- Native iOS/Android apps (PWA covers mobile in MVP).

---

## 2. Target users

| Persona | Description | Primary need |
|---|---|---|
| **Optimizer** | Holds 3–10 reward cards, active on r/CreditCards, knows what MR/UR points are | Fast, accurate best-card lookup incl. rotating categories and caps |
| **Casual holder** | Holds 1–3 cards, vaguely aware of "5% categories" | Simple guidance; discover that a better card exists for their spending |
| **Points traveler** (later phase) | Redeems points for flights/hotels via transfer partners | Best card/combination for travel bookings and benefit usage |

---

## 3. Scope

### 3.1 MVP (Phase 1)

1. **Wallet (local, no login)**
   - No account or sign-in in MVP. The app is usable immediately.
   - User adds cards to their wallet by searching the CardPilot card database.
   - Wallet — held cards, cap-usage entries, activation states, and point-valuation overrides — is stored in the browser's **localStorage**. Recommendations are computed on-device, so **no user data ever leaves the browser**.

2. **Best-card recommendation (owned cards)**
   - User picks a spending category (and optionally an amount).
   - System ranks the user's cards by **effective cash value** for that purchase.
   - Correctly handles fixed multipliers, quarterly rotating categories, and spending caps.

3. **Card database (curated, self-built)**
   - Internal admin tooling to maintain the reward rules. Seeded first with the product owner's own cards (M0), then grown to the **top ~50 mainstream US cards** (M1).

4. **Points valuation settings**
   - Default cent-per-point values per currency; user can override.

> **MVP is for internal testing / dogfooding first**, not a public launch. Success is judged by hands-on use, not product metrics — there is no KPI framework or usage analytics in MVP.
>
> New-card suggestions ("which card should I apply for?") are **not** in MVP — see §3.2 Phase 2.

### 3.2 Phase 2

- **New-card suggestions**: for the selected category (or the user's overall category mix), show top cards *not* in the wallet, ranked by incremental long-term value (annual fee included; welcome bonus shown separately). See §4.4 for the detailed spec.
- **User accounts & cloud sync**: email + password and Google OAuth sign-in, with the wallet stored server-side and synced across devices. Existing local wallets are migrated into the account on first sign-in.
- Spending-profile dashboard: user enters approximate monthly spend per category → annualized "wallet report card" and optimization suggestions.
- Rotating-category activation reminders (email / push via PWA).
- **MCC-resolution engine**: automatically map a specific merchant to its MCC and each card's precise inclusion/exclusion rules, removing the need for the user to judge merchant-level exceptions manually.
- Native mobile app (React Native/Expo) if PWA proves insufficient.

### 3.3 Future (exploratory)

- **Travel booking optimizer**: best card or card combination for a specific hotel/flight booking, factoring in travel credits, status benefits, free-night certificates, transfer-partner redemption values, and purchase protections.
- Affiliate/referral links on new-card suggestions (with clear compliance disclosure).
- Transaction import (Plaid) for automatic cap tracking and spend profiling.

---

## 4. Functional requirements

### 4.1 Authentication & account

_Deferred to Phase 2._ MVP has **no login**: the app works anonymously and the wallet lives in the browser (see §4.2). Email/password + Google OAuth accounts, cloud sync, and account deletion are specified for Phase 2 (§3.2), at which point local wallets migrate into the account on first sign-in.

### 4.2 Wallet management (MVP — local storage)

| ID | Requirement |
|---|---|
| WAL-1 | User can search the card database by card name or issuer and add a card to their wallet. |
| WAL-2 | User can remove a card from the wallet. |
| WAL-3 | For cards with spending caps, the user can manually record how much of a cap has been used (e.g. "$1,800 of $2,500 this year"). Cap usage resets automatically at the cap period boundary. |
| WAL-4 | For rotating-category cards, the user can mark whether the current quarter's category is activated. |
| WAL-5 | User can override the default point valuation per rewards currency (cents per point). |
| WAL-6 | The entire wallet (WAL-1…WAL-5 state) persists in browser localStorage; no server-side account is involved. The user can clear it, and clearing browser data resets the wallet. |
| WAL-7 | The user can export their wallet to a file and import it, so they can move it between browsers/devices without an account. |
| WAL-8 | First run / empty wallet shows an onboarding empty state that explains the app in one line and guides the user straight into adding their first card; the recommendation screen is disabled (with a prompt to add cards) until at least one card is in the wallet. |

### 4.3 Best-card recommendation — owned cards (MVP)

| ID | Requirement |
|---|---|
| REC-1 | User selects one spending category from a fixed taxonomy (see §5.1); amount is optional (default $100 for illustration). |
| REC-2 | System returns the user's cards ranked by effective cash reward for that purchase (see §6 algorithm). |
| REC-3 | Each result shows: earn rate (e.g. "4× MR"), rewards currency, effective cash value in dollars and as a %, and any caveats (cap nearly exhausted, category not activated, portal-only rate, **merchant exceptions** such as "Walmart/Target excluded from Groceries", etc.). |
| REC-4 | If a cap would be crossed mid-purchase, the value is prorated (pre-cap rate up to the cap, post-cap rate beyond) and the result is annotated. |
| REC-5 | A rotating category that is not activated is ranked at its base rate, with a prompt to activate. |
| REC-6 | When a card's applicable rule carries `merchant_exceptions`, the recommendation surfaces the exception note as a caveat. MVP does not auto-detect the actual merchant's MCC; the ranking uses the friendly category rate and the user judges applicability. |
| REC-7 | Response is deterministic and explainable: the UI can show "why" for every ranking. |

### 4.4 New-card suggestions (Phase 2 — not in MVP)

_Deferred to Phase 2 (§3.2)._ Specced here for completeness; not built in MVP.

| ID | Requirement |
|---|---|
| SUG-1 | For the selected category, show up to N (default 3) cards not in the user's wallet with a higher effective rate than the user's current best card. |
| SUG-2 | Ranking metric is **incremental annual value = (candidate's category earnings − user's current best earnings) − annual fee**, computed against the user's stated or assumed category spend. Welcome bonus is **displayed separately** and never included in the ranking score. |
| SUG-3 | Each suggestion shows: annual fee, earn structure for the category, welcome bonus (headline + spend requirement), and the break-even monthly spend at which the card beats the user's current best. |
| SUG-4 | Suggestions are informational only — no application links, no affiliate tracking (see §9 for future monetization). |

### 4.5 Card database administration (MVP, internal)

| ID | Requirement |
|---|---|
| ADM-1 | Admin users can create/update/retire cards and their reward rules through an internal interface (can be a simple admin UI or authenticated API + scripts in MVP). |
| ADM-2 | Rule changes are versioned with an effective date, so recommendations use the rules in force today. |
| ADM-3 | Quarterly rotating calendars (e.g. Discover it, Chase Freedom Flex) are maintained per card per quarter. |
| ADM-4 | Dataset is seeded in phases: **M0** — the product owner's own cards (enough to dogfood the flow end to end); **M1** — the **top ~50 mainstream US consumer cards** by popularity across major issuers (Chase, Amex, Citi, Capital One, Discover, Bank of America, Wells Fargo, US Bank). |
| ADM-5 | Admins can attach `merchant_exceptions` (included/excluded merchant lists + a human-readable note) to any reward rule, so cards with special MCC scoping (e.g. Amex Gold groceries excluding Walmart/Target) carry accurate caveats. |

---

## 5. Domain model

### 5.1 Spending category taxonomy (user-facing)

A fixed, curated list — not raw MCC. Initial set:

`Dining` · `Groceries` · `Gas` · `EV Charging` · `Online Shopping` · `Wholesale Clubs` · `Drugstores` · `Travel — Flights` · `Travel — Hotels` · `Travel — Other` · `Transit & Rideshare` · `Streaming` · `Entertainment` · `Utilities & Phone` · `Home Improvement` · `Rent` · `Everything Else`

Each category maps internally to the MCC groups issuers actually use; the mapping lives in the card database, not in the UI. The taxonomy is versioned and extensible.

**Merchant-level exceptions.** A single friendly category (e.g. `Groceries`) does *not* cover the same merchants on every card. Issuers scope bonus earning to specific MCCs with their own inclusions/exclusions, for example:

- Amex Gold "U.S. supermarkets" (MCC 5411) **excludes** Walmart, Target, and warehouse clubs.
- Blue Cash "U.S. gas stations" excludes fuel bought inside superstores/warehouse clubs.
- Co-branded cards earn at named merchants (e.g. Amazon Prime card: 5% at Amazon and Whole Foods).

For MVP, CardPilot does **not** build a full MCC-resolution engine. Instead, each reward rule can carry **merchant-level exception notes** (see `merchant_exceptions` in §5.2) that are surfaced as caveats in recommendations and on card pages (e.g. "Walmart/Target usually don't count as Groceries on this card"). The user remains responsible for judging whether a specific merchant qualifies. A precise, automatic MCC-resolution engine is deferred to Phase 2 (§3.2).

### 5.2 Core entities

**Server-side (card catalog — read-only to clients):**

```
Card            id, name, issuer, network, annual_fee, rewards_currency,
                welcome_bonus (amount, spend_req, window), status, image
RewardRule      id, card_id, category, multiplier, unit (points|% cash),
                cap_amount, cap_period (monthly|quarterly|annual|none),
                post_cap_multiplier, requires_activation,
                channel_constraint (e.g. portal-only),
                merchant_exceptions (JSON: included[], excluded[], note),
                effective_from/to
RotationEntry   card_id, year, quarter, categories[]
RewardsCurrency id, name (MR, UR, TYP, C1 miles, cash…), default_cpp
```

**Client-side (browser localStorage — the wallet, no server persistence in MVP):**

```
WalletEntry     card_id, added_at
CapUsage        card_id, rule_id, period_key, amount_used
ActivationState card_id, period_key, activated (bool)
Settings        point_valuations_override (map: currency -> cpp)
```

The `User` / `UserSettings` server-side entities and the account linkage of the client-side wallet are introduced in Phase 2 (§3.2) when accounts and cloud sync land.

### 5.3 Points valuation

- Every rewards currency has a **default cents-per-point (cpp)** value maintained by the CardPilot team (e.g. cash = 1.0, UR = 1.6, MR = 1.7, C1 = 1.5 — values to be set editorially and reviewed quarterly).
- Users can override any currency's cpp in settings; overrides are stored locally (browser) and applied to all of that user's calculations.
- All comparisons and rankings are done in **dollars**, never in raw points.

---

## 6. Recommendation algorithm (MVP)

Recommendation runs **entirely client-side**. The browser holds a locally cached copy of the card catalog (cards, reward rules, rotations, currencies — see §8.2) plus the wallet in localStorage, so it computes the ranking with **no backend round-trip** and works fully offline. The backend's only jobs are distributing/updating the catalog and admin editing; it never sees the wallet. For a purchase of `amount A` in category `C`, for each card in the wallet:

1. **Resolve the applicable rule**: the rule for `C` effective today; for rotating cards, the current quarter's rotation entry; fall back to the card's base rate if no category rule matches.
2. **Apply activation**: if the rule `requires_activation` and the user hasn't marked it activated, use the base rate and attach an "activate to earn X%" note.
3. **Apply caps**: with remaining cap `R` (from CapUsage), earnings = `min(A, R) × multiplier + max(A − R, 0) × post_cap_multiplier`.
4. **Convert to cash**: `value_$ = earned_points × cpp(currency) / 100` (cashback uses cpp = 1.0).
5. **Rank** by `value_$` descending. Ties break by (a) no caveats first, (b) lower annual fee.

Annual fees are **not** part of the per-purchase ranking of owned cards (the fee is sunk once the card is held). Fees enter only the Phase 2 new-card suggestion metric (§4.4 SUG-2).

Every step's inputs are retained in the computed result so the UI can render an explanation ("2,000 pts × 1.6¢ = $32; $500 of quarterly cap remaining"). The same pure function is written in TypeScript for the client; if a server-side implementation is ever needed (e.g. Phase 2 suggestions), the rules live in one place (the catalog) so the logic can be mirrored.

---

## 7. Non-functional requirements

| Area | Requirement |
|---|---|
| Performance | Recommendation is computed client-side over a small cached dataset — effectively instant (<50 ms), no network on the hot path. Catalog sync (§8.2) is the only API call and is cached/versioned. |
| Availability | Single-region deployment acceptable for MVP; the whole app is usable with no login. Once the catalog is cached, **everything — browsing, wallet, and recommendations — works fully offline** from the PWA cache. The API is only needed to fetch catalog updates. |
| Security | HTTPS only; no card *numbers* are ever collected — CardPilot stores card *products*, not PANs. This should be stated prominently in the UI. No credentials are handled in MVP (no login). Auth security (password hashing with argon2/bcrypt, OIDC flow, session tokens) is specified for Phase 2 when accounts land. |
| Privacy | MVP stores **no personal data server-side** and the wallet **never leaves the device** — recommendations are computed locally, so no wallet data is sent to the API at all. The only client→server traffic is anonymous catalog fetches. No sale of data. |
| Data accuracy | Every card page shows "rules last verified" date; a disclaimer notes terms can change and the issuer's terms control. Editorial review cadence: monthly, plus quarterly rotation updates. |
| Disclaimers | A persistent, plain-language disclaimer states CardPilot is **informational only and not financial advice**, that reward values are estimates based on editorial point valuations, and that the issuer's current terms always control. Shown on recommendation results and card pages. |
| Compatibility | Responsive web, mobile-first; PWA installable (manifest + service worker, offline shell with cached card DB read-only). Evergreen browsers. |
| i18n | English-only UI for MVP; copy externalized to allow future localization. |
| Accessibility | WCAG 2.1 AA targets for core flows. |

---

## 8. Technical architecture

### 8.1 Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend | **React + Vite + TypeScript** | SPA + PWA (vite-plugin-pwa). **Owns the recommendation engine** (pure TS module over the cached catalog + wallet). State: TanStack Query for the catalog fetch; router: React Router. UI kit TBD during design. |
| Backend | **Python + FastAPI** | Scope is narrow: serve/version the card catalog and host admin editing. Async, typed (Pydantic v2), auto-generated OpenAPI. No recommendation logic, no user data. |
| ORM / DB | SQLAlchemy 2.x + **PostgreSQL** | Alembic migrations. Card rules stored relationally (not JSON blobs) to keep them queryable and versionable. |
| Auth | **None in MVP** | No login; wallet in browser localStorage. Phase 2 adds fastapi-users or custom JWT + Google OIDC (refresh token in httpOnly cookie). |
| Client storage | Browser localStorage + PWA cache | Wallet, cap usage, activation, cpp overrides; export/import to a JSON file (WAL-7). |
| Admin | FastAPI-based admin (e.g. SQLAdmin) or minimal internal React pages | Enough for a small editorial team in MVP. |
| Deployment | Docker; single VM or PaaS (Railway/Fly/Render) for MVP | CI: GitHub Actions — lint, typecheck, tests on PR. |

### 8.2 API sketch

MVP endpoints are all anonymous and read-only. The client downloads the catalog once, caches it (PWA), and computes recommendations locally — so there is **no recommendation endpoint**.

```
GET  /catalog?since=<version>         # if the catalog is newer than <version>,
                                      #   return the FULL bundle (cards, rules,
                                      #   rotations, currencies) + new version tag;
                                      #   otherwise 304 Not Modified. Omit `since`
                                      #   for a first, unconditional full fetch.
GET  /cards/{id}                      # optional: single card detail (deep links)
/admin/**                             # internal, role-gated (card DB editing)
```

The catalog is **versioned** (e.g. a monotonically increasing `version` / ETag). `since` is a conditional-fetch guard, not a delta request: the response is always the whole bundle (small dataset) or a `304`, so the PWA cheaply checks for updates and invalidates its cache. The wallet, cap usage, activation state, and cpp overrides live only in the browser and are **never sent to the server**. There are no `/auth/*`, `/wallet/*`, or `/recommendations` endpoints in MVP. Accounts, cloud sync, per-user `/settings`, and `POST /suggestions` (new-card suggestions) arrive in Phase 2.

### 8.3 Repository layout (proposed)

```
CardPilot/
├── doc/                # this document, ADRs
├── frontend/           # React + Vite + TS
├── backend/            # FastAPI app, alembic, tests
└── docker-compose.yml  # local dev: api + postgres + frontend
```

---

## 9. Business model (brief)

CardPilot launches as a free tool. Potential future revenue, deliberately out of MVP scope:

- **Affiliate/referral links** on new-card suggestions (requires issuer affiliate program onboarding and clear advertising disclosures; ranking must remain independent of commissions to preserve trust).
- Premium tier candidates: transaction sync, travel booking optimizer, household/multi-player wallets.

No monetization work is planned until the recommendation core proves retention.

---

## 10. Roadmap

| Phase | Scope | Rough target |
|---|---|---|
| **M0** | Repo scaffolding, CI, data model, seed the product owner's own cards | Weeks 1–2 |
| **M1 (MVP)** | Local wallet (no login), client-side owned-card recommendation, valuations, **~50 mainstream cards**, admin tooling, PWA shell | Weeks 3–8 |
| **M2** | New-card suggestions, spend-profile dashboard, rotation reminders, **accounts + cloud sync** (migrate local wallets) | Weeks 9–12 |
| **M3** | Travel booking optimizer (discovery + design spike first), native app decision, affiliate exploration | Q4 2026 |

---

## 11. Risks & open questions

| # | Risk / question | Mitigation / next step |
|---|---|---|
| 1 | **Data freshness**: reward rules change without notice; stale data destroys trust. | Verified-date on every card, monthly review cadence, community "report an error" button. |
| 2 | **Rule-model completeness**: some cards have exotic structures (BoA Preferred Rewards tiers, Amex portal-only 5×, choose-your-own-category cards like Custom Cash). | The rule schema (§5.2) covers caps/rotation/activation/channel; tiered-multiplier support (e.g. relationship bonuses) is explicitly deferred — confirm acceptable for MVP. |
| 3 | **Category ambiguity / special MCC scoping**: a purchase's real MCC may not match the user's chosen category, and cards scope bonuses to different MCC sets (e.g. Amex Gold groceries excludes Walmart/Target). | MVP: `merchant_exceptions` on reward rules surfaced as caveats (§5.1, §4.3 REC-6, §4.5 ADM-5). Phase 2: automatic MCC-resolution engine (§3.2). |
| 4 | **Legal/compliance**: recommending financial products; card marks/images, trademark use, and (later) affiliate advertising rules. | Persistent "informational only, not financial advice" disclaimer (§7 Disclaimers); editorial descriptions + issuer-provided art guidelines; legal review before any affiliate launch. |
| 5 | Default cpp valuations are editorial opinions. | Publish methodology; allow user overrides (already in scope). |
| 6 | **Naming / brand availability**: product and repo are both `CardPilot`, but domain and trademark are unverified. | Check domain and trademark availability before public launch; confirm no conflict with existing "…Pilot" fintech products. |

---

*Prepared with the product owner's decisions of 2026-07-04, updated 2026-07-06: US market · user-selected spending categories · **MVP does owned-card recommendation only; new-card suggestions deferred to Phase 2** · self-built card database (M0 seeds the owner's own cards, M1 grows to ~50 mainstream cards) · internal-testing/dogfood MVP, no KPI/analytics · cash-value normalization with user-adjustable point valuations · FastAPI backend · responsive web/PWA first · **MVP has no login — wallet stored locally in the browser and recommendations computed client-side (backend only distributes the card catalog + hosts admin); accounts + Google OAuth + cloud sync moved to Phase 2** · merchant-level exception caveats in MVP, MCC engine in Phase 2 · rotating categories and caps in MVP · annual fee in suggestion math, welcome bonus displayed separately · monetization deferred.*

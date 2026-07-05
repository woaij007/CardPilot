# CreditPilot — Product Requirements Document (Proposal)

| | |
|---|---|
| **Product** | CreditPilot |
| **Version** | 0.1 (Draft) |
| **Date** | 2026-07-04 |
| **Status** | Proposal — pending review |
| **Target market** | United States |

---

## 1. Overview

CreditPilot is a web application (responsive web + PWA first, native mobile app later) that helps US credit card holders answer one question instantly:

> **"Which of my cards should I use for this purchase?"**

The user selects a **spending category** (dining, groceries, gas, online shopping, travel, etc. — user-facing categories, not raw MCC codes), and CreditPilot ranks the user's own cards by real reward value for that purchase. Beyond the user's wallet, CreditPilot also surfaces **cards worth applying for** — showing how much more the user would earn on the same spending if they held a given card.

A later phase extends this into travel: choosing the best card **combination** when booking hotels and flights, taking card-specific benefits (travel credits, free-night certificates, transfer partners, protections) into account.

### 1.1 Problem statement

- US reward cards have complex, overlapping earning structures: fixed category multipliers, quarterly rotating 5% categories, annual/quarterly spending caps, issuer-specific point currencies with different real-world values.
- Cardholders routinely leave money on the table by swiping the wrong card, forgetting to activate rotating categories, or not realizing a cap has been hit.
- Existing tools are either issuer-specific, US-comparison sites focused only on card acquisition (NerdWallet), or paid apps.

### 1.2 Goals

1. Give an instant, correct "best card for this purchase" answer among the user's own cards.
2. Quantify the opportunity: show what a new card would add, in dollars.
3. Keep the reward-rule database accurate and current for the most popular US cards.

### 1.3 Non-goals (for MVP)

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

1. **Account & wallet**
   - Sign up / sign in with email + password and Google OAuth.
   - User adds cards to their wallet by searching the CreditPilot card database.
   - Wallet is stored server-side and syncs across devices.

2. **Best-card recommendation (owned cards)**
   - User picks a spending category (and optionally an amount).
   - System ranks the user's cards by **effective cash value** for that purchase.
   - Correctly handles fixed multipliers, quarterly rotating categories, and spending caps.

3. **New-card suggestions**
   - For the selected category (or the user's overall category mix), show top cards *not* in the wallet, ranked by incremental long-term value (annual fee included; welcome bonus shown separately).

4. **Card database (curated, self-built)**
   - Internal admin tooling to maintain ~50–100 popular US cards and their reward rules.

5. **Points valuation settings**
   - Default cent-per-point values per currency; user can override.

### 3.2 Phase 2

- Spending-profile dashboard: user enters approximate monthly spend per category → annualized "wallet report card" and optimization suggestions.
- Rotating-category activation reminders (email / push via PWA).
- Native mobile app (React Native/Expo) if PWA proves insufficient.

### 3.3 Future (exploratory)

- **Travel booking optimizer**: best card or card combination for a specific hotel/flight booking, factoring in travel credits, status benefits, free-night certificates, transfer-partner redemption values, and purchase protections.
- Affiliate/referral links on new-card suggestions (with clear compliance disclosure).
- Transaction import (Plaid) for automatic cap tracking and spend profiling.

---

## 4. Functional requirements

### 4.1 Authentication & account (MVP)

| ID | Requirement |
|---|---|
| AUTH-1 | Users can register and sign in with email + password. |
| AUTH-2 | Users can sign in with Google OAuth; accounts with matching verified email are linked. |
| AUTH-3 | Password reset via email. |
| AUTH-4 | Sessions via short-lived JWT access token + refresh token (httpOnly cookie). |
| AUTH-5 | Users can delete their account and all associated data. |

### 4.2 Wallet management (MVP)

| ID | Requirement |
|---|---|
| WAL-1 | User can search the card database by card name or issuer and add a card to their wallet. |
| WAL-2 | User can remove a card from the wallet. |
| WAL-3 | For cards with spending caps, the user can manually record how much of a cap has been used (e.g. "$1,800 of $2,500 this year"). Cap usage resets automatically at the cap period boundary. |
| WAL-4 | For rotating-category cards, the user can mark whether the current quarter's category is activated. |
| WAL-5 | User can override the default point valuation per rewards currency (cents per point). |

### 4.3 Best-card recommendation — owned cards (MVP)

| ID | Requirement |
|---|---|
| REC-1 | User selects one spending category from a fixed taxonomy (see §5.1); amount is optional (default $100 for illustration). |
| REC-2 | System returns the user's cards ranked by effective cash reward for that purchase (see §6 algorithm). |
| REC-3 | Each result shows: earn rate (e.g. "4× MR"), rewards currency, effective cash value in dollars and as a %, and any caveats (cap nearly exhausted, category not activated, portal-only rate, etc.). |
| REC-4 | If a cap would be crossed mid-purchase, the value is prorated (pre-cap rate up to the cap, post-cap rate beyond) and the result is annotated. |
| REC-5 | A rotating category that is not activated is ranked at its base rate, with a prompt to activate. |
| REC-6 | Response is deterministic and explainable: the UI can show "why" for every ranking. |

### 4.4 New-card suggestions (MVP)

| ID | Requirement |
|---|---|
| SUG-1 | For the selected category, show up to N (default 3) cards not in the user's wallet with a higher effective rate than the user's current best card. |
| SUG-2 | Ranking metric is **incremental annual value = (candidate's category earnings − user's current best earnings) − annual fee**, computed against the user's stated or assumed category spend. Welcome bonus is **displayed separately** and never included in the ranking score. |
| SUG-3 | Each suggestion shows: annual fee, earn structure for the category, welcome bonus (headline + spend requirement), and the break-even monthly spend at which the card beats the user's current best. |
| SUG-4 | Suggestions are informational only in MVP — no application links, no affiliate tracking. |

### 4.5 Card database administration (MVP, internal)

| ID | Requirement |
|---|---|
| ADM-1 | Admin users can create/update/retire cards and their reward rules through an internal interface (can be a simple admin UI or authenticated API + scripts in MVP). |
| ADM-2 | Rule changes are versioned with an effective date, so recommendations use the rules in force today. |
| ADM-3 | Quarterly rotating calendars (e.g. Discover it, Chase Freedom Flex) are maintained per card per quarter. |
| ADM-4 | Initial dataset: top 50–100 US consumer cards by popularity across major issuers (Chase, Amex, Citi, Capital One, Discover, Bank of America, Wells Fargo, US Bank). |

---

## 5. Domain model

### 5.1 Spending category taxonomy (user-facing)

A fixed, curated list — not raw MCC. Initial set:

`Dining` · `Groceries` · `Gas` · `EV Charging` · `Online Shopping` · `Wholesale Clubs` · `Drugstores` · `Travel — Flights` · `Travel — Hotels` · `Travel — Other` · `Transit & Rideshare` · `Streaming` · `Entertainment` · `Utilities & Phone` · `Home Improvement` · `Rent` · `Everything Else`

Each category maps internally to the MCC groups issuers actually use; the mapping lives in the card database, not in the UI. The taxonomy is versioned and extensible.

### 5.2 Core entities

```
User            id, email, auth_provider, created_at
UserSettings    user_id, point_valuations_override (JSON)
Card            id, name, issuer, network, annual_fee, rewards_currency,
                welcome_bonus (amount, spend_req, window), status, image
RewardRule      id, card_id, category, multiplier, unit (points|% cash),
                cap_amount, cap_period (quarterly|annual|none),
                post_cap_multiplier, requires_activation,
                channel_constraint (e.g. portal-only), effective_from/to
RotationEntry   card_id, year, quarter, categories[]
RewardsCurrency id, name (MR, UR, TYP, C1 miles, cash…), default_cpp
WalletEntry     user_id, card_id, added_at
CapUsage        wallet_entry_id, rule_id, period_key, amount_used
ActivationState wallet_entry_id, period_key, activated (bool)
```

### 5.3 Points valuation

- Every rewards currency has a **default cents-per-point (cpp)** value maintained by the CreditPilot team (e.g. cash = 1.0, UR = 1.6, MR = 1.7, C1 = 1.5 — values to be set editorially and reviewed quarterly).
- Users can override any currency's cpp in settings; overrides apply to all of that user's calculations.
- All comparisons and rankings are done in **dollars**, never in raw points.

---

## 6. Recommendation algorithm (MVP)

For a purchase of amount `A` in category `C`, for each card in the wallet:

1. **Resolve the applicable rule**: the rule for `C` effective today; for rotating cards, the current quarter's rotation entry; fall back to the card's base rate if no category rule matches.
2. **Apply activation**: if the rule `requires_activation` and the user hasn't marked it activated, use the base rate and attach an "activate to earn X%" note.
3. **Apply caps**: with remaining cap `R` (from CapUsage), earnings = `min(A, R) × multiplier + max(A − R, 0) × post_cap_multiplier`.
4. **Convert to cash**: `value_$ = earned_points × cpp(currency) / 100` (cashback uses cpp = 1.0).
5. **Rank** by `value_$` descending. Ties break by (a) no caveats first, (b) lower annual fee.

Annual fees are **not** part of the per-purchase ranking of owned cards (the fee is sunk once the card is held). Fees enter only the new-card suggestion metric (§4.4 SUG-2).

Every step's inputs are retained in the response payload so the UI can render an explanation ("2,000 pts × 1.6¢ = $32; $500 of quarterly cap remaining").

---

## 7. Non-functional requirements

| Area | Requirement |
|---|---|
| Performance | Recommendation endpoint p95 < 300 ms; recommendation math is pure computation over a small dataset — no external calls on the hot path. |
| Availability | Single-region deployment acceptable for MVP; graceful degradation: read-only card browsing works without login. |
| Security | Passwords hashed (argon2/bcrypt); OAuth via standard OIDC flow; HTTPS only; no card *numbers* are ever collected — CreditPilot stores card *products*, not PANs. This should be stated prominently in the UI. |
| Privacy | Only email + wallet composition + optional spend estimates are stored. Account deletion removes all user data. No sale of data. |
| Data accuracy | Every card page shows "rules last verified" date; a disclaimer notes terms can change and the issuer's terms control. Editorial review cadence: monthly, plus quarterly rotation updates. |
| Compatibility | Responsive web, mobile-first; PWA installable (manifest + service worker, offline shell with cached card DB read-only). Evergreen browsers. |
| i18n | English-only UI for MVP; copy externalized to allow future localization. |
| Accessibility | WCAG 2.1 AA targets for core flows. |

---

## 8. Technical architecture

### 8.1 Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend | **React + Vite + TypeScript** | SPA + PWA (vite-plugin-pwa). State: TanStack Query for server state; router: React Router. UI kit TBD during design. |
| Backend | **Python + FastAPI** | Async, typed (Pydantic v2), auto-generated OpenAPI. |
| ORM / DB | SQLAlchemy 2.x + **PostgreSQL** | Alembic migrations. Card rules stored relationally (not JSON blobs) to keep them queryable and versionable. |
| Auth | fastapi-users or custom JWT + Google OIDC | Refresh token in httpOnly cookie. |
| Admin | FastAPI-based admin (e.g. SQLAdmin) or minimal internal React pages | Enough for a small editorial team in MVP. |
| Deployment | Docker; single VM or PaaS (Railway/Fly/Render) for MVP | CI: GitHub Actions — lint, typecheck, tests on PR. |

### 8.2 API sketch

```
POST /auth/register | /auth/login | /auth/google | /auth/refresh
GET  /cards?query=&issuer=            # public card catalog
GET  /cards/{id}                      # card detail incl. current rules
GET  /wallet                          # user's cards
POST /wallet         DELETE /wallet/{cardId}
PUT  /wallet/{cardId}/cap-usage
PUT  /wallet/{cardId}/activation
GET  /recommendations?category=&amount=      # owned-card ranking
GET  /suggestions?category=&monthlySpend=    # new-card suggestions
GET/PUT /settings/valuations
/admin/**                              # internal, role-gated
```

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

CreditPilot launches as a free tool. Potential future revenue, deliberately out of MVP scope:

- **Affiliate/referral links** on new-card suggestions (requires issuer affiliate program onboarding and clear advertising disclosures; ranking must remain independent of commissions to preserve trust).
- Premium tier candidates: transaction sync, travel booking optimizer, household/multi-player wallets.

No monetization work is planned until the recommendation core proves retention.

---

## 10. Roadmap

| Phase | Scope | Rough target |
|---|---|---|
| **M0** | Repo scaffolding, CI, data model, seed 20 cards | Weeks 1–2 |
| **M1 (MVP)** | Auth, wallet, owned-card recommendation, valuations, 50+ cards, admin tooling, PWA shell | Weeks 3–8 |
| **M2** | New-card suggestions, spend-profile dashboard, rotation reminders | Weeks 9–12 |
| **M3** | Travel booking optimizer (discovery + design spike first), native app decision, affiliate exploration | Q4 2026 |

---

## 11. Risks & open questions

| # | Risk / question | Mitigation / next step |
|---|---|---|
| 1 | **Data freshness**: reward rules change without notice; stale data destroys trust. | Verified-date on every card, monthly review cadence, community "report an error" button. |
| 2 | **Rule-model completeness**: some cards have exotic structures (BoA Preferred Rewards tiers, Amex portal-only 5×, choose-your-own-category cards like Custom Cash). | The rule schema (§5.2) covers caps/rotation/activation/channel; tiered-multiplier support (e.g. relationship bonuses) is explicitly deferred — confirm acceptable for MVP. |
| 3 | **Category ambiguity**: a purchase's real MCC may not match the user's chosen category (e.g. Walmart groceries codes as discount store). | Show MCC caveats on category pages ("Walmart usually doesn't count as Groceries"). |
| 4 | **Legal/compliance**: card marks/images, trademark use, and (later) affiliate advertising rules. | Use editorial descriptions + issuer-provided art guidelines; legal review before any affiliate launch. |
| 5 | Default cpp valuations are editorial opinions. | Publish methodology; allow user overrides (already in scope). |
| 6 | Naming: repo is `CardPilot`, product is `CreditPilot`. | Decide canonical name before public launch; check domain/trademark availability. |

---

*Prepared with the product owner's decisions of 2026-07-04: US market · user-selected spending categories · owned-card + new-card recommendations · self-built card database · cash-value normalization with user-adjustable point valuations · FastAPI backend · responsive web/PWA first · email + Google OAuth · rotating categories and caps in MVP · annual fee in suggestion math, welcome bonus displayed separately · monetization deferred.*

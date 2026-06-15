# Weekend Mobile Projects — Idea Catalog

Agentic workflow: React Native (Expo) for apps/simple games, Unity for complex games, Blender/Inkscape/Affinity Designer for assets. Each idea scoped for a single weekend build by Claude Code with relevant MCPs.

Ideas are grouped by complexity to help you pick the right project for your available time and energy.

---

## Tier 1 — Quick Wins (6–10 hours)

Single-screen concepts, minimal state, few or no external dependencies. Ship in a day.

### 1. Flashback
Connects to your camera roll. Every morning, shows photos from exactly 1, 3, and 5 years ago. A memory lane widget. Tap to share or save. No AI, no albums, no editing.
- **Stack:** React Native + photo library access + widget
- **Why it works:** Apple/Google Photos already do this, but buried under features. A dedicated app that does only this feels magical.

### 2. Shout Out
Party game. Phone measures your shout level. Categories like "worst date story" or "unpopular opinion." Loudest within a time window wins the round. Purely ridiculous.
- **Stack:** React Native + decibel meter + timer
- **Why it works:** Turns any gathering into chaos in 30 seconds. Zero learning curve.

### 3. Loadshedding Countdown
Reads phone battery, learns usage patterns, references schedule, and warns: *"You have 2h 14min. Charge by 4:30 PM."* Not a schedule viewer — a personal electricity survival tool.
- **Stack:** React Native + local storage + timer logic + notifications
- **Why it works:** Battery anxiety is real during outages. A tool that's immediately useful.

### 4. Split Decision
Group decision deadlock breaker. Everyone enters preferences for dinner/movie/activity. App uses ranked-choice voting to find the option that makes the *fewest* people unhappy. Results with funny commentary.
- **Stack:** React Native + ranked-choice algo + share sheet
- **Why it works:** "Where should we eat?" is the most asked, least answered question in group chats.

### 5. Noodle
A fidget toy, not a game. Touch a physics blob — stretch it, twist it, flick it. It wobbles back. Soothing haptics, pastel colors. That's the whole app. Pure sensory satisfaction.
- **Stack:** React Native + Skia/Reanimated + haptics
- **Why it works:** People spend hours in fidget apps. This one has zero UI, zero goals, and a blob that feels alive.

### 6. Time Capsule
Write notes to your future self. Set an unlock delay — a week, a month, a year. A private journal that delivers past-you's thoughts back when you'll need them most. No social features, just you and time.
- **Stack:** React Native + local storage + scheduled notifications
- **Why it works:** Journals are cliché; delayed delivery isn't.

### 7. Sabji Index
Real-time street-market vegetable price tracker. Users report what they paid at Kalimati vs. Asan vs. local bazaar. Creates a "fair price" baseline so you don't get fleeced.
- **Stack:** React Native + submit form + location tagging
- **Why it works:** Vegetable prices vary wildly across Kathmandu. Nobody tracks this.

---

## Tier 2 — Solid Weekend Builds (12–16 hours)

Multiple screens, moderate game logic or backend needs, polish expected. The sweet spot.

### 8. Bhatti Radar
Find and rate local *bhattis* (informal tea/snack joints). Crowd-sourced ratings for authenticity, *guff* quality, and chiyo chow chow crispiness — not hygiene scores. Map-based discovery.
- **Stack:** React Native + map + rating CRUD
- **Why it works:** No app for informal street food culture. Word-of-mouth is the competitor.

### 9. Doko Porter
Tilt-based physics balancing game. You're a porter carrying a loaded *doko* up mountain trails. Dodge landslides, yaks, and potholes. One-thumb infinite runner.
- **Stack:** React Native + tilt sensor + physics + procedural terrain
- **Why it works:** Instantly relatable to anyone who's walked a hill trail. Simple mechanic, deep theme.

### 10. Pocket Alarm
Alarm that only turns off when you scan a specific barcode — your toothpaste, coffee bag, or a book across the room. Forces you to physically get up and engage with the thing you said you'd do.
- **Stack:** React Native + camera/barcode scanning + alarm API
- **Why it works:** Every alarm app is a snooze button. This one is a contract.

### 11. Bagh Chal
The classic Nepali tiger-goat strategy board game (~2000 years old). AI opponent with difficulty levels, local 2-player mode. No polished mobile version exists yet.
- **Stack:** React Native + grid board + game logic + simple AI (minimax)
- **Why it works:** Cultural heritage game with real strategic depth. Zero competition on mobile.

### 12. Word Wedge
Insert one letter into a word to make a new word. Chain reactions score combos. Example: BAT → BAIT → BEAST → BREAST. Daily challenge + endless mode.
- **Stack:** React Native + word list dictionary + animation
- **Why it works:** Dead simple rule, deep vocabulary play, infinite replayability.

### 13. Chiya Guff
Interactive fiction set at a tea shop. Overhear fragments of gossip, scandals, and small-town drama. Piece together stories across sessions. *Coffee Talk* meets *Return of the Obra Dinn*.
- **Stack:** React Native + text branching + story data files (JSON/YAML)
- **Why it works:** Storytelling culture is rich. Text-heavy games are cheap to build and deeply engaging.

### 14. Kathmandu Traffic Zen
You're the traffic cop at a broken intersection. No lights — just hand signals. Direct cars, micros, bikes, cows. Prevent gridlock. Starts zen, becomes chaos.
- **Stack:** React Native + grid + pathfinding + wave difficulty
- **Why it works:** Traffic is the most universally cursed topic. Cathartic to be in control.

### 15. One Word
Collaborative journal with friends. Each person contributes exactly one word per day. Over a year: a chaotic, accidental, beautiful collaborative sentence. No editing, no deleting.
- **Stack:** React Native + backend (Firebase/Supabase) + daily push notification
- **Why it works:** Constraints create art. What 5 friends write at one-word-per-day is always surprising.

---

## Tier 3 — Stretch Goals (16–24 hours)

Backend-heavy, complex game logic, or unfamiliar APIs. Ambitious but still weekend-viable with focus.

### 16. Guff Drop
Location-based ephemeral voice notes. Drop audio at a physical spot — others hear it when nearby. Reviews, stories, jokes pinned to geography. A living audio layer on the city.
- **Stack:** React Native + audio recording + GPS + playback + geo-fencing
- **Why it works:** Voice is faster than typing. Geo-audio is unexplored territory.

### 17. Riff
Musical call-and-response. App plays a short MIDI melody. You hum or whistle it back. Scored on pitch accuracy and rhythm. Daily leaderboard, shareable replays. Like Wordle for your ear.
- **Stack:** React Native + pitch detection + MIDI playback
- **Why it works:** Music games are under-explored on mobile. No instruments needed.

### 18. Tempo Tycoon
Dark-humor management sim. Run a microbus route. Hire conductors who shout better, bribe during *bandhs*, upgrade from a Datsun to a HiAce. *Papers, Please* meets *Mini Metro*.
- **Stack:** React Native + resource management + event cards
- **Why it works:** The microbus economy is a rich, absurd system that every commuter understands intimately.

---

## Quick Reference

| # | Idea | Tier | Type | Key Dependency |
|---|------|------|------|---------------|
| 1 | Flashback | 1 · Quick | Utility | Photo library |
| 2 | Shout Out | 1 · Quick | Game | Decibel meter |
| 3 | Loadshedding Countdown | 1 · Quick | Utility | Notifications |
| 4 | Split Decision | 1 · Quick | Utility | Share sheet |
| 5 | Noodle | 1 · Quick | Utility | Skia / Reanimated |
| 6 | Time Capsule | 1 · Quick | Utility | Scheduled notifications |
| 7 | Sabji Index | 1 · Quick | Utility | Location |
| 8 | Bhatti Radar | 2 · Solid | Utility | Maps + CRUD |
| 9 | Doko Porter | 2 · Solid | Game | Tilt + physics |
| 10 | Pocket Alarm | 2 · Solid | Utility | Barcode scanning |
| 11 | Bagh Chal | 2 · Solid | Game | AI (minimax) |
| 12 | Word Wedge | 2 · Solid | Game | Word dictionary |
| 13 | Chiya Guff | 2 · Solid | Game | Branching story engine |
| 14 | Kathmandu Traffic Zen | 2 · Solid | Game | Pathfinding |
| 15 | One Word | 2 · Solid | Utility | Backend (Firebase) |
| 16 | Guff Drop | 3 · Stretch | Utility | Audio + geo-fencing |
| 17 | Riff | 3 · Stretch | Game | Pitch detection |
| 18 | Tempo Tycoon | 3 · Stretch | Game | Simulation engine |

---

## SaaS Ideas

Full-stack B2B platforms. Longer build cycles (weeks, not weekends). Backend-heavy, multi-tenant, designed for the Nepali market.

### 19. Paymensch
Bring Your Own Payment Gateway (BYOPG) platform — merchants connect their existing eSewa, Khalti, Fonepay, and ConnectIPS accounts to a single API. Universal adapter model: add any gateway in any country by implementing one TypeScript interface. Unified dashboard with full audit trail, analytics, and gateway health monitoring. Monetized via monthly SaaS tiers (not per-transaction fees — avoids PSP licensing). Evolves into aggregated PSP settlement and Stripe-like billing/subscriptions for South Asia.
- **Stack:** Node.js/TypeScript, Fastify API, Next.js dashboard, PostgreSQL + Redis + BullMQ monorepo, Prometheus + Grafana monitoring
- **Why it works:** No unified BYOPG platform exists for Nepal's mid-market. Every business builds 4+ gateway integrations from scratch. India's equivalent (Juspay, Plural) is enterprise-only — no self-serve for the next 10,000 companies. BYOPG requires zero regulatory licensing since we never hold funds.

---

## Money-Makers — Built to Earn First

The earlier tiers optimize for delight and shippability. These optimize for revenue. Every idea here leads with a concrete way it makes money and is filtered for **low upfront maintenance**: on-device where possible, stateless where not, no funds-handling, no heavy ops, no subscription platform that needs a team to babysit. Markets are mixed — some global, some Nepal-specific.

### Mobile — One-time Pro, Freemium & Ads (near-zero server cost)

### 20. Receipt Vault *(Global)*
Snap a receipt; on-device OCR pulls the total, date, and merchant. Tracks warranty windows and return deadlines, nags you before they lapse, and exports a clean CSV/PDF at tax time. Nothing leaves the phone.
- **Stack:** React Native + on-device OCR (ML Kit / Apple Vision) + local storage + notifications
- **Earns:** free up to ~15 receipts; one-time Pro unlock (~$8) for unlimited + export. Optional once-a-year tax-season nudge to convert stragglers.
- **Why it works:** no backend means no recurring cost and no data-privacy liability — the #1 reason people distrust receipt apps. Maintenance is near zero.

### 21. Label Lens *(Global)*
Point the camera at any food or cosmetic label; it flags allergens, additives, or anything on your personal blacklist (gluten, palm oil, parabens). Ingredient matching runs against a bundled on-device database.
- **Stack:** React Native + OCR + on-device ingredient DB + family profiles
- **Earns:** free 3 scans/day; Pro (one-time unlock or cheap subscription) for unlimited scans, custom blacklists, and per-family-member profiles.
- **Why it works:** allergy and clean-eating audiences pay for peace of mind. The database ships with updates occasionally — no live infrastructure to run.

### 22. Subscriptions Radar *(Global)*
Log recurring charges (or forward receipt emails); get warned before a free trial converts and before an annual plan silently renews. No bank linking — manual entry plus email parsing only, which sidesteps the regulatory and maintenance load of Plaid-style integrations.
- **Stack:** React Native + local storage + notifications + optional email-receipt parsing
- **Earns:** free up to 3 subscriptions; one-time Pro for unlimited + price-hike alerts + a shared household view.
- **Why it works:** "I forgot to cancel" is a universal, painful, recurring moment. No bank API = foolproof and cheap to run.

### 23. Sun Rate *(Nepal)*
Daily gold, silver, and forex rates with a genuinely clean calculator: tola↔gram, remittance value, "what's my NPR worth today." Existing rate sites are ad-choked and ugly.
- **Stack:** React Native + one daily public rate fetch + local cache
- **Earns:** a single tasteful banner ad + affiliate links to remittance services (IME, Western Union, Ria).
- **Why it works:** millions check rates daily; nobody's shipped a clean, fast app for it. One scheduled fetch a day is the entire backend.

### 24. Form Filler NP *(Nepal)*
Guided, reusable templates for Nepal's most painful bureaucratic forms — passport, PAN, vehicle renewal, citizenship. Fill your details once; reuse across every form; export a print-ready PDF.
- **Stack:** React Native + form templates (JSON) + local profile store + PDF export
- **Earns:** free for the two most common forms; one-time Pro unlock for the full template library + unlimited saved profiles.
- **Why it works:** everyone dreads these forms and the photocopy-shop "agents" who charge to fill them. Templates change rarely — maintenance is occasional, not constant.

### Micro-SaaS & Tools (stateless or low-ops, recurring revenue)

### 25. Frame Drop *(Global)*
Drag in raw app screenshots; get App Store / Play Store / social-ready images — device frames, captions, every required resolution — in one export. A pure, stateless image pipeline.
- **Stack:** Next.js + serverless image rendering (Satori / Sharp) + Stripe credits
- **Earns:** pay-per-export credits or a cheap monthly tier. No user data to manage; scales to zero between uses.
- **Why it works:** every indie dev hates the App Store screenshot ritual. Stateless = foolproof and near-zero idle cost.

### 26. Status Page Lite *(Global)*
A dead-simple hosted status page plus uptime pings, for indie devs who find Statuspage and BetterStack overkill and overpriced.
- **Stack:** Next.js (static status pages) + a single cron pinger + PostgreSQL + Stripe
- **Earns:** one monitored service free; low monthly tier for more checks + a custom domain.
- **Why it works:** the low end of this market is underserved and price-sensitive. A cron job and static pages mean minimal ops and very low churn.

### 27. Invoice Once *(Global / Nepal)*
Generate clean, branded PDF invoices in the browser — logo, line items, tax presets (incl. Nepal's 13% VAT), and optional payment-reminder emails. No signup needed for a one-off invoice.
- **Stack:** Next.js + serverless PDF generation + optional saved-client store + Stripe
- **Earns:** free single invoices; one-time Pro or a cheap annual plan for saved clients, branding, and recurring invoices.
- **Why it works:** freelancers everywhere hate invoicing and existing tools are bloated. Mostly stateless PDF generation — minimal storage, minimal ops.

### 28. QR Menu *(Nepal / Global)*
A restaurant uploads its menu once and gets a hosted QR-code menu page — no app to install, no POS integration, just a link and a printable QR.
- **Stack:** Next.js (static menu pages) + image hosting + Stripe
- **Earns:** one menu free; cheap annual per venue for multiple menus, photos, and custom branding.
- **Why it works:** set-and-forget tools have famously low churn, and small restaurants won't touch complex POS suites. Static pages are trivial to operate at scale.

---

### Money-Makers Quick Reference

| # | Idea | Type | Market | Monetization | Key Dependency |
|---|------|------|--------|--------------|----------------|
| 20 | Receipt Vault | Mobile utility | Global | One-time Pro | On-device OCR |
| 21 | Label Lens | Mobile utility | Global | Freemium / Pro | On-device OCR + DB |
| 22 | Subscriptions Radar | Mobile utility | Global | One-time Pro | Notifications |
| 23 | Sun Rate | Mobile utility | Nepal | Ads + affiliate | Daily rate feed |
| 24 | Form Filler NP | Mobile utility | Nepal | One-time Pro | PDF export |
| 25 | Frame Drop | Micro-SaaS | Global | Credits / low sub | Image pipeline |
| 26 | Status Page Lite | Micro-SaaS | Global | Low subscription | Cron + Postgres |
| 27 | Invoice Once | Micro-SaaS | Global / Nepal | One-time / annual | PDF generation |
| 28 | QR Menu | Micro-SaaS | Nepal / Global | Annual per venue | Static hosting |

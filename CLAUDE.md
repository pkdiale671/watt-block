# WattBlock

A personal prepaid electricity tracker for South African users. Tracks meter readings and visualises usage against Eskom's inclining block tariff system, so users know exactly which tariff block they're in and how close they are to the next (more expensive) tier.

## Problem

Eskom's inclining block tariff (IBT) charges progressively more as monthly consumption increases. But prepaid meters only show a declining balance—users have no visibility into which block they're currently in, when they'll tip into a more expensive tier, or how much their next unit of electricity actually costs. WattBlock solves this by letting users log meter readings and see their block position visualised.

## Stack

- React + Vite
- TypeScript
- Tailwind CSS v4
- Supabase (auth, database, RLS)
- TanStack Query (for data fetching/caching)
- React Router (or TanStack Router)

## Design Direction

**Fintech style (Monzo/Revolut aesthetic):**
- Bold coral accent color (`#FF5A5F` primary, `#FF385C` hover)
- Light background (`slate-50`)
- Rounded everything (`rounded-3xl` on cards, `rounded-2xl` on buttons)
- Friendly, conversational copy (not clinical)
- Emoji used sparingly for personality
- Large, bold numbers as heroes
- Cards with subtle shadows (`shadow-lg shadow-slate-200/50`)
- The header uses coral gradient with rounded bottom edge, main content cards overlap it

**Tone of UI copy:**
- "You've spent" not "Total expenditure"
- "What does your meter say?" not "Enter meter reading"
- Block names: "Cheap 😎", "Normal 👍", "Pricey 😬", "Ouch 🔥"
- Friendly messages based on usage: "You're cruising in the cheap zone! 🎉"

## Data Model (Supabase)

### meters table
Stores prepaid meters. A user can have multiple meters (main house, cottage, etc.).

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key, auto-generated |
| user_id | uuid | References auth.users(id), cascade delete |
| name | text | User-friendly name, e.g., "Main House" |
| meter_number | text | Optional physical meter number |
| tariff_config | jsonb | Block tariff structure (limits and rates) |
| created_at | timestamptz | Auto-generated timestamp |

**Default tariff_config (City Power Joburg):**
```json
[
  {"limit": 350, "rate": 2.4986, "label": "Cheap 😎"},
  {"limit": 500, "rate": 3.06, "label": "Normal 👍"},
  {"limit": null, "rate": 3.23, "label": "Pricey 😬"}
]
```
- `limit`: Upper kWh threshold for this block (null = unlimited)
- `rate`: Cost per kWh in Rands (excl. VAT)
- `label`: Friendly UI label with emoji

### readings table
Stores meter readings. Each reading belongs to a specific meter.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key, auto-generated |
| user_id | uuid | References auth.users(id), cascade delete |
| meter_id | uuid | References meters(id), cascade delete |
| reading | numeric | The actual meter reading number |
| date | date | Date the reading was taken |
| created_at | timestamptz | Auto-generated timestamp |

### Row Level Security
Both tables have RLS enabled. Users can only see/insert/update/delete their own data. Policies check `auth.uid() = user_id`.

## Inclining Block Tariffs (IBT)

Electricity in South Africa is priced using inclining block tariffs—the more you use, the higher the rate per kWh. Both Eskom (for direct customers) and municipalities (for most urban users) use this structure, but the actual rates and block thresholds differ by provider.

### How IBT works
- Block 1: Cheapest rate for low usage (lifeline/subsistence)
- Block 2-3: Progressive increases for moderate usage
- Block 4+: Highest rates for heavy consumption (some providers only have 3 blocks)

### City Power Johannesburg — Residential Prepaid High (2025/2026)

This is the default tariff for the app (user's actual provider).

| Block | Range (kWh) | Rate (R/kWh) | UI Label |
|-------|-------------|--------------|----------|
| 1 | 0–350 | R2.4986 | Cheap 😎 |
| 2 | 351–500 | R3.06 | Normal 👍 |
| 3 | 500+ | R3.23 | Pricey 😬 |

Rates are exclusive of VAT (15%) and effective from 1 July 2025.

Note: City Power also charges a fixed service charge of ~R200/month for Prepaid High customers, separate from the per-kWh charges. This is not currently tracked in the app but could be added later.

### Example: Eskom Homelight (2024/2025)

For comparison, Eskom direct customers have a different structure:

| Block | Range (kWh) | Rate (R/kWh) |
|-------|-------------|--------------|
| 1 | 0–100 | ~R1.46 |
| 2 | 101–350 | ~R1.89 |
| 3 | 351–600 | ~R2.47 |
| 4 | 600+ | ~R3.12 |

### Other municipalities
Each municipality publishes their own tariff schedule annually (usually July, aligned with municipal financial year). Examples:
- City of Tshwane
- City of Cape Town
- eThekwini (Durban)

### Tariff storage approach
Tariff config is stored as a JSON column (`tariff_config`) on the `meters` table with City Power Joburg rates as the database default.

**MVP scope:**
- Tariff config is set automatically when a meter is created (uses DB default)
- No UI for editing tariff config yet—that's a future feature

**Future enhancements:**
- Allow users to manually adjust rates when municipalities publish updates
- Select from a predefined list of municipality tariffs
- Support different tariffs per meter (for users with properties in different municipalities)

## Key Features (MVP)

1. **Auth**: Email/password signup and login via Supabase Auth
2. **Add meter**: User creates at least one meter on first login
3. **Log reading**: User enters current meter reading, app calculates usage since last reading
4. **Dashboard**: Shows current block, usage this month, estimated cost, progress to next block
5. **Reading history**: List of past readings with usage deltas
6. **Purchase simulator**: Estimate cost for a desired kWh purchase based on current block position

### Purchase Simulator (Feature Detail)

The problem: Prepaid users buy electricity in Rands ("give me R200"), but don't know how that translates to kWh—especially when near a block boundary where rates change.

**Mode 1: kWh → Rands (MVP)**
User inputs how many units they want to buy. App calculates:
- Total cost
- Breakdown by block (e.g., "50 kWh @ R2.50 = R125, 50 kWh @ R3.06 = R153")
- Where they'll end up after the purchase (new block position)

**Mode 2: Rands → kWh (future)**
User inputs their budget. App calculates:
- How many units they'll receive
- Block breakdown
- New position after purchase

**Calculation logic:**
```
Given: currentUsage = 300 kWh, desiredPurchase = 100 kWh
Blocks: [{limit: 350, rate: 2.4986}, {limit: 500, rate: 3.06}, {limit: null, rate: 3.23}]

Walk through:
1. Currently in Block 1 (300 < 350)
2. Space left in Block 1: 350 - 300 = 50 kWh
3. First 50 kWh @ R2.4986 = R124.93
4. Remaining 50 kWh spills into Block 2 @ R3.06 = R153.00
5. Total: R277.93
6. New position: 400 kWh (now in Block 2)
```

**UI approach (fintech style):**
- Slider or input field for desired kWh/Rands
- Real-time updating cost estimate as user adjusts
- Visual breakdown showing blocks with portions highlighted
- Friendly copy: "That'll cost you R277.93 — here's the breakdown 👇"

## Usage Calculation Logic

Usage is derived by comparing consecutive readings:
```
usage = current_reading - previous_reading
```

Monthly usage is the sum of all usage values within the current billing month (typically 1st to end of month, but could be configurable).

Block position is determined by comparing monthly usage against the block thresholds.

### Monthly reset (no cronjob needed)

The block position is **calculated**, not stored. When a new month starts, the app simply queries readings for that month—since there are few/no readings yet, monthly usage is 0 or low, which automatically puts the user back in Block 1.

There's no kWh "rollover" between months. Each month is independent. The meter reading itself keeps climbing (45000, 45200, 45500...), but monthly usage resets because we only sum readings within the current month's date range.

**Edge case:** If a reading straddles two months (last reading Jan 28th, next reading Feb 3rd), attribute all usage to the month of the newer reading. Note in UI: "For best accuracy, log a reading near the start of each month."

### Cost calculation example

```
Example: 450 kWh total (City Power Joburg rates)
- First 350 kWh × R2.4986 = R874.51
- Next 100 kWh × R3.06 = R306.00
- Total = R1,180.51 (excl. VAT)
```

## File Structure (suggested)

```
src/
├── components/
│   ├── ui/                  # Reusable UI components
│   ├── Dashboard.tsx        # Main dashboard view
│   ├── ReadingForm.tsx      # Form to log a reading
│   ├── BlockProgress.tsx    # Visual block progression
│   ├── ReadingHistory.tsx
│   └── PurchaseSimulator.tsx # Cost estimator for planned purchases
├── hooks/
│   ├── useReadings.ts       # TanStack Query hooks for readings
│   ├── useMeters.ts         # TanStack Query hooks for meters
│   └── useAuth.ts           # Auth state hook
├── lib/
│   ├── supabase.ts          # Supabase client init
│   ├── tariffs.ts           # Block tariff config and calculation logic
│   └── utils.ts             # Date helpers, formatters
├── pages/
│   ├── Login.tsx
│   ├── Signup.tsx
│   └── Home.tsx
└── App.tsx
```

## Environment Variables

```
VITE_SUPABASE_URL=https://xxxxx.supabase.co
VITE_SUPABASE_KEY=eyJ...
```

## Commands

```bash
npm run dev      # Start dev server
npm run build    # Production build
npm run lint     # Run ESLint
```

## Notes

- This is a learning project for Supabase, so prioritise understanding over speed
- Start with single-meter flow, add multi-meter switching later
- User is on City Power Johannesburg (Prepaid High) — default tariff config should reflect this
- Tariff rates should eventually be user-configurable or location-aware (municipal vs Eskom)
- PWA capabilities (offline, installable) are a future enhancement
- VAT (15%) is excluded from the rates stored in tariff_config — decide whether to apply it in calculations or UI display
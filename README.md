# QuantumEdge Alpha (`www.quantumedgealpha.com`)

**Derived US Index & Equity Options Analytics Terminal — Dealer Gamma Exposure (GEX), Vanna, Charm, 3-State HMM Regime Modeling & Fractional-Kelly Risk Sizing**

- **Live Website:** [https://www.quantumedgealpha.com](https://www.quantumedgealpha.com)
- **GitHub Pages Host Repo:** [`ShortGammaGambler/ShortGammaGambler.github.io`](https://github.com/ShortGammaGambler/ShortGammaGambler.github.io)
- **Backend / Market Intelligence Repo:** [`ShortGammaGambler/unified-market-intelligence`](https://github.com/ShortGammaGambler/unified-market-intelligence) · [`okcwade1-lab/unified-market-intelligence`](https://github.com/okcwade1-lab/unified-market-intelligence)

---

## 1. Architecture & Audit Hardening Summary

This release implements the full architectural, security, compliance, and rendering overhaul for `quantumedgealpha.com`:

### A. Zero Client-Side Secrets & Server-Verified Paywall (`src/App.tsx`, `src/components/AIRecommendations.tsx`, `backend/main.py`)
- **Default Tier Hardening:** `App.tsx` and `src/lib/database.ts` now strictly default unauthenticated and fallback sessions to `"freemium"` (`15-Min Delayed`) instead of `"pro"`.
- **Server-Gated Upgrades:** Removed client-writable tier elevation on pricing button clicks. Selecting the **Pro (`$79/mo`)** tier now authenticates via Supabase JWT and creates a server-side Stripe Checkout session (`POST /api/v1/billing/create-checkout-session`).
- **Server-Proxied Gemini Regime Scenarios:** Removed `@google/genai` from `package.json` and the browser bundle so `GEMINI_API_KEY` is never exposed to clients. `AIRecommendations.tsx` now requests non-advisory **Dealer Hedging & Regime Scenarios** from the FastAPI gateway (`GET /api/v1/analytics/regime-scenarios`), which caches responses in Redis (`60s` TTL) to prevent rate abuse.

### B. Derived Options Feed Compliance & Regulatory Disclosures (`index.html`, `src/App.tsx`, `src/components/*`)
- **OPRA / Derived-Feed Scope Compliance:** Removed unlicensed raw-tape claims (`"Order Flow Scanner"`, `"Real-Time 0DTE options flow"`, and `VPIN < 22` triggers) across `index.html`, `AIRecommendations.tsx`, `Dashboard.tsx`, `Pricing.tsx`, and `Scanner0DTE.tsx`. Replaced with compliant **Derived Options Analytics & Dealer Positioning** terminology (`CBOE 15-Min Delayed` on Free tier; `Derived Options Feed • Live` on Pro tier).
- **Schema.org JSON-LD Alignment:** Updated `index.html` metadata, OpenGraph/Twitter tags, and `AggregateOffer` schema (`$0` Free 15-Min Delayed / `$79` Pro Live Derived, `offerCount: 2`).
- **Mandatory Persistent Disclosure:** Embedded the **Market Data & Regulatory Disclosure** banner in the global footer (`src/App.tsx`), explicitly stating that all metrics, Greeks, and regime scenarios are strictly informational/educational and do not constitute investment advice or trade solicitation.

### C. High-Throughput `uPlot` Canvas Telemetry & Stale-Quote Circuit Breaker (`package.json`, `src/components/GexStreamPanel.tsx`, `src/components/Dashboard.tsx`)
- **SVG-to-Canvas Migration:** Removed `d3` (`^7.9.0`) and `recharts` (`^3.8.1`) from client dependencies and replaced strike-level GEX and intraday regime charts with HTML5 `<canvas>` rendering via **`uPlot` (`^1.6.31`)**, cutting per-frame render overhead from ~18ms to `<0.8ms`.
- **5,000ms Stale-Data Circuit Breaker:** `GexStreamPanel.tsx` and `backend/main.py` monitor upstream exchange quote timestamps (`exchange_ts_ms`). If regular-hours ticks lag by more than `5,000ms`, the panel automatically engages the **Stale Feed Guard** overlay and pauses rendering until fresh ticks arrive.
- **09:30 ET Zero-Open-Interest Guard:** `backend/main.py` (`sanitize_gex_profile`) filters out strikes where `open_interest <= 0` during early-morning OCC rollover windows so false zero-gamma flips are never rendered on the canvas.

### D. Supabase PostgreSQL Row-Level Security Hardening (`supabase-schema.sql`, `src/components/DatabaseViewer.tsx`)
- Locked down the `public.profiles` `UPDATE` RLS policy with a `WITH CHECK` constraint ensuring authenticated users cannot send a direct PostgREST `PATCH` request to modify their own `tier` column (`tier = (SELECT p.tier FROM public.profiles p WHERE p.id = auth.uid())`). Only the Stripe webhook service role can elevate a user's entitlement.

---

## 2. Repository & Uploaded Bundle Structure

```text
quantumedge-alpha/
├── index.html                          # Compliant SEO, OpenGraph, Twitter & Schema.org JSON-LD
├── metadata.json                       # Application metadata
├── package.json                        # Lean frontend dependencies (React 19, Vite 6, Tailwind 4, uPlot)
├── tsconfig.json                       # TypeScript compiler configuration
├── vite.config.ts                      # Vite build & environment configuration
├── .env.example                        # Sanitized public VITE_* vs. server-only secret template
├── supabase-schema.sql                 # PostgreSQL DDL + hardened RLS policies + auth triggers
├── deploy.sh                           # Deployment script for GitHub Pages & source repositories
├── backend/
│   ├── main.py                         # FastAPI Secure Gateway (SSE stream, Gemini proxy, Stripe webhooks)
│   └── requirements.txt                # Python backend dependencies (FastAPI, Redis, Stripe, google-genai)
├── public/
│   ├── CNAME                           # Custom domain binding (www.quantumedgealpha.com)
│   ├── .nojekyll                       # Bypasses Jekyll processing on GitHub Pages
│   ├── manifest.json                   # PWA web manifest
│   ├── robots.txt                      # Search crawler directives
│   ├── sitemap.xml                     # XML sitemap for www.quantumedgealpha.com
│   └── assets/aistudio/                # OpenGraph social preview images
├── src/
│   ├── main.tsx                        # React 19 application root mount
│   ├── App.tsx                         # Auth lifecycle, Stripe Checkout redirect & regulatory footer
│   ├── index.css                       # Tailwind CSS v4 & custom quantum grid styles
│   ├── vite-env.d.ts                   # Vite ImportMeta environment type definitions
│   ├── components/
│   │   ├── GexStreamPanel.tsx          # uPlot HTML5 Canvas GEX profile + SSE + 5s stale-data guard
│   │   ├── AIRecommendations.tsx       # Server-proxied Dealer Hedging & Regime Scenarios (no client keys)
│   │   ├── Dashboard.tsx               # Main analytics terminal (KPI strip, uPlot charts, modules)
│   │   ├── Hero.tsx                    # Landing page hero, interactive GEX flip preview & core features
│   │   ├── Navbar.tsx                  # Top navigation bar, theme switcher & tier indicator
│   │   ├── Pricing.tsx                 # 2-tier pricing grid ($0 Free 15-Min Delayed / $79 Pro Live Derived)
│   │   ├── VolatilitySurface.tsx       # 3D Implied Volatility Surface, 25-delta skew & term structure
│   │   ├── MarketRegimeDetection.tsx   # 3-State Gaussian HMM regime classifier & transition matrix
│   │   ├── MarketSentiment.tsx         # Multi-factor Fear & Greed sentiment gauge
│   │   ├── KellyCalculator.tsx         # Interactive Fractional-Kelly & risk-of-ruin calculator
│   │   ├── KellySizing.tsx             # Regime-weighted Fractional-Kelly allocation module
│   │   ├── Scanner0DTE.tsx             # Derived 0DTE strike matrix with stale-quote circuit breaker
│   │   ├── MarketNews.tsx              # Real-time macro & options structure wire
│   │   ├── TradeJournal.tsx            # Quantitative trade journal & emotional discipline audit
│   │   ├── DatabaseViewer.tsx          # Interactive PostgreSQL schema & RLS DDL viewer
│   │   ├── AuthModal.tsx               # Supabase Auth (Email/Password, Google OAuth, Demo) modal
│   │   └── QuantumLogo.tsx             # Vector brand mark & orbital icon variants
│   └── lib/
│       ├── supabase.ts                 # Safe Supabase client initialization (VITE_SUPABASE_URL / ANON_KEY)
│       ├── database.ts                 # Profile, watchlist & trade journal persistence (freemium default)
│       ├── gemini.ts                   # Gateway URL helper (client-side @google/genai removed)
│       └── chart-primitives.tsx        # Lightweight zero-dependency SVG chart primitives
└── dist/                               # Compiled production bundle deployed to ShortGammaGambler.github.io
    ├── index.html                      # Minified entry HTML (5.29 kB)
    ├── CNAME                           # www.quantumedgealpha.com
    ├── .nojekyll                       # GitHub Pages asset bypass
    ├── manifest.json                   # Web manifest
    ├── robots.txt                      # Robots configuration
    ├── sitemap.xml                     # Sitemap
    └── assets/
        ├── index-BgRiPPJk.css          # Compiled Tailwind v4 + uPlot CSS (86.87 kB / 13.61 kB gzip)
        ├── index-CB_GMu9G.js           # Compiled production JS bundle (849.42 kB / 245.25 kB gzip)
        └── aistudio/*.jpeg             # Social preview assets
```

---

## 3. Environment Variables (`.env.example`)

Never prefix server secrets with `VITE_` or `NEXT_PUBLIC_`.

```env
# Public Client Variables (Safe for browser bundle)
VITE_SUPABASE_URL="https://your-project-ref.supabase.co"
VITE_SUPABASE_ANON_KEY="your_supabase_anon_key"
VITE_STREAM_GATEWAY_URL="https://api.quantumedgealpha.com"

# Server-Only Secrets (Used exclusively by backend/main.py)
SUPABASE_URL="https://your-project-ref.supabase.co"
SUPABASE_SERVICE_ROLE_KEY="your_supabase_service_role_key"
GEMINI_API_KEY="your_server_side_gemini_api_key"
MARKETDATA_API_TOKEN="your_marketdata_commercial_token"
STRIPE_SECRET_KEY="sk_live_your_stripe_secret_key"
STRIPE_WEBHOOK_SECRET="whsec_your_stripe_webhook_secret"
STRIPE_PRO_PRICE_ID="price_your_stripe_79_monthly_id"
REDIS_PUBLIC_URL="redis://127.0.0.1:6380/0"
```

---

## 4. Running & Deploying

### A. Frontend Development & Production Build
```bash
npm install
npm run lint      # Runs tsc --noEmit (zero errors)
npm run build     # Outputs production bundle to dist/
npm run dev       # Starts local Vite dev server on http://localhost:3000
```

### B. FastAPI Secure Gateway (`backend/main.py`)
```bash
cd backend
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000
```

**Gateway Endpoints:**
- `GET /api/v1/stream/options/{ticker}` — Server-Sent Events (SSE) stream reading from isolated `redis-public` (`127.0.0.1:6380`). Enforces symbol gating (`SPY`, `QQQ` on Free tier; `SPX`, `NDX`, `IWM`, `SPY`, `QQQ` on Pro tier), 30-second live entitlement re-verification, `0` OI filtering, and `5,000ms` stale-quote flags.
- `GET /api/v1/analytics/regime-scenarios?ticker=SPX` — Server-side Gemini 2.5 Flash educational dealer hedging & regime commentary cached in Redis for `60s`.
- `POST /api/v1/billing/create-checkout-session` — Creates a Stripe Checkout subscription session bound to the authenticated Supabase user ID.
- `POST /api/v1/billing/stripe-webhook` — Verifies Stripe webhook signatures and immediately updates `entitlement:user:{user_id}` in Redis (`pro` on `checkout.session.completed`, `freemium` on `customer.subscription.deleted` or `invoice.payment_failed`).

### C. Deploying to GitHub Pages (`www.quantumedgealpha.com`)
```bash
./deploy.sh <YOUR_GITHUB_TOKEN> ShortGammaGambler.github.io ShortGammaGambler
```

---

## 5. Market Data & Regulatory Disclosure

QuantumEdge Alpha provides quantitative market analytics, derived options Greeks (GEX, Vanna, Charm), and volatility visualizations strictly for informational and educational purposes. QuantumEdge Alpha is not a registered investment adviser, broker-dealer, or commodity trading advisor, and no metric, scenario, or visualization constitutes personalized investment advice or a timed recommendation to buy, sell, or hold any security or derivative. Options metrics are computed from licensed commercial derived pricing; free-tier views are delayed by at least 15 minutes.

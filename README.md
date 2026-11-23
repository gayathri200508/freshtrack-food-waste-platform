# FreshTrack: AI-Powered Food Waste Management Platform

Surplus identification and redistribution optimization for restaurants, supermarkets and food businesses, with NGOs and food banks as recipients.

**Stack:** Python · FastAPI · SQLAlchemy · SQLite (PostgreSQL-ready) · Prophet · pandas · NumPy · APScheduler · vanilla JavaScript · html5-qrcode · pytest · Docker

---

## What it does

| Module | Features |
|---|---|
| **Inventory & expiry (Weeks 1–2)** | Manual entry, CSV bulk upload with row-level validation, POS REST API with FEFO (first-expiring, first-out) deduction, barcode/QR scanning, transaction history, donation log |
| **AI prediction (Weeks 3–4)** | Per-category demand forecasting (Prophet vs 7-day baseline, best model chosen), waste risk score 0–100, Top 20 risk page, smart reorder with storage-capacity caps |
| **Redistribution & impact (Weeks 5–6)** | Surplus listings, NGO profiles, explainable NGO matching, pickup slot requests, confirm / reject / picked-up workflow, printable pickup pass with QR, impact metrics and certificate |

### Key rules
- **Food safety:** expired items cannot be donated. Only waste or compost is allowed.
- **Matching:** scored 0–100 as proximity 50% + dietary and category fit 25% + capacity fit 25%. Hard rules: veg-only NGOs never receive meat or egg food; chilled and frozen food needs cold transport; NGOs more than 25 km away are excluded. Every score shows its reasons.
- **Reorder:** `reorder qty = max(0, 7-day forecast × 1.2 − usable stock)`. Usable stock counts non-expired batches only.
- **Expiry alerts:** created at 7, 3 and 1 day before expiry, and for expired items. Checked hourly.
- **Multi-tenant:** every table is scoped by `business_id`. One business cannot read another's data (tested).
- **Impact estimates:** 1 meal ≈ 0.4 kg food; CO₂e = 2.5 kg per kg of food diverted. Both are estimates, not measured values.

---

## Quick start (Windows)

```powershell
cd foodwaste
python -m venv venv
venv\Scripts\Activate.ps1
pip install -r requirements.txt
python -m uvicorn app.main:app --reload
```

Open **http://127.0.0.1:8000**.

Use `python -m uvicorn` on Windows if `uvicorn.exe` is blocked by security policy.

### Demo accounts (created automatically on startup, local demo only)

| Role | Email | Password |
|---|---|---|
| Store / restaurant | `manager@worldmart.com` | `password123` |
| NGO / food bank | `partner@feedingindia.org` | `password123` |

### Demo flow
1. Sign in as the store account. Upload `sample/world_food_inventory.csv` on the **CSV upload** tab (1,383 batches).
2. **Marketplace:** post a surplus listing and check the **Top 3 NGOs**.
3. Sign in as the NGO account. Save the NGO profile, then claim the listing and choose a pickup slot.
4. Back in the store account: **Confirm**, then **Mark picked up**.
5. Open the **Pickup pass** (NGO side) and the **Impact** tab (certificate download).

Sales history is simulated the first time a business needs it, so reorder suggestions work on demo data.

---

## Tests

```powershell
python -m pytest -q
```

Expected: `12 passed`. Covers inventory, CSV validation, POS, scanning, alerts, tenant isolation, and the marketplace rules and pickup workflow.

Load test (server must be running):

```powershell
python scripts\load_test.py --email manager@worldmart.com --password password123 --requests 200 --workers 20
```

---

## API

Interactive docs: **http://127.0.0.1:8000/docs**

| Area | Main endpoints |
|---|---|
| Auth | `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/me` |
| Inventory | `GET/POST /api/inventory`, `POST /api/inventory/upload-csv`, `GET /api/inventory/template`, `POST /api/inventory/{id}/donate`, `POST /api/inventory/{id}/waste` |
| POS | `POST /api/v1/pos/sales`, `POST /api/v1/pos/stock` (API key) |
| Scanning | `GET /api/scan/lookup/{barcode}`, `POST /api/scan/update` |
| Prediction | `GET /api/predict/risk`, `GET /api/predict/forecast`, `GET /api/predict/reorder`, `GET /api/predict/capacity` |
| Alerts & dashboard | `POST /api/expiry/run`, `GET /api/alerts`, `GET /api/dashboard` |
| Marketplace | `POST/GET /api/market/listings`, `GET /api/market/listings/{id}/match`, `GET/POST /api/market/ngo-profile`, `POST/GET /api/market/claims`, `POST /api/market/claims/{id}/confirm`, `…/reject`, `…/picked-up`, `GET /api/market/claims/{id}/slip`, `GET /api/market/impact`, `GET /api/market/impact/certificate` |

---

## Project structure

```
foodwaste/
├── app/
│   ├── main.py            # FastAPI app, startup, scheduler, static files
│   ├── models.py          # Tables (multi-tenant)
│   ├── database.py        # DB connection (DATABASE_URL)
│   ├── security.py        # Auth tokens
│   ├── services.py        # Stock logic (FEFO, waste, donate)
│   ├── expiry.py          # Expiry alerts
│   ├── taxonomy.py        # Food categories, storage, shelf life
│   ├── seed.py            # Demo accounts
│   ├── ml/
│   │   ├── risk.py        # Waste risk score and forecast fields
│   │   └── sales.py       # Simulated sales history and baseline forecast
│   └── routers/           # auth, inventory, pos, scan, predict, market, insights
├── static/                # Web UI: index.html, app.js, market.js, scan.html
├── scripts/               # Sample data, forecast, load test
├── sample/                # Sample inventory CSVs
├── tests/                 # pytest suites
├── Dockerfile             # Container build
├── render.yaml            # Render blueprint (cloud deploy)
├── DEPLOY.md              # Deployment steps
└── requirements.txt
```

---

## Deployment

- **Local demo:** see Quick start. For phones on the same Wi-Fi, run with `--host 0.0.0.0` and open `http://<laptop-ip>:8000`.
- **Cloud:** `Dockerfile` and `render.yaml` are ready. See `DEPLOY.md`. The free tier uses ephemeral SQLite, so data resets on redeploy. Use PostgreSQL (`DATABASE_URL`) for a permanent deployment.

Configuration:

| Variable | Purpose | Default |
|---|---|---|
| `DATABASE_URL` | Database connection | SQLite file |
| `SECRET_KEY` | Token signing | dev value (set in production) |
| `DISABLE_SCHEDULER` | `1` turns off the hourly expiry job | `0` |
| `EXPIRY_CHECK_MINUTES` | Expiry job interval | `60` |

---

## Known limitations

- **Matching** uses explainable rules, not a learned model. NGO distance needs coordinates.
- **One claim per listing.** A partial pickup needs a new listing.
- **Impact figures** (meals, CO₂e, value) are estimates based on stated assumptions.
- **Sample data is simulated.** Forecast accuracy describes the demo data, not real sales.
- **LSTM** is not built. Forecasting uses Prophet and a 7-day baseline.
- **Mobile:** the UI is responsive. A native Flutter app is set up in `freshtrack_flutter_lib` but needs a machine where its build tool is allowed to run.
- **Notifications** are in-app. Email is optional and needs SMTP settings.
- **React** is not used; the frontend is vanilla JavaScript.

---

## Roadmap

- Native mobile app (Flutter) on a supported build machine
- PostgreSQL for permanent cloud data
- Push notifications for expiry and pickup alerts
- LSTM comparison for forecasting
- Discount actions for near-expiry stock
- NGO verification workflow

---

## Milestone status

| Milestone | Status |
|---|---|
| M1 · Inventory & expiry tracking | ✅ Complete |
| M2 · AI waste prediction | ✅ Complete (Prophet + baseline, risk score, reorder) |
| M3 · Redistribution marketplace & impact | ✅ MVP working end to end |
| M4 · Final testing, documentation, presentation | 🔄 In progress |

---

## Team

Gollapolu Gayathri · Batch 4 (AI Domain) · Infosys Springboard Virtual Internship 7.0

## License

Academic project for internship evaluation.

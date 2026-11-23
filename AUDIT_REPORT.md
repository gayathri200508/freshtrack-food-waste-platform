# Audit Report — AI-Powered Food Waste Management Platform
**Infosys Springboard Internship · Milestone 2 Deadline: 1 Nov 2026**

---

## 1. Feature Audit Table

### Milestone 1: Inventory & Expiry Tracking

| Requirement | Status | Evidence (file:line) | Gap |
|:---|:---|:---|:---|
| 1.1 Manual inventory entry | Done | `food_management/views.py` → `inventory_create`, `food_management/forms.py` → `InventoryForm` | — |
| 1.2 CSV bulk upload with row-wise error report | Done | `views.py` → `inventory_import`; per-row `rejected_rows` list shown via `messages.warning` | — |
| 1.3 POS REST API (API key per tenant, idempotency) | Done | `food_management/pos_api.py`, `food_management/urls.py` | — |
| 2.1 Configurable days-to-expiry thresholds | Done | `CategoryConfig` model/admin plus `forms.py` and `services.py:get_category_thresholds` | Defaults remain 3/7/30 days when no category override exists |
| 2.2 Status labels (fresh/warning/critical/expired) | Done | `models.py` `RISK_CHOICES`, auto-assigned in `forms.py:clean` and `views.py:inventory_import` | — |
| 2.3 Scheduled alerts with deduplication | Done | `send_expiry_alerts` management command + `ExpiryAlert` model | Run daily via cron or Windows Task Scheduler; per-user delivery preferences supported; expired items are labelled `EXPIRED` |
| 3.1 Food taxonomy with perishability & storage | Done | Managed `CategoryConfig` records expose perishability and storage guidance in Django Admin | Defaults are seeded by migration |
| 3.2 Auto-suggest category | Done | `InventoryForm` category datalist is populated from active `CategoryConfig` records | Users may still enter a new category when needed |
| 4.1 Barcode/QR scanning (mobile-friendly) | Done | `static/js/barcode-scanner.js`, inventory form camera controls, tenant-scoped lookup endpoint, backend tests | Requires a supporting browser such as Chrome/Edge; physical camera hardware not tested. Barcode generation is not implemented or required by the PRD |
| 5.1 Multi-tenant inventory (RLS or equivalent) | Done | Inventory list/edit/delete and dashboard metrics are scoped by `request.user`; legacy rows have an explicit backfill command | Keep ownership checks on every object action |
| 5.2 Transaction history (sale/purchase/waste/donation) | Done | `models.py:Transaction`, `services.py:record_transaction`, migration 0002 | Inventory purchase/import, POS, and donation collection flows record transactions |
| 5.3 Surplus donation logs | Done | `SurplusListing`, `DonationRequest` models + views | — |

### Milestone 2: AI Waste Prediction Engine

| Requirement | Status | Evidence (file:line) | Gap |
|:---|:---|:---|:---|
| 1.1 Category-wise daily time series | Done | `ml-service/data/preprocess.py:prepare_category_series` and live export bridge | ML inference prefers exported Django `Transaction` rows and uses synthetic data only as fallback |
| 2.1 Prophet forecasting | Done | `ml-service/models/baseline_prophet.py`, artifacts in `models/saved/` | — |
| 2.2 LSTM forecasting | Done | `ml-service/models/lstm_model.py`, `.pt` artifacts | — |
| 2.3 Daily + weekly forecast | Done | `/predict/demand` returns `weekly_summary` alongside daily forecasts | — |
| 2.4 Naive baseline | Done | `train_and_eval.py:naive_seasonal_baseline` lag-7 | — |
| 2.5 MAE / RMSE / MAPE evaluation | Done | `train_and_eval.py:compute_metrics`, `models/metrics.json` | MAPE uses safe denominator `max(\|y\|, 1.0)` — documented |
| 2.6 High-risk item flagging | Done | `risk_scorer.py` returns `risk_level` Critical/High/Medium/Low + `action_recommendation` | — |
| 3.1 Waste risk score 0–100 | Done | `risk_scorer.py:calculate_waste_risk` combining D, S, F | — |
| 4.1 Smart reorder recommendation | Done (fixed) | `reorder/optimizer.py`; endpoint now calls trained model when `forecast_lead_demand` omitted | — |
| 5.1 On-demand scoring endpoints | Done | `/predict/demand`, `/risk/score`, `/reorder/recommend` with API-key auth | — |
| 5.2 Scheduled batch predictions | Done | `api/scheduler.py` daily 02:00 AM cron, now uses Prophet/LSTM models | — |

---

## 2. Extra Features Built (beyond requirements)

- **Gamification system** — points, streaks, levels, badges (`services.py`, `core/models.py`)
- **Leaderboard** — top users by points (`views.py:leaderboard`)
- **Admin user management** — suspend / delete users (`views.py:admin_user_toggle_status`, `admin_user_delete`)
- **Role-based access** — business / NGO / admin roles with `@role_required` decorator
- **Dashboard real-time metrics** — auto-refreshing `/dashboard/metrics/` JSON endpoint
- **Forecast comparison plots** — 6 PNG charts saved to `ml-service/reports/`
- **Docker packaging** — `ml-service/Dockerfile` + healthcheck for ML service
- **`ml_client.py`** — zero-dependency HTTP client with graceful offline fallback

---

## 3. Quality & Security Issues

### High Severity

| # | Issue | Evidence | Fix Applied |
|:---|:---|:---|:---|
| H1 | Hardcoded API key in source (`dev-secret-key-food-waste-2026`) | `ml-service/api/config.py:15` (original) | ✅ Fixed — commit `401901d`: no default, startup validator rejects weak keys |
| H2 | Same hardcoded key in Django `ml_client.py` | `food_management/ml_client.py:27` (original) | ✅ Fixed — commit `2677be9`: empty string default, reads from env |
| H3 | No tenant isolation on inventory — all users saw all records | `views.py:inventory_list` no filter | ✅ Fixed — commit `d794116`: `business` FK + per-user queryset filter |
| H4 | `/risk/batch/{business_id}` accepted any tenant ID with no check | `main.py:get_batch_risk` (original) | ✅ Fixed — commit `401901d`: `verify_tenant_access` + `ALLOWED_BUSINESS_IDS` |
| H5 | Scheduler batch used static heuristic, not trained models | `scheduler.py:94` | ✅ Fixed — commit `c459313`: `_get_forecast_demand()` with Prophet/LSTM |
| H6 | Django `SECRET_KEY` hardcoded in settings | `config/settings.py` | ✅ Fixed — startup fails when `SECRET_KEY` is absent |

### Medium Severity

| # | Issue | Evidence | Fix Applied |
|:---|:---|:---|:---|
| M1 | Reorder endpoint docstring claimed model query but used heuristic | `main.py:307` | ✅ Fixed — commit `8485524` |
| M2 | Prophet does not beat naive on every category | `MILESTONE2_REPORT.md §3` / `metrics.json` | ✅ Documented accurately — the report identifies bakery/produce exceptions and selects the best model per category |
| M3 | Scheduler tenant filter fell back to all-items on miss | `scheduler.py:55-58` | ✅ Fixed — commit `401901d` |
| M4 | No weekly-aggregated forecast output | `/predict/demand` response | ✅ Fixed — response includes `weekly_summary` |
| M5 | Live transaction export depends on the scheduled bridge script | `scripts/export_live_transactions.ps1`, `ml-service/api/config.py` | ✅ Fixed — ML inference prefers the atomically exported live file |
| M6 | Silent exception swallowing in batch cache read | `main.py:get_batch_risk` | ✅ Fixed — invalid cache reports are logged and recomputed |
| M7 | `business_id` field in `DemandPredictRequest` accepted but never used | `main.py` / `schemas.py` | ✅ Fixed — tenant is validated and LSTM history is filtered |

### Low Severity

| # | Issue | Evidence | Fix Applied |
|:---|:---|:---|:---|
| L1 | `/health` discloses model inventory without auth | `main.py:116` | By design (Docker healthcheck); acceptable |
| L2 | MAPE uses safe denominator `max(\|y\|, 1.0)` — not textbook formula | `train_and_eval.py:49-51` | Documented in report; acceptable |
| L3 | No POS REST API (M1 Req 1.3) | `food_management/pos_api.py` | ✅ Fixed |
| L4 | Barcode/QR generation is not provided | PRD future-enhancement language and current implementation | Not required — scanning and tenant-scoped lookup are implemented; generation is outside the documented requirement |
| L5 | No scheduled expiry alerts | Django | ✅ Fixed — `send_expiry_alerts` management command |
| L6 | Category thresholds hardcoded (3/7/30 days) | `forms.py`, `services.py`, alert command | ✅ Fixed — `CategoryConfig` overrides govern expiry risk and expiry alerts. There is no separate low-stock threshold |

---

## 4. Prioritized Plan (Milestone 2 by 1 Nov 2026)

### P0 — Must fix before submission

| Item | Why | Effort | Files |
|:---|:---|:---|:---|
| Fix Django `SECRET_KEY` to load from env (done) | Security — hardcoded in `settings.py` | 15 min | `config/settings.py`, `.env` |
| Wire `Transaction` rows on inventory CRUD (done) | M2 data pipeline needs real sales/waste history | 1 hr | `food_management/views.py`, `models.py` |
| Run `python manage.py migrate` | Apply all current migrations to DB | Done — migrations applied locally on 1 Nov 2026 | DB |
| Fix MILESTONE2_REPORT.md §3 Prophet claim | Submission accuracy | Done | `ml-service/MILESTONE2_REPORT.md` |

### P1 — Should add if time allows

| Item | Why | Effort | Files |
|:---|:---|:---|:---|
| POS REST API endpoint (done) | M1 Req 1.3 — completes Milestone 1 | 2 hrs | `food_management/pos_api.py`, `urls.py` |
| Weekly-horizon forecast output (done) | M2 Req 2.3 — adds aggregate weekly view | 1 hr | `ml-service/api/main.py`, `schemas.py` |
| Scheduled expiry alerts (management command) (done) | M1 Req 2.3 | 2 hrs | `management/commands/send_expiry_alerts.py` |
| Seed/demo data script (full flow) | Demo & evaluation | Done | `food_management/management/commands/seed_demo_data.py` |

### P2 — Milestone 3 / polish

| Item | Why | Effort | Files |
|:---|:---|:---|:---|
| Item-level LSTM forecasting | Better accuracy than category-level | 4 hrs | `ml-service/models/` |
| Automatic ML ingestion from Django transactions | Removes manual export dependency | Done via scheduled live export bridge | `scripts/export_live_transactions.ps1`, `ml-service/api/` |

---

## 5. Verified ML Artifacts

Tracked datasets, model artifacts, and charts are listed below. Model binaries,
metrics, and the live transaction export are used at runtime; source datasets
and charts are regenerable from the training pipeline.

| Artifact | Size | Status |
|:---|---:|:---|
| `ml-service/data/daily_transactions.csv` | 312,273 bytes | Regenerable synthetic/bootstrap fallback |
| `ml-service/data/inventory_snapshot.csv` | 15,126 bytes | Regenerable batch input; runtime fallback/demo data |
| `ml-service/models/saved/lstm_bakery.pt` | 31,557 bytes | Runtime |
| `ml-service/models/saved/lstm_beverages.pt` | 31,593 bytes | Runtime |
| `ml-service/models/saved/lstm_dairy.pt` | 31,545 bytes | Runtime |
| `ml-service/models/saved/lstm_meat.pt` | 31,533 bytes | Runtime |
| `ml-service/models/saved/lstm_packaged.pt` | 31,581 bytes | Runtime |
| `ml-service/models/saved/lstm_produce.pt` | 31,569 bytes | Runtime |
| `ml-service/models/saved/prophet_bakery.joblib` | 41,811 bytes | Runtime |
| `ml-service/models/saved/prophet_beverages.joblib` | 41,811 bytes | Runtime |
| `ml-service/models/saved/prophet_dairy.joblib` | 41,744 bytes | Runtime |
| `ml-service/models/saved/prophet_meat.joblib` | 41,811 bytes | Runtime |
| `ml-service/models/saved/prophet_packaged.joblib` | 41,811 bytes | Runtime |
| `ml-service/models/saved/prophet_produce.joblib` | 41,811 bytes | Runtime |
| `ml-service/models/metrics.json` | 2,434 bytes | Runtime model selection/metrics |
| `ml-service/reports/batch_predictions/batch_biz_001_latest.json` | 12,606 bytes | Regenerable batch output |
| `ml-service/reports/forecast_comparison_bakery.png` | 171,651 bytes | Regenerable report chart |
| `ml-service/reports/forecast_comparison_beverages.png` | 174,174 bytes | Regenerable report chart |
| `ml-service/reports/forecast_comparison_dairy.png` | 157,001 bytes | Regenerable report chart |
| `ml-service/reports/forecast_comparison_meat.png` | 164,621 bytes | Regenerable report chart |
| `ml-service/reports/forecast_comparison_packaged.png` | 150,084 bytes | Regenerable report chart |
| `ml-service/reports/forecast_comparison_produce.png` | 159,595 bytes | Regenerable report chart |

Training prefers `ml-service/data/live_transactions.csv` when it exists,
logging that source explicitly, and falls back to
`daily_transactions.csv`. Runtime inference and scheduled batch scoring use
the same live-first policy.

---

## 6. Demo Flow

Run this sequence for the strongest demo:

```
1. Register as "business" user
2. Add inventory item (bakery, expiry 2 days) → risk_level auto-sets to "critical"
3. View dashboard → expiry alert appears
4. POST /risk/score with that item's data → score ~90, Critical
5. POST /reorder/recommend → order quantity with reasoning
6. Import CSV with 10 items → row-wise error report shown
7. GET /risk/batch/biz_001 → full tenant batch report
8. GET /models/metrics → MAE/RMSE/MAPE comparison table
```

---

## 7. Commits Applied During Audit

| Commit | Description |
|:---|:---|
| `401901d` | Remove hardcoded API key; add tenant isolation on batch endpoint |
| `c459313` | Scheduler batch uses trained Prophet/LSTM models |
| `2677be9` | Remove hardcoded ML key from Django ml_client.py |
| `adf1374` | Django UX: dashboard metrics endpoint, CSV validation |
| `8485524` | Reorder endpoint uses trained model for auto-forecast |
| `d794116` | Add business FK + Transaction model (migration 0002) |

---

## 8. End-to-End Expiry Date & Expired Product Verification

| Case | Result | Verified Implementation & Evidence | Fix Applied |
|:---|:---:|:---|:---|
| **a. Expiry = today, tomorrow, yesterday** | **PASS** | days_until_expiry uses 	imezone.localdate() ( , 1, -1). Boundary logic tested in ExpiryRegressionAuditTests. | Fixed off-by-one and negative day handling across models, forms, and alerts. |
| **b. Timezone (Asia/Kolkata)** | **PASS** | TIME_ZONE = 'Asia/Kolkata' and USE_TZ = True. Replaced all date.today() with 	imezone.localdate() in models, services, commands, and views. | Eliminates midnight drift and inconsistencies with server UTC. |
| **c. days_left == category threshold** | **PASS** | At boundary days_left <= threshold, item triggers the alert and sets the corresponding risk category. | Verified exact boundary comparison in send_expiry_alerts and InventoryForm. |
| **d. Expired items handling** | **PASS** | Expired stock (days_left < 0) is explicitly flagged with status label EXPIRED, risk score 100 / Critical, and counted in a dedicated dashboard alert card. | Fixed calculate_waste_risk_score() to return 100 for expired items, updated UI banner. |
| **e. Expired stock in POS sales** | **PASS** | When a sale is posted for a category that contains expired batches, the response includes an explicit safety warning: 'Warning: Expired stock exists for category...'. | Added physical verification check and safety warning in pos_transaction_ingest. |
| **f. Reorder & forecasting stock count** | **PASS** | Active stock filters require expiry_date >= today, ensuring expired stock is excluded from usable inventory. | Verified in demand forecasting and optimizer inventory inputs. |
| **g. CSV import validation** | **PASS** | Blank/invalid date inputs or past expiry dates (expiry_date < today) are cleanly rejected with a row-specific error message without crashing. | Enforced date parsing and past-date rejection in inventory_import. |
| **h. Alert deduplication** | **PASS** | ExpiryAlert uniquely logs (item, threshold_days, alert_date). Transition from warning to expired registers threshold  , preventing duplicate notices within the same calendar day. | Added threshold 0 tracking for expired items in send_expiry_alerts. |
| **i. Multiple batches of same product** | **PASS** | Fresh and expired batches remain isolated per barcode/batch. Active inventory tallies only include non-expired batches. | Verified multi-batch aggregate filtering in regression test suite. |
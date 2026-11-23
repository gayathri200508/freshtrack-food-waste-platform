# Deploying FreshTrack

## Option A: Render (free tier, Docker)
1. Push this folder to GitHub (without `venv/` and `*.db`, see `.gitignore`).
2. On render.com: New > Blueprint > pick the repo. Render reads `render.yaml`.
3. Wait for the build (Prophet takes several minutes). Open the URL it gives you.
4. Log in with a demo account, then upload `sample/world_food_inventory.csv`.

Notes:
- Free-tier disk is ephemeral. The SQLite file in `/tmp` resets on restart and on redeploy. For a permanent demo, move to PostgreSQL (set `DATABASE_URL`).
- Demo accounts (`manager@worldmart.com`, `partner@feedingindia.org`) are created on startup.

## Option B: Local demo on a laptop
`uvicorn app.main:app --host 0.0.0.0 --port 8000` then open `http://<laptop-ip>:8000` from a phone on the same Wi-Fi.
Camera scanning needs HTTPS or localhost, so use manual barcode entry on phones.

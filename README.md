# 0xchou00 - Lightweight Security Detection Tool

`0xchou00` is a local-first security detection tool with a FastAPI backend and a React SOC dashboard.
The backend ingests SSH, HTTP, and firewall logs, normalizes them into one event model, enriches source IPs, runs bounded detections, correlates related alerts, and records the stored state in a verifiable integrity chain.
The dashboard reads the same backend through `GET /health`, `GET /alerts`, and `GET /logs`.

## Repository layout

- `backend/` FastAPI API, normalization, detection, correlation, storage, and integrity logic
- `backend/rules/` YAML detection rules, correlation rules, and static blacklist data
- `agent/` local tailing agent for auth, nginx, and firewall logs
- `dashboard/` React SOC dashboard
- `docs/` technical notes
- `scripts/` systemd units and Linux install script

## Backend

Features:

- FastAPI ingest and query API
- normalization for SSH, nginx-style HTTP, and firewall-style network logs
- SSH brute-force detection
- suspicious web behavior detection
- port-scan detection
- YAML rule engine with regex and aggregation
- alert correlation
- IP enrichment with local GeoIP, static blacklist data, and optional AbuseIPDB lookups
- SQLite storage
- contract-style integrity verification

Run locally:

```bash
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cd backend
uvicorn main:app --reload
```

## Full Linux Setup & Testing Guide (Debian / Ubuntu / Kali)

This workflow is written for Kali Linux and is valid for Debian/Ubuntu with the same commands.

### 1. Cloning & Initial Setup

```bash
cd /home/kali
git clone https://github.com/0xchou00/0xchou00-Detection-.git
cd /home/kali/0xchou00-Detection-
git checkout fix/linux-compatibility
```

Validate runtime versions before installing:

```bash
python3 --version
pip3 --version
node --version
npm --version
```

Recommended minimums:

- Python `3.10+`
- Node.js `18+`
- npm `9+`

### 2. System Dependencies

Install base packages:

```bash
sudo apt-get update
sudo apt-get install -y \
  python3 python3-venv python3-pip \
  nodejs npm \
  curl net-tools git ca-certificates jq
```

### 3. Environment Setup

Create Python virtual environment and install backend dependencies:

```bash
cd /home/kali/0xchou00-Detection-
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Install dashboard dependencies:

```bash
cd /home/kali/0xchou00-Detection-/dashboard
npm install
```

Critical dependencies used by runtime:

- `fastapi`: API service exposing ingest/query routes
- `uvicorn[standard]`: ASGI server for backend process
- `httpx`: outbound HTTP client for enrichment and agent forwarding
- `PyYAML`: rule/config parsing
- `geoip2`: optional GeoIP enrichment

### 4. Configuration

Create `.env` in project root:

```bash
cd /home/kali/0xchou00-Detection-
cat > .env <<'EOF'
SIEM_DB_PATH=/home/kali/0xchou00-Detection-/backend/data/0xchou00-tool.db
SIEM_ADMIN_API_KEY=siem-admin-dev-key
SIEM_ANALYST_API_KEY=siem-analyst-dev-key
SIEM_VIEWER_API_KEY=siem-viewer-dev-key
SIEM_ALLOWED_ORIGINS=http://localhost:5173,http://127.0.0.1:5173,http://localhost:4173,http://127.0.0.1:4173
SIEM_GEOIP_DB_PATH=/home/kali/0xchou00-Detection-/backend/data/GeoLite2-City.mmdb
ABUSEIPDB_API_KEY=
EOF
```

Port and endpoint mapping:

- Backend API: `http://127.0.0.1:8000`
- Dashboard UI: `http://127.0.0.1:5173`
- Dashboard -> Backend base URL: `VITE_TOOL_API_BASE` (defaults to `http://localhost:8000`)
- Dashboard auth header: `X-API-Key` using viewer key

### 5. Linking Tool with Dashboard

Connection model:

1. Logs are submitted to `POST /ingest` with analyst API key.
2. Backend normalizes, enriches, detects, correlates, and stores in SQLite.
3. Dashboard polls `GET /health`, `GET /alerts`, and `GET /logs` with viewer API key.

API endpoints used by dashboard:

- `GET /health`
- `GET /alerts`
- `GET /logs`

Realistic ingest example:

```bash
curl -sS -X POST http://127.0.0.1:8000/ingest \
  -H "Content-Type: application/json" \
  -H "X-API-Key: siem-analyst-dev-key" \
  -d '{
    "source_type": "firewall",
    "lines": [
      "Apr 21 12:00:01 sensor kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00 SRC=203.0.113.55 DST=192.168.1.20 LEN=60 TOS=0x00 PREC=0x00 TTL=51 ID=20001 DF PROTO=TCP SPT=45671 DPT=22 WINDOW=64240 RES=0x00 SYN URGP=0",
      "Apr 21 12:00:03 sensor kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00 SRC=203.0.113.55 DST=192.168.1.20 LEN=60 TOS=0x00 PREC=0x00 TTL=51 ID=20002 DF PROTO=TCP SPT=45672 DPT=80 WINDOW=64240 RES=0x00 SYN URGP=0"
    ]
  }'
```

Expected successful response example:

```json
{
  "processed": 2,
  "alerts_generated": 1,
  "ingested_at": "2026-04-21T12:00:03.000000+00:00"
}
```

### 6. Running the Project

Terminal 1 - start backend:

```bash
cd /home/kali/0xchou00-Detection-
source .venv/bin/activate
cd backend
uvicorn main:app --host 0.0.0.0 --port 8000
```

Terminal 2 - start dashboard:

```bash
cd /home/kali/0xchou00-Detection-/dashboard
npm run dev -- --host 0.0.0.0 --port 5173
```

Expected output:

- Backend prints `Uvicorn running on http://0.0.0.0:8000`
- Dashboard prints local URL including `http://localhost:5173`

### 7. Testing Workflow

Run API validation:

```bash
curl -sS http://127.0.0.1:8000/health -H "X-API-Key: siem-viewer-dev-key" | jq
```

Simulate ingest:

```bash
curl -sS -X POST http://127.0.0.1:8000/ingest \
  -H "Content-Type: application/json" \
  -H "X-API-Key: siem-analyst-dev-key" \
  -d '{"source_type":"ssh","lines":["Apr 21 12:05:00 kali sshd[14500]: Failed password for invalid user admin from 198.51.100.42 port 54221 ssh2"]}' | jq
```

Validate stored logs and alerts:

```bash
curl -sS "http://127.0.0.1:8000/logs?limit=5" -H "X-API-Key: siem-viewer-dev-key" | jq
curl -sS "http://127.0.0.1:8000/alerts?limit=5" -H "X-API-Key: siem-viewer-dev-key" | jq
```

Dashboard verification:

1. Open `http://127.0.0.1:5173`.
2. Set API base URL to `http://127.0.0.1:8000` if needed.
3. Use viewer key `siem-viewer-dev-key`.
4. Confirm health card is green, logs table updates, and new alerts appear.

### 8. Deployment Mode (systemd)

Use bundled installer:

```bash
cd /home/kali/0xchou00-Detection-
chmod +x setup.sh run.sh scripts/install.sh
./scripts/install.sh
sudo systemctl start 0xchou00.service
sudo systemctl start 0xchou00-agent.service
sudo systemctl status 0xchou00.service --no-pager
sudo systemctl status 0xchou00-agent.service --no-pager
```

## Common Issues on Linux & Fixes

### Permission denied on scripts

```bash
chmod +x /home/kali/0xchou00-Detection-/setup.sh
chmod +x /home/kali/0xchou00-Detection-/run.sh
chmod +x /home/kali/0xchou00-Detection-/scripts/install.sh
```

### Port already in use

```bash
sudo netstat -tulpn | grep :8000
sudo netstat -tulpn | grep :5173
sudo kill -9 <PID>
```

Or run dashboard on another port:

```bash
cd /home/kali/0xchou00-Detection-/dashboard
npm run dev -- --host 0.0.0.0 --port 5174
```

### Missing dependencies

```bash
sudo apt-get update
sudo apt-get install -y python3 python3-venv python3-pip nodejs npm curl net-tools jq
cd /home/kali/0xchou00-Detection-
source .venv/bin/activate
pip install -r requirements.txt
cd dashboard && npm install
```

### API connection errors from dashboard

Check backend health and key:

```bash
curl -sS http://127.0.0.1:8000/health -H "X-API-Key: siem-viewer-dev-key" | jq
```

If CORS fails, verify `SIEM_ALLOWED_ORIGINS` in `/home/kali/0xchou00-Detection-/.env` includes the dashboard origin.

API:

- `GET /health`
- `POST /ingest`
- `GET /logs`
- `GET /alerts`
- `GET /integrity/verify`

Viewer key:

```text
siem-viewer-dev-key
```

## Dashboard

Run locally:

```bash
cd dashboard
npm install
npm run dev
```

Default frontend target:

- `http://localhost:8000`

The dashboard polls:

- `GET /health`
- `GET /alerts`
- `GET /logs`

## Example ingest

```bash
curl -X POST http://127.0.0.1:8000/ingest \
  -H "Content-Type: application/json" \
  -H "X-API-Key: siem-analyst-dev-key" \
  -d '{
    "source_type": "firewall",
    "lines": [
      "Apr 19 10:00:00 sensor kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00 SRC=198.51.100.50 DST=192.168.1.10 LEN=60 TOS=0x00 PREC=0x00 TTL=51 ID=54321 DF PROTO=TCP SPT=41233 DPT=22 WINDOW=64240 RES=0x00 SYN URGP=0",
      "Apr 19 10:00:02 sensor kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00 SRC=198.51.100.50 DST=192.168.1.10 LEN=60 TOS=0x00 PREC=0x00 TTL=51 ID=54322 DF PROTO=TCP SPT=41234 DPT=80 WINDOW=64240 RES=0x00 SYN URGP=0"
    ]
  }'
```

## Technical notes

See `docs/TECHNICAL.md` for:

- architecture
- detection logic
- enrichment logic
- correlation
- integrity model
- design decisions
- limitations

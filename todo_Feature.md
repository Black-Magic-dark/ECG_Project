# ECG Project — Full Feature & Task Checklist
> Single source of truth for all work across all phases.
> Each person owns the tasks tagged next to them.
>
> **Architecture (v2)**: ESP32 is a dumb USB ADC bridge → RPi 4B runs full ML pipeline locally (Edge AI) → Cloud API stores summaries + serves React UI.

---

## 🔧 Phase 0 — Breakage Fixes (COMPLETED — Satyarth)
- [x] Fix `conftest.py` broken `sys.path` (was pointing at deleted `src/`, now `backend/src/`)
- [x] Fix `tests/test_signal_processing.py` broken import path → `backend/src/`
- [x] Fix `tests/test_feature_extraction.py` broken import path → `backend/src/`
- [x] Remove `eventlet` from `backend/requirements.txt` (conflicts with `async_mode="threading"`)
- [x] Remove training-only deps (`wfdb`, `matplotlib`) from `backend/requirements.txt`
- [x] Create `ml_training/requirements.txt` for training environment only
- [x] Add `pymongo` and `python-dotenv` to `backend/requirements.txt`
- [x] Create `.env.example` with all required env variable templates
- [x] Create `.gitignore` (was completely missing — credentials were at risk)
- [x] Delete stale `__pycache__/` at project root
- [x] Delete `frontend/legacy_dashboard/` (dead Streamlit code)
- [x] Move `assets/` → `ml_training/assets/`
- [x] Create `backend/logs/.gitkeep`
- [x] Commit: `fix: Phase 0 - fix broken paths, add .gitignore, .env.example, cleanup`

---

## 🏗️ Phase 1 — Architecture Restructure (COMPLETED)
- [x] Create `backend/`, `frontend/`, `ml_training/` directories
- [x] Move `src/` → `backend/src/`
- [x] Move `server.py` → `backend/server.py`
- [x] Move `requirements.txt` → `backend/requirements.txt`
- [x] Move `model/ecg_rf_model_v1.pkl` + `scaler_v1.pkl` + `model_metadata.json` → `backend/model/`
- [x] Move training scripts → `ml_training/model/`
- [x] Scaffold React app with Vite in `frontend/react_app/`
- [x] Commit: `refactor: Phase 1 architecture restructure`

---

## 🗄️ Phase 2 — Database Setup (Sohan)
> **Sign up**: [MongoDB Atlas](https://www.mongodb.com/cloud/atlas/register)
>
> **Note (v2 Architecture)**: Raw ECG waveform data is NO LONGER sent to the cloud. Only 5-second ML summaries and alerts are stored. This massively reduces DB load.

- [ ] Create MongoDB Atlas account (free M0 tier)
- [ ] Create cluster, create database user (`ecg_admin`), allow `0.0.0.0/0` network access
- [ ] Copy the `MONGO_URI` connection string → share with Satyarth securely (NOT via git — use WhatsApp/DM)
- [ ] Create `users` collection — schema: `{username, email, password_hash, role, created_at}`
- [ ] Create `patients` collection — schema: `{user_id, name, dob, assigned_room, assigned_doctors[], assigned_nurses[]}`
- [ ] Create `devices` collection — schema: `{device_id (RPi hostname or MAC), room_number, status}`
- [ ] Create `ecg_summaries` collection — Bucket Pattern, one doc per 5-second inference window:
  - Fields: `patient_id, start_time, end_time, heart_rate, rr_mean, rr_std, sdnn, rmssd, beat_variance, sqi, prediction, probability, consecutive_count`
- [ ] Create `alerts` collection — schema: `{patient_id, device_id, severity, timestamp, consecutive_count, probability, acknowledged, acknowledged_by}`
  - **No raw ECG collection needed** — raw data stays on RPi local storage only
- [ ] Verify connection from a local Python script using `MONGO_URI` before handing off
- [ ] Commit: `git commit -m "docs: MongoDB schemas finalised for v2 RPi architecture"`

---

## ⚙️ Phase 3 — RPi 4B Edge Node Setup (Ajit)
> **Architecture Decision (v2)**: ESP32 is kept as a **dumb USB ADC bridge only** — no WiFi, no HTTP, no cloud communication.
> All intelligence (ML inference, MongoDB upload, alert dispatch) runs on the **Raspberry Pi 4B**.
> The existing `ECGInferenceEngine` in `realtime_inference.py` runs on the RPi almost unchanged.

### 3a — ESP32 Firmware (Simplify — Remove All WiFi Code)
- [ ] Open `firmware/ECG_Firmware/ECG_Firmware.ino` in Arduino IDE
- [ ] **Remove** (or comment out): any `WiFi.h`, `HTTPClient.h`, `ArduinoJson.h` includes (none exist yet — keep it that way)
- [ ] Keep the existing code exactly as-is — it already does exactly what we need:
  - 250Hz hardware timer ISR ✅
  - ADC read on GPIO34 ✅
  - 3-sample moving average ✅
  - Lead-off detection on GPIO32/GPIO33 ✅
  - Serial output format: `<millis>,<ecg_value>,<lead_off>\n` ✅
  - Reads `BUZZ_ON` / `BUZZ_OFF` commands from Serial ✅
- [ ] Flash firmware to ESP32 — verify Serial Monitor shows CSV output at 115200 baud
- [ ] Connect ESP32 to RPi 4B via USB cable — verify it appears as `/dev/ttyUSB0` or `/dev/ttyACM0` on RPi

### 3b — Raspberry Pi 4B OS & Environment Setup
- [ ] Flash **Raspberry Pi OS 64-bit** (Bookworm, headless) onto SD card using Raspberry Pi Imager
- [ ] Enable SSH in Imager settings (set hostname, username, password, Wi-Fi credentials)
- [ ] Boot RPi, SSH in: `ssh <username>@<hostname>.local`
- [ ] Update system: `sudo apt update && sudo apt upgrade -y`
- [ ] Install Python 3.11+ and pip: `sudo apt install python3 python3-pip python3-venv -y`
- [ ] Create project virtual environment: `python3 -m venv ~/ecg_env && source ~/ecg_env/bin/activate`
- [ ] Copy `backend/` folder to RPi (use `scp -r backend/ <user>@<hostname>.local:~/ecg_edge/`)
- [ ] Install backend dependencies on RPi: `pip install -r requirements.txt`
- [ ] Verify serial port permissions: `sudo usermod -aG dialout $USER` then reboot

### 3c — Configure RPi as Edge Inference Node
- [ ] Create `.env` file on RPi at `~/ecg_edge/` with:
  - `MONGO_URI=<from Sohan>`
  - `FLASK_SECRET_KEY=<random string>`
  - `JWT_SECRET=<random string>`
  - `SERIAL_PORT=/dev/ttyUSB0` (or `/dev/ttyACM0` — check with `ls /dev/tty*`)
  - `EDGE_DEVICE_ID=<RPi hostname or a unique string set by admin>`
- [ ] Verify model loads correctly: `python3 -c "import joblib; joblib.load('model/ecg_rf_model_v1.pkl'); print('Model OK')"`
- [ ] Run `server.py` on RPi: `python3 server.py` — confirm it starts without errors
- [ ] Open browser on laptop connected to same Wi-Fi → `http://<rpi-hostname>.local:5000` — confirm dashboard loads
- [ ] Start inference engine in Demo Mode → confirm waveform appears
- [ ] Start inference engine with Serial port → confirm live ECG from ESP32 appears

### 3d — Auto-start on RPi Boot (so it runs without SSH)
- [ ] Create a systemd service file `/etc/systemd/system/ecg_edge.service`:
  ```
  [Unit]
  Description=ECG Edge Inference Service
  After=network.target

  [Service]
  User=<your_username>
  WorkingDirectory=/home/<your_username>/ecg_edge
  ExecStart=/home/<your_username>/ecg_env/bin/python3 server.py
  Restart=always
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```
- [ ] Enable and start: `sudo systemctl enable ecg_edge && sudo systemctl start ecg_edge`
- [ ] Verify service is running: `sudo systemctl status ecg_edge`
- [ ] Reboot RPi and confirm service auto-starts and ECG stream resumes

### 3e — Buzzer on RPi (Replace ESP32 Serial Command)
> The buzzer is currently controlled by the RPi sending `BUZZ_ON` over Serial to the ESP32.
> This still works in Option B — the ESP32 reads `BUZZ_ON` from Serial and fires the buzzer.
> No change needed — the `_send_serial_command("BUZZ_ON")` path in `realtime_inference.py` already handles this.
- [ ] Verify buzzer fires correctly when `BUZZ_ON` is sent from RPi to ESP32 via Serial
- [ ] Commit firmware: `git commit -m "feat(firmware): verified dumb ADC bridge mode, no WiFi needed"`
- [ ] Commit RPi setup docs: `git commit -m "feat(edge): RPi 4B edge node setup and systemd service"`

---

## 🔌 Phase 4 — Backend Split: RPi (Edge) + Cloud (API) (Satyarth)
> **v2 Architecture**: `server.py` is split into two roles:
> - **RPi (`edge_server.py`)**: Runs the ECGInferenceEngine, reads ESP32 serial, does ML, uploads summaries to MongoDB, serves local WebSocket.
> - **Cloud (`cloud_api.py` on Render)**: Thin REST API — stores data from RPi, serves React frontend.

### 4a — Environment & Config (Both Nodes)
- [ ] Add `from dotenv import load_dotenv` + `load_dotenv()` to `server.py` at startup
- [ ] Replace hardcoded `SECRET_KEY = "ecg_secret_2026"` → `os.getenv("FLASK_SECRET_KEY")`
- [ ] Create `backend/database.py` — MongoClient singleton with collection accessors
- [ ] Add `EDGE_DEVICE_ID` env var — unique identifier for this RPi unit (maps to a room)
- [ ] Add `PyJWT` to `backend/requirements.txt`

### 4b — RPi Edge Server (`server.py` — runs on RPi locally)
- [ ] On startup: look up `EDGE_DEVICE_ID` in MongoDB `devices` collection → resolve `patient_id`
- [ ] If device not registered in DB → log warning, run in local-only mode (don't crash)
- [ ] After each 5-second inference window (`_run_inference`): replace CSV log with MongoDB insert into `ecg_summaries`
- [ ] On ABNORMAL alert (3 consecutive windows): insert into `alerts` collection with debounce (no duplicate if one exists in last 5 minutes)
- [ ] Keep existing Socket.IO `push_data_loop` — this serves the real-time waveform to any local browser on same network
- [ ] Add `patient_id` context to all Socket.IO `update` events (so cloud-connected browsers know which patient)
- [ ] Test locally: run `server.py` on RPi, open browser on laptop on same WiFi → confirm live ECG and predictions

### 4c — Cloud API (`cloud_api.py` — NEW file, runs on Render)
> This is a **new, separate, lightweight Flask app** — NOT the same as `server.py`.
> It has no ML model, no serial port, no inference engine. It only reads/writes MongoDB.
- [ ] Create `backend/cloud_api.py` — new Flask app (no SocketIO inference, just REST)
- [ ] `POST /api/auth/login` — validate user from `users` collection, return JWT
- [ ] `POST /api/ingest/summary` — **called by RPi** — receives 5-second summary, saves to `ecg_summaries`
  - Auth: `X-Edge-Key` header (shared secret between RPi and Render, set in `.env`)
- [ ] `POST /api/ingest/alert` — **called by RPi** — saves alert to `alerts` collection
  - Auth: same `X-Edge-Key`
- [ ] `GET /api/doctor/patients` — JWT protected (doctor role) — list of assigned patients
- [ ] `GET /api/patients/<id>/ecg-history` — JWT protected — paginated `ecg_summaries` from MongoDB
- [ ] `GET /api/alerts?patient_id=<id>` — JWT protected — unacknowledged alerts for doctor/nurse
- [ ] `POST /api/alerts/<id>/acknowledge` — JWT protected (doctor/nurse) — mark alert as seen
- [ ] `POST /api/admin/users` — JWT protected (admin) — create new user
- [ ] `POST /api/admin/assign-device` — map `device_id` (RPi identifier) to `room_number`
- [ ] `POST /api/admin/assign-patient` — map `patient_id` to `room_number`
- [ ] `POST /api/admin/assign-doctor` — link doctor/nurse ObjectId to patient
- [ ] `GET /api/patients/me` — JWT protected (patient role) — own records

### 4d — RPi → Cloud Communication
- [ ] Add background thread in `server.py` (RPi): after each inference window, HTTP POST summary to `https://<render-url>/api/ingest/summary`
- [ ] On alert: HTTP POST to `https://<render-url>/api/ingest/alert`
- [ ] Add retry logic: if Render is sleeping (cold start), retry after 5 seconds, up to 3 times
- [ ] If all retries fail: cache summary locally to a queue file, flush when connection restored

### 4e — Cloud Deployment Prep
- [ ] Create `backend/Procfile`: `web: python cloud_api.py`
- [ ] Create `backend/cloud_requirements.txt` — only what `cloud_api.py` needs (flask, pymongo, PyJWT, python-dotenv — no scipy, no joblib, no pyserial)
- [ ] Test `cloud_api.py` locally with `python cloud_api.py`
- [ ] Commit: `git commit -m "feat(backend): split into RPi edge server and Render cloud API"`

---

## 🎨 Phase 5 — React Frontend (Deepika + Rupam)

### 5a — Foundation & Routing (Rupam)
- [ ] Install dependencies: `npm install react-router-dom axios socket.io-client recharts`
- [ ] Set up `react-router-dom` with routes: `/login`, `/admin`, `/doctor`, `/patient`
- [ ] Create `AuthContext` (stores JWT token, user role, expiry) using React Context API
- [ ] Create `ProtectedRoute` component — redirects to `/login` if not authenticated
- [ ] Create `src/api/` folder with Axios instance (base URL = `VITE_API_BASE_URL`, auto-attaches JWT)

### 5b — Login Page (Deepika)
- [ ] Build `LoginPage.jsx` — email + password form, POST to `/api/auth/login`, store JWT in context
- [ ] Show role-appropriate redirect after login (admin → `/admin`, doctor → `/doctor`, patient → `/patient`)

### 5c — Admin Dashboard (Rupam)
- [ ] Build `AdminPage.jsx` with three panels:
  - [ ] **Users panel**: list users, create new user with role selector (admin/doctor/nurse/patient)
  - [ ] **Device panel**: form to register RPi `device_id` and map to room number
  - [ ] **Patient panel**: assign patient to room, assign doctor/nurse to patient

### 5d — Doctor / Nurse Dashboard (Deepika)
- [ ] Build `DoctorPage.jsx`:
  - [ ] Left sidebar: list assigned patients, fetched from `/api/doctor/patients`
  - [ ] On patient select: fetch recent ECG history from `/api/patients/<id>/ecg-history`
  - [ ] Display BPM trend chart for last 24 hours using Recharts `LineChart`
  - [ ] Display last prediction badge (Normal / ABNORMAL) + SQI indicator
  - [ ] **Alert panel**: polls `/api/alerts?patient_id=<id>` every 30 seconds
  - [ ] Alert banner: plays browser audio ping + red banner when unacknowledged ABNORMAL alert exists
  - [ ] Acknowledge button: calls `POST /api/alerts/<id>/acknowledge`

### 5e — Patient Dashboard (Deepika)
- [ ] Build `PatientPage.jsx`:
  - [ ] Fetch personal info and ECG history from `/api/patients/me`
  - [ ] Display BPM trend for last 24 hours using Recharts
  - [ ] Display last ML prediction and timestamp

### 5f — Deploy Frontend (Rupam)
- [ ] Push `frontend/react_app/` to its own GitHub repo
- [ ] Connect to [Vercel](https://vercel.com/) → New Project → set root to `/`
- [ ] Set env variable in Vercel: `VITE_API_BASE_URL=https://<render-cloud-api-url>`
- [ ] Verify login, admin panel, doctor dashboard work on deployed Vercel URL
- [ ] Commit: `git commit -m "feat(frontend): full React UI with RBAC, alert polling, ECG history"`

---

## 🚀 Phase 6 — Deployment & Integration Testing (Rupam + All)

### 6a — Deploy Cloud API to Render
- [ ] Push `backend/cloud_api.py` + `backend/cloud_requirements.txt` + `backend/Procfile` to its own GitHub repo
- [ ] Connect to [Render](https://render.com/) → New Web Service
- [ ] Set start command: `python cloud_api.py`
- [ ] Set all env variables: `MONGO_URI`, `JWT_SECRET`, `FLASK_SECRET_KEY`, `EDGE_KEY`
- [ ] Set up free cron job at [cron-job.org](https://cron-job.org) to ping `/api/status` every 10 min (keep Render awake)

### 6b — Configure RPi for Production
- [ ] Update RPi `.env`: set `CLOUD_API_URL=https://<render-url>`, `EDGE_KEY=<shared secret>`
- [ ] Restart systemd service: `sudo systemctl restart ecg_edge`

### 6c — End-to-End Test Checklist
- [ ] ESP32 (USB) → RPi serial → ECG waveform appears in `server.py` logs ✅
- [ ] RPi ML inference → prediction logged + summary POSTed to Render `/api/ingest/summary` ✅
- [ ] Summary appears in MongoDB Atlas `ecg_summaries` collection ✅
- [ ] Doctor on Vercel → logs in → sees patient in sidebar → ECG history chart shows real data ✅
- [ ] ABNORMAL detected on RPi → alert POSTed to Render → doctor sees alert banner in browser ✅
- [ ] Buzzer fires on ESP32 (RPi sends `BUZZ_ON` over Serial) ✅
- [ ] Admin creates a new user → assigns RPi device to room → assigns patient → doctor logs in and sees patient ✅
- [ ] Commit: `git commit -m "deploy: full system integration verified"`

---

## 📚 Phase 7 — Documentation & Knowledge Base (All Members)
- [ ] Update `knowledge_base.md` with final API endpoint list, env vars, and v2 data flow diagram
- [ ] Add architecture diagram to `README.md`
- [ ] Write `backend/README.md` — how to run `cloud_api.py` locally and on Render
- [ ] Write `backend/EDGE_README.md` — how to set up RPi, run `server.py`, systemd service
- [ ] Write `frontend/react_app/README.md` — how to run React locally and deploy to Vercel
- [ ] Write `firmware/ECG_Firmware/README.md` — how to flash ESP32, verify Serial output
- [ ] Commit: `git commit -m "docs: complete project documentation for v2 RPi architecture"`

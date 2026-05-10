# ECG Project Features Todo List

## Phase 1: Architecture Restructuring
- [ ] Move ML models and training code to `ml_training/` directory.
- [ ] Move Streamlit/Web UI code to `frontend/` directory (or rewrite using React).
- [ ] Move Arduino/C++ code to `firmware/` directory.
- [ ] Create `backend/` directory for FastAPI logic.

## Phase 2: Database Setup (Sohan)
- [ ] Create MongoDB Atlas account.
- [ ] Set up `Users` collection (roles: Admin, Doctor, Nurse, Patient).
- [ ] Set up `Patients` collection.
- [ ] Set up `Devices` collection (map ESP32 MAC to Room Number).
- [ ] Set up `ECG_Data` collection (Time series data).
- [ ] Set up `Alerts` collection.

## Phase 3: Hardware Integration (Ajit)
- [ ] Connect ESP32 to AD8232 sensor.
- [ ] Read analog data.
- [ ] Connect to Wi-Fi.
- [ ] Send HTTP POST request with `device_id` and data payload.

## Phase 4: Backend API & ML (Satyarth)
- [ ] Set up FastAPI server.
- [ ] Connect FastAPI to MongoDB Atlas.
- [ ] Create endpoint `/api/data` to receive ESP32 POST request.
- [ ] Fetch Patient ID from DB based on `device_id` (Room mapping).
- [ ] Buffer incoming data.
- [ ] Load `ecg_rf_model_v1.pkl` and predict on buffer.
- [ ] Save results and any Alerts to MongoDB.

## Phase 5: Frontend Development (Deepika & Rupam)
- [ ] Scaffold React (Next.js) or HTML/JS project.
- [ ] Implement JWT/Cookie-based Login.
- [ ] Build **Admin Dashboard**:
  - Add/Remove Users.
  - Assign ESP32 Device (MAC Address) to Room Number.
  - Assign Patient to Room Number.
  - Assign Doctor/Nurse to Patient.
- [ ] Build **Doctor Dashboard**:
  - View assigned patients.
  - Real-time ECG graph for selected patient.
  - Receive Alerts (WebSocket or Polling).
- [ ] Build **Patient Dashboard**:
  - View personal historical data.

## Phase 6: Deployment (Rupam)
- [ ] Deploy Backend API on Render.
- [ ] Deploy Frontend on Vercel.
- [ ] Configure Environment Variables (CORS, MongoDB URI, Secret Keys).
- [ ] End-to-end test (ESP32 -> Render -> MongoDB -> Vercel UI).

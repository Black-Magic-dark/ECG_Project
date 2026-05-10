# ECG Project Knowledge Base

This document serves as the single source of truth for the project's technical details, decisions, and references for the LLM and the team.

## Architecture Decisions

1. **Monolith to Microservices**:
   - The original architecture housed the ML model, Streamlit UI, and server all in one monolithic codebase.
   - We are splitting this into **Backend (FastAPI)**, **Frontend (React/Vercel)**, and **Firmware (ESP32/C++)**.
   - **Reason**: Separation of concerns, easier independent deployments on free tiers, and better Role-Based Access Control (RBAC) capabilities.

2. **Hosting/Deployment Stack (Free Tier Strategy)**:
   - **Database**: MongoDB Atlas. (Reason: Free, document-oriented structure is great for JSON sensor payloads).
   - **Backend API**: Render Web Service. (Reason: Free tier for Python/FastAPI. Note: spins down after 15 minutes of inactivity).
   - **Frontend**: Vercel. (Reason: Excellent free tier, fast CDN, great support for React/Next.js).

3. **Data Communication Protocol**:
   - ESP32 -> Backend: HTTP POST over Wi-Fi. (Reason: Simpler than MQTT, no need to host a separate broker).
   - Backend -> Database: Motor/PyMongo connection.
   - Backend -> Frontend: REST API (HTTP GET). Optional: WebSockets for real-time ECG streaming.

## System Features & Data Models

### Hospital Room Feature (Core Logic)
1. Every ESP32 hardware device has a unique MAC Address (`device_id`).
2. An Admin pairs a `device_id` to a "Room Number".
3. An Admin assigns a `patient_id` to a "Room Number".
4. When the Backend receives an HTTP POST with `device_id`, it looks up the associated `patient_id` and stores the ECG data under that patient.

### MongoDB Schemas (Draft)

**Users Collection**:
```json
{
  "_id": "ObjectId",
  "username": "dr_smith",
  "password_hash": "...",
  "role": "doctor" // Options: admin, doctor, nurse, patient
}
```

**Patients Collection**:
```json
{
  "_id": "ObjectId",
  "name": "John Doe",
  "assigned_room": "101",
  "assigned_doctors": ["ObjectId(dr_smith)"]
}
```

**Devices Collection**:
```json
{
  "device_id": "AA:BB:CC:DD:EE:FF",
  "room_number": "101",
  "status": "active"
}
```

**ECG_Data Collection**:
```json
{
  "patient_id": "ObjectId",
  "timestamp": "ISO_Date",
  "ecg_value": 345,
  "ml_prediction": "Normal" // Appended after batch processing
}
```

## Useful Tools & Commands

- **Run FastAPI Locally**: `uvicorn backend.main:app --reload`
- **Run React Locally**: `npm run dev`
- **Freeze Python Requirements**: `pip freeze > requirements.txt`

## Open Issues / Notes
- Render free tier delays on cold start. The ESP32 might need retry logic built into its C++ firmware to handle the first failed connection if the server is waking up.
- ML inference needs to be triggered either on every `POST` request or on a scheduled batch (e.g., every 5 seconds). Currently leaning towards batch processing for efficiency.

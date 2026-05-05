# HeartCare Monitor — System Architecture

## Overview

HeartCare Monitor is a real-time ECG monitoring platform connecting ESP32 hardware sensors
to a cloud-hosted web application. Patients and doctors access role-specific dashboards
with live ECG waveforms, historical records, alerts, and clinical notes.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                  │
│  ┌──────────────────┐          ┌──────────────────────────────────┐  │
│  │  Patient Browser │          │       Doctor Browser             │  │
│  │  (Next.js App)   │          │       (Next.js App)              │  │
│  └────────┬─────────┘          └──────────────┬───────────────────┘  │
└───────────┼────────────────────────────────────┼─────────────────────┘
            │  HTTPS + Firebase SDK (WS)          │
┌───────────▼────────────────────────────────────▼─────────────────────┐
│                        FIREBASE PLATFORM                              │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │  Firebase Auth  │  │  Realtime DB     │  │  Cloud Firestore    │  │
│  │  (JWT Tokens)   │  │  /ecg_data/{uid} │  │  users / records   │  │
│  │                 │  │  Live ECG stream │  │  notes / alerts    │  │
│  └─────────────────┘  └────────┬─────────┘  └─────────────────────┘  │
└────────────────────────────────┼──────────────────────────────────────┘
                                  │ Firebase REST API (HTTPS)
┌────────────────────────────────▼──────────────────────────────────────┐
│                         HARDWARE LAYER                                 │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │  ESP32 Microcontroller                                        │     │
│  │  ┌──────────┐   ┌──────────────┐   ┌────────────────────┐    │     │
│  │  │ AD8232   │→  │  ADC Pin 34  │→  │  WiFi → Firebase   │    │     │
│  │  │ ECG Chip │   │  (Sampling)  │   │  REST PUT /ecg     │    │     │
│  │  └──────────┘   └──────────────┘   └────────────────────┘    │     │
│  └───────────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### 1. Frontend (Next.js 14 App Router)

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Framework | Next.js 14 (App Router) | SSR + CSR hybrid rendering |
| Styling | Tailwind CSS + CSS Variables | Design system, dark mode |
| Charts | Recharts | ECG waveform, heart rate trend |
| Auth | Firebase Auth SDK | JWT-based sessions |
| Realtime | Firebase Realtime DB SDK | Live ECG WebSocket stream |
| PDF Export | jsPDF + html2canvas | Report downloads |
| State | React Context + useState/useReducer | App-wide state |

### 2. Backend / Database (Firebase)

#### Firebase Realtime Database Schema
```
/
├── ecg_data/
│   └── {patientUID}/
│       ├── current/
│       │   ├── value: number        # ECG amplitude (0–4095)
│       │   ├── bpm: number          # Heart rate
│       │   ├── status: string       # "normal" | "warning" | "critical"
│       │   └── timestamp: number    # Unix ms
│       └── buffer: [...]            # Last 200 samples for waveform
│
└── alerts/
    └── {patientUID}/
        └── {alertID}/
            ├── type: string
            ├── message: string
            └── timestamp: number
```

#### Cloud Firestore Schema
```
/users/{uid}
  ├── role: "patient" | "doctor"
  ├── name: string
  ├── email: string
  ├── age: number
  ├── gender: string
  ├── phone: string
  ├── photo: string (Storage URL)
  ├── assignedDoctor: uid (for patients)
  └── assignedPatients: [uid] (for doctors)

/ecg_records/{recordId}
  ├── patientUID: string
  ├── date: Timestamp
  ├── avgBPM: number
  ├── minBPM: number
  ├── maxBPM: number
  ├── status: string
  ├── duration: number (seconds)
  └── dataSnapshot: [...] (sampled ECG array)

/doctor_notes/{noteId}
  ├── doctorUID: string
  ├── patientUID: string
  ├── content: string
  ├── prescription: string
  ├── createdAt: Timestamp
  └── updatedAt: Timestamp
```

### 3. ESP32 Hardware

- **Sensor**: AD8232 ECG Module
- **Sampling Rate**: 200Hz (ADC read every 5ms)
- **Upload Rate**: Batch every 500ms to Firebase REST
- **WiFi**: WPA2 home/hospital network
- **Power**: USB or Li-Po battery

---

## Data Flow — Live ECG

```
ESP32 (every 500ms)
  → HTTP PUT /ecg_data/{uid}/current.json
  → Firebase Realtime DB updated

Browser (WebSocket listener)
  → onValue() fires on every DB change
  → New ECG sample appended to buffer (max 200 pts)
  → Recharts re-renders waveform (no page reload)
  → BPM and status indicators update
  → Alert triggers if status = "critical"
```

---

## Security Architecture

| Concern | Solution |
|---------|---------|
| Authentication | Firebase Auth (email/password + JWT) |
| DB Access Control | Firebase Security Rules (role-based) |
| Patient Data Privacy | Each patient's data keyed by UID; doctors can only read assigned patients |
| HTTPS | All Firebase SDK traffic is TLS encrypted |
| API Keys | Firebase config keys are frontend-safe; sensitive ops use Security Rules |

### Firebase Security Rules (Realtime DB)
```json
{
  "rules": {
    "ecg_data": {
      "$uid": {
        ".read": "auth.uid === $uid || root.child('users').child(auth.uid).child('assignedPatients').child($uid).exists()",
        ".write": "auth.uid === $uid"
      }
    }
  }
}
```

---

## Scalability Considerations

- **Firebase Realtime DB** handles up to 200,000 concurrent connections per project
- **Firestore** scales automatically; ECG records paginated (20/page)
- **ESP32** uses HTTP REST (not SDK) to minimize memory footprint
- **Next.js** deployed to Vercel with edge functions for global low latency
- **Future**: Add WebRTC for peer-to-peer doctor-patient video consult

---

## Deployment Architecture

```
GitHub Repository
    ↓ (push to main)
Vercel CI/CD Pipeline
    ↓
Next.js Application (Edge Network)
    ↓                    ↓
Firebase Auth      Firebase Services
(Google Cloud)     (us-central1)
```

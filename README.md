# 🫀 HeartCare Monitor

**Real-Time ECG Monitoring Platform for Patients & Doctors**

[![Next.js](https://img.shields.io/badge/Next.js-14-black?logo=next.js)](https://nextjs.org/)
[![Firebase](https://img.shields.io/badge/Firebase-10-orange?logo=firebase)](https://firebase.google.com/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-3-38bdf8?logo=tailwindcss)](https://tailwindcss.com/)
[![ESP32](https://img.shields.io/badge/ESP32-Arduino-blue)](https://www.espressif.com/)

---

## 📋 Features

### Patient Dashboard
- 📈 Live ECG waveform (real-time, no page refresh)
- 💓 Heart rate (BPM) with trend indicator
- 🚦 Status badges: Normal / Warning / Critical
- 📅 ECG history with date/time filter
- 📄 Download reports as PDF
- 🌙 Dark mode toggle

### Doctor Dashboard
- 👥 Assigned patient list with live status
- 🔴 View any patient's live ECG remotely
- 📋 Add prescriptions / clinical notes
- 🔔 Emergency alert notifications
- 🔍 Search patients

### Hardware (ESP32)
- AD8232 ECG sensor integration
- WiFi → Firebase Realtime Database push
- 200Hz sampling, 500ms upload batch

---

## 🚀 Quick Start

### Prerequisites
- Node.js 18+
- npm or yarn
- Firebase project (free Spark plan works)
- ESP32 + AD8232 ECG module (for hardware)

### 1. Clone & Install

```bash
git clone https://github.com/abhinavtripathi-cloud/heartcare-monitor.git
cd heartcare-monitor
npm install
```

### 2. Firebase Setup

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Create a new project: **HeartCare Monitor**
3. Enable **Authentication** → Email/Password
4. Enable **Realtime Database** (start in test mode)
5. Enable **Cloud Firestore**
6. Go to Project Settings → Your Apps → Add Web App
7. Copy the config

### 3. Environment Variables

```bash
cp .env.example .env.local
```

Edit `.env.local` with your Firebase config:

```env
NEXT_PUBLIC_FIREBASE_API_KEY=your_api_key
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=your_project.firebaseapp.com
NEXT_PUBLIC_FIREBASE_DATABASE_URL=https://your_project-default-rtdb.firebaseio.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your_project_id
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=your_project.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
NEXT_PUBLIC_FIREBASE_APP_ID=your_app_id
```

### 4. Firebase Security Rules

**Realtime Database** → Rules:
```json
{
  "rules": {
    "ecg_data": {
      "$uid": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    },
    "alerts": {
      "$uid": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    }
  }
}
```

**Firestore** → Rules:
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth.uid == uid;
    }
    match /ecg_records/{recordId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null;
    }
    match /doctor_notes/{noteId} {
      allow read, write: if request.auth != null;
    }
  }
}
```

### 5. Run Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000)

---

## 🔌 ESP32 Setup

### Hardware Wiring (AD8232 → ESP32)

| AD8232 Pin | ESP32 Pin |
|-----------|-----------|
| OUTPUT    | GPIO 34 (ADC) |
| 3.3V      | 3V3 |
| GND       | GND |
| LO+       | GPIO 32 |
| LO-       | GPIO 33 |

### Flash Firmware

1. Open `esp32/ecg_sender.ino` in Arduino IDE
2. Install libraries:
   - `WiFi.h` (built-in)
   - `HTTPClient.h` (built-in)
   - `ArduinoJson` (Library Manager)
3. Update credentials in the sketch:
   ```cpp
   const char* ssid = "YOUR_WIFI_SSID";
   const char* password = "YOUR_WIFI_PASSWORD";
   const char* FIREBASE_HOST = "your-project.firebaseio.com";
   const char* FIREBASE_AUTH = "your-database-secret";
   const char* PATIENT_UID = "firebase-patient-uid";
   ```
4. Upload to ESP32

---

## 📁 Project Structure

```
heartcare-monitor/
├── src/
│   ├── app/                    # Next.js App Router pages
│   │   ├── auth/               # Login & Register
│   │   ├── patient/            # Patient-specific pages
│   │   └── doctor/             # Doctor-specific pages
│   ├── components/
│   │   ├── auth/               # Auth forms
│   │   ├── common/             # Shared UI components
│   │   ├── ecg/                # ECG chart components
│   │   └── doctor/             # Doctor-specific components
│   ├── contexts/               # React context providers
│   ├── hooks/                  # Custom hooks
│   ├── lib/                    # Firebase, PDF utilities
│   └── utils/                  # ECG analysis, formatters
├── esp32/
│   └── ecg_sender.ino          # ESP32 Arduino sketch
├── public/                     # Static assets
├── architecture.md
├── approach.md
├── implementation.md
└── README.md
```

---

## 🏗️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | Next.js 14 (App Router) |
| Styling | Tailwind CSS |
| Charts | Recharts |
| Authentication | Firebase Auth |
| Realtime Data | Firebase Realtime Database |
| Persistent Data | Cloud Firestore |
| PDF Export | jsPDF |
| State Management | React Context |
| Hardware | ESP32 + AD8232 |

---

## 🔒 Environment Variables Reference

| Variable | Description |
|---------|-------------|
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase Web API Key |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Firebase Auth Domain |
| `NEXT_PUBLIC_FIREBASE_DATABASE_URL` | Realtime DB URL |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Firebase Project ID |
| `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | Storage Bucket |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | Messaging Sender ID |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | Firebase App ID |

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

## 👥 Team

Built with ❤️ for better cardiac care. 

> ⚠️ **Medical Disclaimer**: HeartCare Monitor is a monitoring aid, not a certified medical device. Always consult a qualified healthcare professional for diagnosis and treatment.

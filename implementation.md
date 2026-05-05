# HeartCare Monitor — Implementation Guide

## Step-by-Step Implementation Reference

This document provides implementation details, patterns, and gotchas for every major
system in HeartCare Monitor.

---

## Phase 1: Firebase Setup (Day 1)

### 1.1 Create Firebase Project

```bash
# Install Firebase CLI
npm install -g firebase-tools
firebase login
firebase init

# Select:
# ✓ Firestore
# ✓ Realtime Database
# ✓ Hosting (optional, use Vercel instead)
```

### 1.2 Firebase Config File (`src/lib/firebase.js`)

```javascript
import { initializeApp, getApps } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getDatabase } from 'firebase/database';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  // ... other keys
};

// Prevent multiple initializations (Next.js hot reload issue)
const app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApps()[0];

export const auth = getAuth(app);
export const db = getDatabase(app);   // Realtime DB
export const firestore = getFirestore(app); // Firestore
```

---

## Phase 2: Authentication (Day 1-2)

### 2.1 Registration Flow

```
POST /auth/register
1. createUserWithEmailAndPassword(auth, email, password)
2. Set displayName via updateProfile()
3. Write to Firestore: /users/{uid} { role, name, email, ... }
4. Redirect to /{role}/dashboard
```

### 2.2 AuthContext Pattern

```jsx
// Use context to avoid prop-drilling auth state
const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [userProfile, setUserProfile] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    return onAuthStateChanged(auth, async (firebaseUser) => {
      if (firebaseUser) {
        setUser(firebaseUser);
        const profile = await getUserProfile(firebaseUser.uid);
        setUserProfile(profile);
      } else {
        setUser(null);
        setUserProfile(null);
      }
      setLoading(false);
    });
  }, []);
}
```

### 2.3 Route Protection (Middleware)

```javascript
// middleware.js
export function middleware(request) {
  const token = request.cookies.get('auth-token');
  const role = request.cookies.get('user-role');
  
  if (!token && request.nextUrl.pathname.startsWith('/patient')) {
    return NextResponse.redirect(new URL('/auth/login', request.url));
  }
  // ... role-based redirects
}
```

---

## Phase 3: ECG Real-Time Hook (Day 2-3)

### 3.1 `useECGData` Hook

```javascript
// Returns: { ecgBuffer, bpm, status, isConnected }
export function useECGData(patientUID) {
  const [ecgBuffer, setEcgBuffer] = useState(Array(200).fill(2048));
  const [bpm, setBpm] = useState(0);
  const [status, setStatus] = useState('normal');
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    if (!patientUID) return;
    
    const ecgRef = ref(db, `ecg_data/${patientUID}/current`);
    
    const unsubscribe = onValue(ecgRef, (snapshot) => {
      const data = snapshot.val();
      if (!data) return;
      
      setIsConnected(true);
      setBpm(data.bpm);
      setStatus(data.status);
      
      // Sliding window: keep last 200 samples
      if (data.buffer && Array.isArray(data.buffer)) {
        setEcgBuffer(data.buffer.slice(-200));
      }
    }, (error) => {
      setIsConnected(false);
      console.error('ECG stream error:', error);
    });
    
    return () => unsubscribe(); // Cleanup WebSocket on unmount
  }, [patientUID]);

  return { ecgBuffer, bpm, status, isConnected };
}
```

### 3.2 ECG Chart Component

```jsx
// Recharts LineChart with no dots for performance
<LineChart data={chartData} width={width} height={height}>
  <Line
    type="monotone"
    dataKey="value"
    stroke="#00D4AA"
    strokeWidth={2}
    dot={false}          // Critical: dots at 200pts = huge CPU cost
    isAnimationActive={false}  // Critical: disable animation for realtime
  />
  <YAxis domain={[0, 4095]} hide />
  <XAxis dataKey="t" hide />
</LineChart>
```

**Why `isAnimationActive={false}`?**
Recharts animates transitions by default. With 200 data points updating every 500ms,
animation causes visual artifacts. Disable it for smooth ECG display.

---

## Phase 4: Firestore Data Layer (Day 3-4)

### 4.1 Save ECG Record

```javascript
export async function saveECGRecord(patientUID, data) {
  await addDoc(collection(firestore, 'ecg_records'), {
    patientUID,
    date: serverTimestamp(),
    avgBPM: data.avgBPM,
    minBPM: data.minBPM,
    maxBPM: data.maxBPM,
    status: data.status,
    duration: data.duration,
    dataSnapshot: data.ecgSamples.slice(0, 500), // Store 500 sample snapshot
  });
}
```

### 4.2 Query ECG History with Date Filter

```javascript
export async function getECGHistory(patientUID, startDate, endDate) {
  const q = query(
    collection(firestore, 'ecg_records'),
    where('patientUID', '==', patientUID),
    where('date', '>=', startDate),
    where('date', '<=', endDate),
    orderBy('date', 'desc'),
    limit(20)
  );
  const snapshot = await getDocs(q);
  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
```

### 4.3 Doctor Notes

```javascript
export async function addDoctorNote(doctorUID, patientUID, content, prescription) {
  await addDoc(collection(firestore, 'doctor_notes'), {
    doctorUID,
    patientUID,
    content,
    prescription,
    createdAt: serverTimestamp(),
    updatedAt: serverTimestamp(),
  });
}
```

---

## Phase 5: PDF Report Generation (Day 4)

```javascript
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';

export async function generateECGReport(patientProfile, ecgData, chartElementId) {
  const pdf = new jsPDF('p', 'mm', 'a4');
  
  // Header
  pdf.setFontSize(20);
  pdf.setTextColor(13, 27, 42);
  pdf.text('HeartCare Monitor — ECG Report', 15, 20);
  
  // Patient info table
  pdf.setFontSize(12);
  pdf.text(`Patient: ${patientProfile.name}`, 15, 35);
  pdf.text(`Date: ${new Date().toLocaleDateString()}`, 15, 42);
  pdf.text(`Average BPM: ${ecgData.avgBPM}`, 15, 49);
  pdf.text(`Status: ${ecgData.status.toUpperCase()}`, 15, 56);
  
  // Capture chart as image
  const chartElement = document.getElementById(chartElementId);
  if (chartElement) {
    const canvas = await html2canvas(chartElement, { scale: 2 });
    const imgData = canvas.toDataURL('image/png');
    pdf.addImage(imgData, 'PNG', 15, 65, 180, 80);
  }
  
  pdf.save(`ECG_Report_${patientProfile.name}_${Date.now()}.pdf`);
}
```

---

## Phase 6: ESP32 Firmware (Day 4-5)

### Key Implementation Points

1. **R-Peak Detection (BPM calculation)**:
   ```cpp
   // Simple threshold-crossing BPM detection
   if (ecgValue > threshold && lastECGValue <= threshold) {
     unsigned long now = millis();
     if (lastPeakTime > 0) {
       bpm = 60000 / (now - lastPeakTime);
     }
     lastPeakTime = now;
   }
   ```

2. **Firebase REST Upload**:
   ```cpp
   String url = "https://" + FIREBASE_HOST + 
                "/ecg_data/" + PATIENT_UID + 
                "/current.json?auth=" + FIREBASE_AUTH;
   http.PUT(payload);
   ```

3. **Lead-off Detection**:
   ```cpp
   if (digitalRead(LO_PLUS) || digitalRead(LO_MINUS)) {
     // Electrodes not properly attached — don't upload garbage data
     return;
   }
   ```

---

## Phase 7: Alert System (Day 5)

### Client-side Alert Logic

```javascript
// In useECGData hook
useEffect(() => {
  if (bpm === 0) return;
  
  if (bpm < 40 || bpm > 150) {
    setStatus('critical');
    // Trigger alert modal
    onCriticalAlert?.({ bpm, message: bpm < 40 ? 'Bradycardia detected' : 'Tachycardia detected' });
  } else if (bpm < 60 || bpm > 100) {
    setStatus('warning');
  } else {
    setStatus('normal');
  }
}, [bpm]);
```

---

## Phase 8: UI Implementation Patterns

### Status Badge Colors

```jsx
const STATUS_CONFIG = {
  normal:   { bg: 'bg-emerald-100', text: 'text-emerald-700', dot: 'bg-emerald-500' },
  warning:  { bg: 'bg-amber-100',   text: 'text-amber-700',   dot: 'bg-amber-500'   },
  critical: { bg: 'bg-red-100',     text: 'text-red-700',     dot: 'bg-red-500 animate-pulse' },
};
```

### Real-time BPM Formatting

```jsx
// Smooth BPM transition (avoid jarring jumps)
const [displayBPM, setDisplayBPM] = useState(0);
useEffect(() => {
  const timer = setTimeout(() => setDisplayBPM(bpm), 150);
  return () => clearTimeout(timer);
}, [bpm]);
```

---

## Testing Checklist

### Without Hardware (Simulated ECG)
1. Run `npm run dev`
2. Register as Patient → see simulated ECG (sine wave with noise)
3. Register as Doctor → assign yourself as doctor
4. Test alert by simulating BPM > 150

### With ESP32
1. Flash firmware with your Firebase credentials
2. Connect AD8232 electrodes
3. Open Patient dashboard → live ECG appears
4. Test lead-off detection (remove electrode → flat line)

---

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| ECG not updating | Firebase Rules blocking | Set rules to allow auth reads |
| Chart flickering | Animation enabled | Set `isAnimationActive={false}` |
| Auth loop | Stale cookie | Clear browser storage |
| ESP32 POST fails | Wrong Firebase URL | Check `.json` suffix in URL |
| BPM = 0 | Lead-off | Check electrode placement |
| Dark mode flash | CSS load order | Add `suppressHydrationWarning` |

---

## Deployment to Vercel

```bash
# Install Vercel CLI
npm install -g vercel

# Deploy
vercel --prod

# Set env vars in Vercel dashboard:
# Project → Settings → Environment Variables
# Add all NEXT_PUBLIC_FIREBASE_* vars
```

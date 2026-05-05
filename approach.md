# HeartCare Monitor — Engineering Approach

## Philosophy

This document explains the engineering decisions, trade-offs, and reasoning behind
each major technical choice in the HeartCare Monitor platform.

---

## 1. Why Next.js 14 (App Router)?

**Decision**: Next.js 14 with App Router over plain React or Vue.

**Reasoning**:
- **SSR for auth pages**: Login/register pages are server-rendered → no flash of unauthenticated content
- **Route-based code splitting**: Patient and doctor dashboards load only their own code chunks
- **Built-in API routes**: If we add backend logic later (e.g., alert push notifications), we don't need a separate server
- **Vercel deployment**: Zero-config CI/CD, edge functions, and automatic HTTPS

**Trade-off**: App Router has a steeper learning curve than Pages Router. We mitigate this with clear layout files and a consistent file structure.

---

## 2. Why Firebase over Supabase?

**Decision**: Firebase (Auth + Realtime DB + Firestore) over Supabase.

**Reasoning**:

| Feature | Firebase | Supabase |
|---------|---------|---------|
| Realtime WebSocket | Native (Realtime DB) | Via Postgres LISTEN/NOTIFY |
| ESP32 HTTP Push | Simple REST API | REST API (similar) |
| Offline support | Built-in persistence | Requires manual setup |
| Free tier | 1GB Realtime DB, 1M reads/day | 500MB DB, limited realtime |
| Latency | ~50ms global | ~100ms (depends on region) |

For ECG data — which changes 200x per second — **Firebase Realtime Database** is the correct tool. Its persistent WebSocket connection gives sub-100ms latency for live waveform updates.

**Firestore** is used alongside it for structured data (user profiles, records, notes) where querying and indexing matter more than realtime speed.

---

## 3. Why Two Databases (Realtime DB + Firestore)?

**Decision**: Split data between Firebase Realtime DB and Firestore.

| Data Type | Database | Why |
|-----------|---------|-----|
| Live ECG stream | Realtime DB | WebSocket push, JSON tree, perfect for time-series |
| User profiles | Firestore | Queries, indexing, rich documents |
| ECG history records | Firestore | Date filtering, pagination, complex queries |
| Doctor notes | Firestore | Rich text, timestamps, joins |
| Alert buffer | Realtime DB | Instant propagation needed |

This is the standard Firebase architecture pattern for applications with both live and persistent data.

---

## 4. ECG Waveform Rendering Strategy

**Problem**: ECG data arrives at 200Hz. Rendering 200 React re-renders per second would destroy performance.

**Solution**: Buffered windowed rendering.

```
ESP32 uploads batch every 500ms
    ↓
Firebase onValue() fires (WebSocket)
    ↓
JavaScript buffer: last 200 samples (1 second of data)
    ↓
Recharts LineChart renders full 200-point window
    ↓
Next batch arrives → oldest 100 samples removed → 100 new added
    ↓
One React render per 500ms (not per sample)
```

This gives a smooth scrolling ECG appearance at 2 renders/second — imperceptible lag, zero jank.

**Why Recharts over Chart.js?**
- Recharts is React-native (no imperative DOM manipulation)
- Better TypeScript support
- Easier custom tooltip and axis components
- Chart.js requires a canvas ref and manual cleanup; leads to memory leaks in React strict mode

---

## 5. Authentication & Role-Based Access

**Decision**: Firebase Auth with custom Firestore claims approach.

**Flow**:
1. User registers with role selection (patient/doctor)
2. Firestore `/users/{uid}` document created with `role` field
3. On login, `role` is read from Firestore
4. Next.js middleware checks role → redirects to correct dashboard
5. Firebase Security Rules enforce data access at the DB level

**Why not Firebase Custom Claims?**
Custom claims require Admin SDK (server-side). For a frontend-only deployment, reading role from Firestore is simpler and equally secure when combined with proper Security Rules.

---

## 6. ESP32 Communication Strategy

**Decision**: HTTP REST PUT (not MQTT, not WebSocket from ESP32).

**Reasoning**:
- Firebase Realtime DB REST API is simple: one PUT request per batch
- ESP32 has limited RAM — maintaining a WebSocket is expensive
- HTTP is stateless — easier error recovery (just retry the PUT)
- Battery-friendly: radio on → PUT → radio off pattern possible

**Upload format**:
```json
PUT /ecg_data/{uid}/current.json
{
  "value": 2048,
  "bpm": 72,
  "status": "normal",
  "timestamp": 1699000000000,
  "buffer": [2040, 2048, 2055, 2049, ...]
}
```

---

## 7. Alert System Design

**Alert trigger logic** (runs client-side for zero latency):

```
BPM < 40  → Critical (bradycardia)
BPM > 150 → Critical (tachycardia)
BPM < 60  → Warning
BPM > 100 → Warning
Otherwise → Normal
```

**Alert propagation**:
1. ESP32 calculates rough BPM (R-peak detection)
2. Firebase entry status field updated
3. Browser listener detects status change
4. If Critical: popup modal + red pulsing indicator
5. Doctor dashboard shows alert badge on patient card
6. Alert logged to Firestore for history

**Why client-side alert logic?**
Firebase Cloud Functions (server triggers) add ~2-3 seconds of latency. For critical cardiac alerts, client-side detection with immediate UI response is safer.

---

## 8. PDF Report Generation

**Decision**: jsPDF + html2canvas (client-side only, no server needed).

**Flow**:
1. Patient clicks "Download Report"
2. `html2canvas` captures the ECG chart canvas
3. `jsPDF` creates PDF with:
   - Patient info header
   - ECG chart image
   - BPM statistics table
   - Doctor notes (if any)
4. PDF downloads directly in browser

No server-side PDF generation needed — reduces complexity and cost.

---

## 9. Dark Mode Implementation

**Decision**: CSS variables + Tailwind's `dark:` classes + localStorage.

```css
:root { --bg: #F0F4F8; --surface: #FFFFFF; --text: #0D1B2A; }
.dark { --bg: #0D1B2A; --surface: #1A2E4A; --text: #E8F4FD; }
```

ThemeContext provides `toggleTheme()` and `isDark` to all components.
Preference persisted in localStorage and applied before first render (no flash).

---

## 10. Performance Budget

| Metric | Target | Approach |
|--------|--------|---------|
| First Contentful Paint | < 1.5s | SSR login page, code split |
| ECG render latency | < 500ms | Buffered batch upload |
| Dashboard load | < 2s | Lazy load charts |
| Lighthouse Score | > 90 | Image optimization, font preload |
| Bundle size | < 300KB | Dynamic imports for chart |

---

## 11. Folder Structure Rationale

```
src/app/           → Next.js routes (collocated with their pages)
src/components/    → Reusable UI atoms and molecules
src/contexts/      → Global state (Auth, Theme)
src/hooks/         → Data-fetching and Firebase hooks
src/lib/           → Third-party integrations (Firebase init, jsPDF)
src/utils/         → Pure functions (ECG analysis, date formatters)
```

This follows the **feature-adjacent** organization pattern — components live near the features they serve, not in a flat global components folder.

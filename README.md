# waterMETERCAM-firebase

```
┌─────────────┐      HTTPS POST (image + metadata)       ┌─────────────────────┐
│  ESP32-CAM  │ ───────────────────────────────────────► │  Cloud Function     │
│  (Edge)     │                                         │  (Ingest API)       │
└─────────────┘                                         └──────────┬──────────┘
                                                                    │
                              ┌─────────────────────────────────────┼──────────────────────┐
                              ▼                                     ▼                      ▼
                    ┌─────────────────┐                 ┌─────────────────────┐   ┌────────────────┐
                    │ Firebase Storage│                 │ Firestore (metadata)│   │ Cloud Function │
                    │ /meter_images/  │                 │ /devices/{id}/      │   │ (OCR Trigger)  │
                    │ {device}/{ts}.jpg│                 │   readings/{doc}    │   └───────┬────────┘
                    └─────────────────┘                 └─────────────────────┘           │
                                                                                          ▼
                                                                               ┌─────────────────────┐
                                                                               │ Google Cloud Vision │
                                                                               │ API (Text Detection)│
                                                                               └──────────┬──────────┘
                                                                                          │
                                                                                          ▼
                                                                               ┌─────────────────────┐
                                                                               │ Firestore (update   │
                                                                               │  with meterReading) │
                                                                               └──────────┬──────────┘
                                                                                          │
                              ┌──────────────────────────────────────────────────────────┘
                              ▼
                    ┌─────────────────┐
                    │ Firebase Hosting│
                    │  (React/Web App)│◄── Consumer views real-time analytics & history
                    └─────────────────┘
```


# logic 


---

Component Breakdown

### A. ESP32-CAM (The Edge Device)
**Role:** Capture and transmit only. No heavy logic.
- **WiFi Connection:** Standard `WiFi.h` connection.
- **Time Sync:** Sync with NTP server (pool.ntp.org) on every boot/wake cycle. This guarantees the timestamp is real-world accurate.
- **Image Capture:** Use the OV2640 camera at **VGA (640×480)** or **CIF** resolution with JPEG quality set to **10–20**. This keeps the payload under ~20KB, preventing ESP32 heap exhaustion and reducing latency.
- **Transmission:** Perform an **HTTPS POST** as `multipart/form-data` to a Firebase Cloud Function public URL.
  - Payload fields: `deviceId`, `timestamp` (ISO 8601 string), `image` (binary JPEG).
- **Power:** If battery-powered, use **Deep Sleep** between captures (e.g., every 15 mins). If mains-powered, a simple `delay()`.

### B. Firebase Cloud Functions (The Backend Brain)
Two primary functions:

1. **`ingestImage` (HTTP Trigger)**
   - Receives the ESP32 POST request.
   - Validates a pre-shared API key (simple security against spam).
   - Generates a filename: `{deviceId}_{YYYYMMDD_HHMMSS}.jpg`.
   - Uploads the image buffer to **Firebase Storage** at `meter_images/{deviceId}/{filename}`.
   - Writes an initial document to Firestore at `devices/{deviceId}/readings/{autoId}`:
     ```json
     {
       "deviceId": "esp32_01",
       "timestamp": <Firestore Timestamp>,
       "formattedTime": "15-01-2025 14:32:05",
       "imagePath": "meter_images/esp32_01/...",
       "imageUrl": null,
       "meterReading": null,
       "status": "PENDING",
       "rawOcrText": null
     }
     ```
   - Returns `200 OK` to ESP32 immediately so it can go back to sleep.

2. **`processMeterOCR` (Storage Trigger)**
   - Fires automatically when a new `.jpg` lands in Storage.
   - Downloads the image to the Function’s temporary memory (`/tmp`).
   - Calls **Google Cloud Vision API** (`TEXT_DETECTION`).
   - **Post-Processing Logic:**
     - Extract all text blocks.
     - Apply regex to find numeric patterns (e.g., `\d+\.?\d*`).
     - **Validation:** Compare against the previous confirmed reading. If the new reading is lower, or jumps by an impossible amount (e.g., +10,000L in 1 hour), flag as `REVIEW` instead of `CONFIRMED`.
   - Updates the Firestore document with:
     - `meterReading` (float/integer)
     - `rawOcrText`
     - `status` (`CONFIRMED` or `REVIEW`)
     - `consumptionDelta` (difference from last confirmed reading)
     - `imageUrl` (signed public URL or public Storage URL if rules allow)

3. **`aggregateDaily` (Scheduled Trigger — optional but recommended)**
   - Runs every hour or day.
   - Pre-computes daily/monthly totals into a `analytics` collection so the website reads cheap, fast documents instead of running expensive queries on thousands of readings.

### C. Firebase Storage (The Image Lake)
- **Path Convention:** `meter_images/{deviceId}/{YYYYMMDD}_{HHMMSS}.jpg`
- **Lifecycle:** Keep images for 30–90 days, then auto-delete or move to cold storage if costs matter. The Firestore database retains the actual numerical history forever.
- **Rules:** Write = only from Cloud Functions (via Admin SDK). Read = public or authenticated (depending on if you want consumers to see raw meter photos).

### D. Firestore (The Structured Database)
**Schema:**
```
devices/{deviceId} {
  location: string,
  ownerId: string,
  lastConfirmedReading: number,
  lastReadingTimestamp: timestamp,
  createdAt: timestamp
}

devices/{deviceId}/readings/{readingId} {
  timestamp: timestamp,
  formattedTime: string,       // "15-01-2025 14:32:05"
  meterReading: number | null,
  consumptionDelta: number | null,
  imageUrl: string,
  rawOcrText: string,
  status: "PENDING" | "CONFIRMED" | "REVIEW"
}

devices/{deviceId}/analytics/{YYYY-MM} {
  month: string,
  totalConsumption: number,
  dailyBreakdown: {
    "01": 120.5,
    "02": 115.0,
    ...
  },
  averageDailyUse: number
}
```
**Why this schema?**
- Subcollections under `devices` keep data isolated and secure.
- Monthly analytics documents prevent the website from downloading thousands of records to calculate totals.

### E. Consumer Website (Firebase Hosting)
**Stack:** Static React app (or vanilla JS) hosted on Firebase Hosting.
**Features:**
- **Real-Time Dashboard:** Use Firestore `onSnapshot` to show the latest meter reading instantly.
- **Analytics Charts:** (Chart.js)
  - **Line Chart:** Consumption trend over the last 7/30 days.
  - **Bar Chart:** Daily consumption comparison.
  - **Gauge/Card:** Current total meter reading and estimated monthly bill.
- **History Table:** Paginated list of all readings with thumbnails linking to the Storage image.
- **Alerts:** Highlight readings flagged as `REVIEW` (OCR failed validation).

---

## 3. AI / OCR Logic (The Critical Piece)

Since this is a **water meter**, the display is likely either:
- Mechanical rolling digits
- Digital LCD

**Chosen Approach: Google Cloud Vision API + Regex Validation**

| Stage | Action |
|-------|--------|
| **1. Detection** | Vision API `TEXT_DETECTION` finds all text in the image. |
| **2. Extraction** | Regex scans OCR output for the largest contiguous number (e.g., `001234.5` or `12345`). |
| **3. Sanitization** | Remove spaces, commas, and non-numeric chars (except one decimal point). |
| **4. Validation** | Check `newReading >= lastReading`. If not, flag `REVIEW`. If `newReading > lastReading + threshold`, flag `REVIEW`. |
| **5. Fallback** | If no digits found, status remains `PENDING` and website shows "Image capture unclear — check camera angle/lighting." |

**Why not a custom TensorFlow model on the ESP32?**
The ESP32-CAM lacks the RAM/CPU to run a reliable OCR model locally. Offloading to Vision API is more accurate and the cost is negligible at low volume ($1.50 per 1,000 images after 1,000 free/month).

**Future-proofing:** If the meter has analog dials, Vision API may struggle. In that future case, we swap the Cloud Function to use a custom **AutoML Vision** or **Vertex AI** model trained on your specific meter photos without changing the ESP32 or website code.

---

## 4. Timestamp Strategy
- **ESP32:** Uses NTP to get Unix epoch time, then formats locally to `DD-MM-YYYY HH:MM:SS` for display and sends ISO 8601 (`2025-01-15T14:32:05Z`) to the backend.
- **Firestore:** Stores the native `Timestamp` data type for precise querying (e.g., "get all readings between Jan 1 and Jan 31").
- **Website:** Uses `toLocaleString()` to render in the user’s local timezone, while the raw stored time remains consistent.

---

## 5. Security Model
| Layer | Rule |
|-------|------|
| **ESP32 → Function** | Pre-shared secret key in HTTP header (`x-api-key`). |
| **Storage** | Only the Firebase Admin SDK (Cloud Functions) can write. Reads allowed for authenticated users or public if images are non-sensitive. |
| **Firestore** | Users can only read `devices` they own (matched by `ownerId == request.auth.uid`). Writes are denied from client entirely (only backend writes). |
| **Website** | Firebase Auth (Email/Google sign-in) so consumers only see their own water meter data. |

---

## 6. Why This Stack?
- **Firebase Storage** is cheaper and more robust than storing Base64 images in Firestore.
- **Firestore** gives real-time sync to the website without you writing WebSocket code.
- **Cloud Functions** let you process images asynchronously so the ESP32 gets an immediate response and sleeps.
- **Firebase Hosting** serves the website globally with a CDN and custom domain support.

---


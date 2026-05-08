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

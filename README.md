# 🌾 AgriShield
### Smart Climate-Resilient Agricultural Early Warning System

> IoT + AI powered flood prediction and multi-channel early-warning platform protecting paddy farmers and rural agricultural communities from climate unpredictability.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Solution Overview](#2-solution-overview)
3. [System Architecture](#3-system-architecture)
4. [Repository Structure](#4-repository-structure)
5. [Component Deep-Dive](#5-component-deep-dive)
   - [5.1 Firmware (C++ / ESP32)](#51-firmware-c--esp32)
   - [5.2 Cloud Backend & ML Engine](#52-cloud-backend--ml-engine)
   - [5.3 Mobile App (Farmers)](#53-mobile-app-farmers)
   - [5.4 Web Dashboard (Officers / Hierarchy)](#54-web-dashboard-officers--hierarchy)
   - [5.5 SMS Fallback Channel](#55-sms-fallback-channel)
6. [Data Sources & Datasets](#6-data-sources--datasets)
7. [Communication & Fault Tolerance](#7-communication--fault-tolerance)
8. [Security Model](#8-security-model)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Getting Started](#10-getting-started)
11. [Testing & Validation Strategy](#11-testing--validation-strategy)
12. [Cost Breakdown (Per Node)](#12-cost-breakdown-per-node)
13. [Team & Contribution Guide](#13-team--contribution-guide)
14. [License](#14-license)

---

## 1. Problem Statement

Sri Lanka's paddy sector — supporting **1.5M+ smallholder farmers** — is repeatedly devastated by flash floods during Maha/Yala seasons. Existing gaps:

- **No field-level early warning.** District-level forecasts don't translate to "your field, in 3 hours."
- **App-based solutions fail in rural areas** — low smartphone penetration, patchy 4G/data affordability, literacy and language barriers.
- **Manual sluice-gate and irrigation decisions** are reactive, not predictive.
- **Disconnected data silos** — meteorological, hydrological, and reservoir-release data aren't fused into one actionable signal.

## 2. Solution Overview

AgriShield is a **hybrid, tiered-access platform** — it doesn't force every user onto one channel:

| User | Device Reality | Channel |
|---|---|---|
| Smallholder farmer (no smartphone) | Feature phone | **SMS in Sinhala/Tamil** |
| Progressive farmer (has smartphone) | Android device | **AgriShield Farmer App** (offline-first, push + SMS fallback) |
| Agricultural / Irrigation Officer | Desktop/tablet | **Web GIS Dashboard** |
| District / Ministry hierarchy | Desktop | **Web Dashboard — aggregated multi-district view, role-based access** |

The core pipeline: **Solar ESP32 sensor nodes → Cloud AI risk engine → fan-out to SMS, mobile push, and web dashboard simultaneously.**

## 3. System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│ FIELD EDGE LAYER (C++ / ESP32)                                           │
│                                                                          │
│  JSN-SR04T Ultrasonic ─┐                                                 │
│  Capacitive Soil Sensor─┼──▶ ESP32-WROOM-32 ──▶ SIM800L (GSM/GPRS)      │
│  Solar Panel + Li-ion ──┘         │                                      │
│                                    │ HTTPS POST (batched JSON)           │
└────────────────────────────────────┼─────────────────────────────────────┘
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ CLOUD INGESTION LAYER                                                    │
│  FastAPI Telemetry Gateway ──▶ TimescaleDB/PostgreSQL (time-series)     │
│  + OpenWeatherMap / Open-Meteo forecast ingestion (cron)                 │
│  + Irrigation Dept. reservoir release feed (scraper/API)                 │
└────────────────────────────────────┼─────────────────────────────────────┘
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ AI / PREDICTIVE ENGINE                                                   │
│  Feature store ──▶ XGBoost/LightGBM inundation-probability model        │
│  + rule-based override layer (sensor-fault & rate-of-rise guards)        │
│  + risk-scoring service (0–100, per field polygon)                       │
└──────┬───────────────────────┬───────────────────────┬───────────────────┘
       ▼                       ▼                       ▼
┌─────────────┐      ┌───────────────────┐   ┌────────────────────────┐
│ SMS Gateway │      │ Push Notification │   │ Officer Web Dashboard  │
│ (Sinhala/   │      │ Service (Farmer   │   │ (React + GIS map +     │
│  Tamil)     │      │  Mobile App)      │   │  sluice-gate control)  │
└─────────────┘      └───────────────────┘   └────────────────────────┘
```

## 4. Repository Structure

```
agrishield/
├── firmware/                 # C++ / PlatformIO / ESP32 source
│   ├── src/
│   │   ├── main.cpp
│   │   ├── sensors/          # UltrasonicSensor.cpp, SoilMoisture.cpp
│   │   ├── comms/            # GsmModem.cpp (SIM800L), OfflineBuffer.cpp
│   │   ├── power/            # SolarManager.cpp, DeepSleep.cpp
│   │   └── config/           # device_config.h, secrets.h (gitignored)
│   ├── platformio.ini
│   └── test/                 # PlatformIO unit tests (Unity framework)
│
├── backend/                  # Cloud engine
│   ├── app/
│   │   ├── main.py           # FastAPI entrypoint
│   │   ├── ingestion/        # telemetry, weather, reservoir feeds
│   │   ├── ml/                # training pipeline, model registry, inference
│   │   ├── alerts/            # SMS + push dispatch, message templates (si/ta/en)
│   │   └── db/                 # SQLAlchemy models, Alembic migrations
│   ├── requirements.txt
│   └── tests/
│
├── mobile-app/                # Farmer-facing app (Flutter)
│   ├── lib/
│   │   ├── screens/           # field_status, alerts, recommendations
│   │   ├── services/          # offline_cache, push_handler, i18n
│   │   └── l10n/               # si.arb, ta.arb, en.arb
│   └── pubspec.yaml
│
├── web-dashboard/              # Officer / hierarchy console (React + Vite)
│   ├── src/
│   │   ├── pages/               # LiveMap, SluiceControl, DistrictOverview
│   │   ├── components/
│   │   └── services/api.ts
│   └── package.json
│
├── ml-notebooks/                # Model research & experiments (Jupyter)
├── docs/                         # Architecture decision records, hardware BOM
└── README.md
```

## 5. Component Deep-Dive

### 5.1 Firmware (C++ / ESP32)

**Stack:** PlatformIO + Arduino framework, C++17.

Key responsibilities per 15-minute wake cycle:

1. Wake from deep sleep (timer-triggered, RTC).
2. Take **10 ultrasonic + 10 soil-moisture readings**, apply a **median filter** to reject outliers.
3. Compare against rolling average; if deviation **>300%**, tag `SENSOR_FAULT` instead of transmitting a raw anomalous value.
4. Package reading as compact JSON, POST via SIM800L over GPRS.
5. On failure, append to a **circular buffer in flash (up to 200 records)**; batch-upload on reconnect.
6. Return to deep sleep — target **<3 mA average draw** for multi-week solar autonomy even in low-sun conditions.

```cpp
// firmware/src/main.cpp (simplified loop)
void setup() {
  Sensors::init();
  Gsm::init();
  OfflineBuffer::loadFromFlash();
}

void loop() {
  SensorReading reading = Sensors::takeFilteredReading();
  if (Sensors::isFaulty(reading)) {
    Dashboard::flagFault(reading);
  } else if (!Gsm::isConnected()) {
    OfflineBuffer::store(reading);
  } else {
    OfflineBuffer::flushIfAny();
    Gsm::postTelemetry(reading);
  }
  Power::enterDeepSleep(FIFTEEN_MINUTES);
}
```

**Hardware BOM (per node):**

| Component | Purpose |
|---|---|
| ESP32-WROOM-32 | Main MCU, deep-sleep capable |
| JSN-SR04T (waterproof) | Water-level distance sensing |
| Capacitive soil moisture sensor | Corrosion-resistant moisture sensing |
| SIM800L | GSM/GPRS uplink |
| 5V/6W solar panel + 18650 Li-ion + TP4056 | Power autonomy |
| IP65 enclosure | Field durability against monsoon exposure |

### 5.2 Cloud Backend & ML Engine

- **API layer:** FastAPI (async), deployed serverless (e.g. AWS Lambda + API Gateway or a lightweight VPS for the hackathon/pilot phase).
- **Storage:** PostgreSQL + TimescaleDB extension for efficient time-series queries on sensor history.
- **Model:** Gradient-boosted trees (XGBoost/LightGBM) predicting **field inundation probability** over a 2–6 hour horizon, trained on fused sensor + weather + reservoir-release + historical flood-extent features.
- **Feature set:** rate-of-rise of water level, soil saturation, upstream rainfall accumulation (3h/6h/24h), reservoir discharge rate, field elevation/slope (from DEM), historical flood recurrence for that field polygon.
- **Retraining cadence:** scheduled retraining each season using newly logged ground-truth flood events (officer-confirmed via the dashboard) to correct model drift.
- **Alert dispatch:** on risk score crossing threshold, the alerts service fans out to SMS + mobile push + dashboard websocket simultaneously, using pre-approved templates in Sinhala, Tamil, and English.

### 5.3 Mobile App (Farmers)

**Stack:** Flutter (single codebase for Android — the dominant rural device OS).

Designed for **low-literacy, low-connectivity, non-technical users**:

- **Offline-first:** last-known field status and recommendations cached locally; syncs opportunistically.
- **Push notifications** as the primary channel when data is available, with **automatic SMS fallback** if the app hasn't been opened in the last hour or push delivery fails — no farmer is left uninformed for lack of data balance.
- **Icon-driven, minimal-text UI**: a traffic-light field-status widget (green/amber/red) before any text.
- **Full Sinhala/Tamil/English localization** via Flutter's `intl`/`.arb` files, with a large-font, high-contrast "elder mode."
- **Voice playback of alerts** (text-to-speech) for low-literacy users — tap a speaker icon to hear the warning read aloud.
- **One-tap "Confirm Action Taken"** — farmer taps to confirm they've opened drainage/moved crops, feeding back into officer dashboard compliance tracking.

### 5.4 Web Dashboard (Officers / Hierarchy)

**Stack:** React + Vite + a mapping library (e.g. Leaflet/Mapbox GL) for GIS overlays.

- **Live risk map** — every registered field rendered as a polygon, color-coded by current risk score.
- **Role-based access control (RBAC):** Field Officer (single division), District Officer (aggregated view across divisions), Ministry/Hierarchy (national rollup + trend analytics).
- **Sluice-gate alert & control log** — automated alerts to irrigation engineers, with a manual override/acknowledge workflow.
- **Sensor health panel** — surfaces `SENSOR_FAULT` flags and battery/solar-charge telemetry per node for maintenance dispatch.
- **Historical replay** — scrub through past flood events to validate model calls against what actually happened (ground-truth labeling loop).

### 5.5 SMS Fallback Channel

Retained as the **universal baseline channel** — works on any feature phone, no data plan required. Delivered through a local telecom SMS gateway/aggregator API, in the exact bilingual format from the original design (warning + numbered action steps), keyed to a short field ID (e.g. `POL-112`) so multi-field farmers know which plot is at risk.

## 6. Data Sources & Datasets

| Dataset | Use | Link |
|---|---|---|
| NASA POWER | Historical rainfall, humidity, surface wetness for feature engineering & backtesting | https://power.larc.nasa.gov/ |
| JRC/Google Global Surface Water | Historical surface-water dynamics & flood extent maps for labeling | https://global-surface-water.appspot.com/ |
| Open-Meteo API | Real-time & forecast rainfall, soil moisture, atmospheric data | https://open-meteo.com/ |
| Humanitarian Data Exchange – Sri Lanka | River basin boundaries, historical disaster statistics | https://data.humdata.org/group/lka |
| Copernicus / Sentinel-1 SAR imagery | Radar-based flood-extent detection (cloud-penetrating, useful for monsoon conditions) | https://scihub.copernicus.eu/ |
| SRTM / Copernicus DEM | Field elevation & slope for hydrological modeling | https://dwtkns.com/srtm30m/ |
| Sri Lanka Dept. of Meteorology (open bulletins) | Localized short-term forecasts & advisories | http://www.meteo.gov.lk/ |
| Sri Lanka Irrigation Department | Reservoir water-level and discharge bulletins | http://www.irrigation.gov.lk/ |

**Key research foundations:**

1. IoT-based ultrasonic water-level monitoring reliability in paddy/runoff channels — validates the JSN-SR04T sensing approach used here (*IEEE Access*).
2. Evidence that targeted SMS-based agricultural alerts measurably reduce harvest losses in developing-country contexts (*Quarterly Journal of Economics*).
3. Gradient-boosted-tree flood-susceptibility models outperforming heavy physical hydraulic simulations on real-time inference latency (*Journal of Hydrology*).

## 7. Communication & Fault Tolerance

- **15-minute telemetry cycle** with median-filtered readings.
- **Offline buffering** — up to 200 cached readings on-device flash during GSM outages, batch-flushed on reconnect.
- **Anomaly guard** — >300% deviation from rolling average → `SENSOR_FAULT` flag on dashboard instead of a false alert to farmers, preventing warning fatigue.
- **Multi-channel redundancy** — if push delivery to the mobile app fails, SMS is dispatched automatically as a fallback for that farmer within minutes.

## 8. Security Model

- **Device-level:** per-node API keys provisioned at manufacture, rotated via OTA config push; TLS for all HTTPS telemetry.
- **Backend:** JWT-based auth for dashboard/app sessions; RBAC enforced at the API layer, not just UI.
- **Data privacy:** farmer phone numbers stored hashed/encrypted at rest; SMS gateway credentials held in a secrets manager, never in firmware source.
- **Firmware:** signed OTA updates to prevent tampering with field-deployed nodes.

## 9. Implementation Roadmap

| Phase | Duration | Milestones |
|---|---|---|
| **1. Prototyping** | Month 1–2 | Firmware + single sensor node bench-tested; backend ingestion API live; base ML model on historical data |
| **2. Cloud & App Development** | Month 3–4 | Mobile app (Flutter) MVP; web dashboard MVP; SMS gateway integration; RBAC |
| **3. Pilot Deployment** | Month 5–7 | 20–30 nodes deployed in one flood-prone division (e.g. Polonnaruwa); ground-truth feedback loop with officers |
| **4. Nationwide Scale-Up** | Month 8+ | Multi-district rollout; model retraining pipeline; insurer/B2G integrations |

## 10. Getting Started

### Prerequisites
- Python 3.10+
- PostgreSQL 15 + TimescaleDB extension
- PlatformIO CLI (ESP32 firmware)
- Flutter SDK 3.x (mobile app)
- Node.js 18+ (web dashboard)

### Backend
```bash
git clone https://github.com/your-team/agrishield.git
cd agrishield/backend
pip install -r requirements.txt
python manage.py db upgrade
uvicorn app.main:app --reload --port 8000
```

### Firmware
```bash
cd agrishield/firmware
pio run --target upload
pio device monitor
```

### Mobile App
```bash
cd agrishield/mobile-app
flutter pub get
flutter run
```

### Web Dashboard
```bash
cd agrishield/web-dashboard
npm install
npm run dev
```

## 11. Testing & Validation Strategy

- **Firmware:** Unity framework unit tests for filtering/fault-detection logic; bench simulation of GSM dropouts.
- **Backend:** Pytest for ingestion & alert-dispatch logic; load-testing telemetry endpoint for a full node fleet.
- **ML:** Backtesting against historical flood events (Global Surface Water dataset) for precision/recall on the inundation-probability threshold; officer-confirmed ground truth feeds a continuous validation loop.
- **Mobile/Web:** Field usability testing with actual farmers for the app's icon/voice UX, and with officers for dashboard workflow fit.

## 12. Cost Breakdown (Per Node)

| Item | Approx. Cost (LKR) |
|---|---|
| ESP32 + sensors + SIM800L | ~8,000 |
| Solar panel + battery + charge controller | ~4,500 |
| IP65 enclosure + mounting | ~2,000 |
| **Total per node** | **< 15,000** |

## 13. Team & Contribution Guide

- Branch naming: `feature/<component>-<short-desc>`, `fix/<component>-<short-desc>`
- Firmware changes require a bench test log attached to the PR.
- ML model changes require a before/after backtest metric comparison.
- All farmer-facing copy (SMS templates, app strings) must be reviewed for Sinhala/Tamil accuracy before merge.

## 14. License

Specify your team's chosen license here (e.g. MIT for the software stack; hardware designs under CERN-OHL if open-sourced).

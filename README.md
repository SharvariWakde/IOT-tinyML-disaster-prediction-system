# 🌊 HydroSense
### Edge-Intelligent Multi-Hazard Monitoring System
**Flood | Drought | Heavy Rainfall Detection using ESP32, LoRa & TinyML**

> Built for Riverathon 2026 — A low-cost, cloud-independent edge AI system for real-time environmental risk monitoring in rural and semi-urban areas.

---

## 📌 Table of Contents
- [Overview](#overview)
- [Hazards Covered](#hazards-covered)
- [System Architecture](#system-architecture)
- [Hardware Components](#hardware-components)
- [Node Descriptions](#node-descriptions)
- [Edge Logic — Risk Rules](#edge-logic--risk-rules)
- [LoRa Communication](#lora-communication)
- [Cloud Dashboard](#cloud-dashboard)
- [TinyML Model](#tinyml-model)
- [Project Structure](#project-structure)
- [Setup & Flashing](#setup--flashing)
- [JSON Payload Formats](#json-payload-formats)
- [Novelty](#novelty)
- [Limitations](#limitations)
- [Future Work](#future-work)
- [Team](#team)

---

## 🌐 Overview

HydroSense is a two-transmitter + one-receiver LoRa-based wireless sensor network that monitors three environmental hazards in real time:

- **Flood** — via TinyML water level classification on Node 1
- **Drought / Water Scarcity** — via rule-based edge logic on the Receiver
- **Heavy Rainfall** — via rate-of-change analysis on water level + humidity

The system is designed to operate **without internet connectivity**. All critical alerts (buzzer, LED, relay) are triggered locally by the Receiver ESP32. The cloud dashboard is used only for visualization, history, and explainability — not for decisions.

---

## ⚠️ Hazards Covered

| Hazard | Detection Method | Where Processed |
|---|---|---|
| 🌊 Flood | TinyML (Edge Impulse) on water level % | Node 1 (ESP32) |
| 🌵 Drought | Rule-based: soil moisture + temperature + water trend | Receiver (ESP32) |
| 🌧️ Heavy Rainfall | Rule-based: rapid water rise rate + high humidity | Receiver (ESP32) |

---

## 🏗️ System Architecture

```
┌─────────────────────┐        LoRa        ┌──────────────────────────────┐
│      NODE 1         │ ─────────────────► │                              │
│  ESP32 + JSN-SR04T  │                    │         RECEIVER NODE        │
│  TinyML Flood Model │                    │  ESP32 + LoRa + Wi-Fi        │
└─────────────────────┘                    │                              │
                                           │  • Merges Node 1 + Node 2    │
┌─────────────────────┐        LoRa        │  • Drought risk logic        │
│      NODE 2         │ ─────────────────► │  • Heavy rainfall logic      │
│  ESP32 + Soil       │                    │  • Triggers buzzer/LED/relay │
│  Sensor + DHT11     │                    │  • Pushes JSON to cloud      │
└─────────────────────┘                    └──────────────┬───────────────┘
                                                          │ Wi-Fi
                                                          ▼
                                               ┌─────────────────┐
                                               │  Cloud Dashboard │
                                               │  (Visualization) │
                                               └─────────────────┘
```

---

## 🔧 Hardware Components

| Component | Quantity | Purpose |
|---|---|---|
| ESP32 Dev Board | 3 | Node 1, Node 2, Receiver |
| LoRa SX1278 Module (433/868/915 MHz) | 3 | Wireless communication |
| JSN-SR04T Ultrasonic Sensor | 1 | Waterproof water level measurement |
| Capacitive Soil Moisture Sensor | 1 | Soil moisture % reading |
| DHT11 Temperature & Humidity Sensor | 1 | Temp °C + Humidity % |
| Buzzer | 1 | Local audio alert on Receiver |
| LEDs (Red/Yellow/Green) | 3 | Visual risk indicators |
| Relay Module (optional) | 1 | External siren or pump control |
| Power Supply / Battery | — | Per node requirement |

---

## 📡 Node Descriptions

### Node 1 — Flood TinyML Node

**Hardware:** ESP32 + LoRa SX1278 + JSN-SR04T

**Workflow:**
1. JSN-SR04T reads distance (20–450 cm) in waterproof mode
2. Distance is mapped to water level percentage (0–100%)
3. Edge Impulse TinyML model on ESP32 classifies the reading
4. Output tier: `NORMAL` / `WARNING` / `DANGER`
5. Sends JSON payload via LoRa

**Why TinyML?** Low latency, no internet dependency, instant on-device inference.

---

### Node 2 — Soil & Microclimate Node

**Hardware:** ESP32 + LoRa SX1278 + Capacitive Soil Sensor + DHT11

**Workflow:**
1. Reads soil moisture % from capacitive sensor
2. Reads temperature (°C) and humidity (%) from DHT11
3. Packages raw values into JSON
4. Transmits via LoRa to Receiver

No local processing — raw data is sent for the Receiver to interpret.

---

### Receiver Node [NODE 3]— The Edge Brain

**Hardware:** ESP32 + LoRa SX1278 + Wi-Fi + Buzzer + LEDs

**Responsibilities:**
- Listens for LoRa packets from Node 1 and Node 2
- Maintains latest values from both nodes
- Runs drought risk rules
- Runs heavy rainfall detection rules
- Triggers local alerts for any HIGH risk state
- Sends combined multi-hazard JSON to cloud via Wi-Fi

---

## 🧠 Edge Logic — Risk Rules

### 🌊 Flood Risk
Flood tier is determined by the TinyML model on Node 1 and passed as-is to the Receiver.

| Tier | Water Level |
|---|---|
| NORMAL | 0 – 40% |
| WARNING | 40 – 70% |
| DANGER | 70 – 100% |

---

### 🌵 Drought / Water Scarcity Risk

```
IF soil_pct < 25% AND temp_c > 33°C         → DROUGHT = HIGH
IF soil_pct 25–40% AND temp_c > 30°C        → DROUGHT = MEDIUM
IF water_pct rapidly decreasing (trend)     → Increase drought tier by one level
IF all conditions within normal range       → DROUGHT = LOW
```

---

### 🌧️ Heavy Rainfall Risk

```
IF water_pct rise rate > threshold over last N samples
   AND hum_pct > 80%                        → RAINFALL = WARNING

IF water_pct rise rate > critical threshold
   AND hum_pct > 85%                        → RAINFALL = DANGER

IF conditions within normal range           → RAINFALL = LOW
```

> Note: Rise rate is computed as delta(water_pct) / delta(time) over a rolling window of N readings stored on the Receiver.

---

### Combined Multi-Hazard Output

The Receiver generates a final JSON combining all three risk tiers plus a human-readable explanation:

```json
{
  "flood_tier": "DANGER",
  "flood_water_pct": 78.2,
  "drought_tier": "LOW",
  "soil_pct": 45.0,
  "temp_c": 28.0,
  "hum_pct": 87.0,
  "rainfall_tier": "WARNING",
  "explanation": "Flood DANGER due to water level at 78%. Heavy Rainfall WARNING due to rapid water rise and high humidity (87%). Drought LOW — soil moisture adequate.",
  "timestamp": "2026-05-01T10:30:00"
}
```

---

## 📶 LoRa Communication

- **Protocol:** LoRa point-to-multipoint (Nodes TX → Receiver RX)
- **Module:** SX1278 (433 MHz / 868 MHz / 915 MHz depending on region)
- **Library:** RadioHead or LoRa.h (Arduino)
- **Packet format:** JSON string
- **Transmission interval:** Configurable (recommended: every 10–30 seconds)
- **Range tested:** ~X meters indoors / ~X meters outdoors (fill after testing)

---

## ☁️ Cloud Dashboard

The Receiver pushes the combined JSON to a cloud endpoint via Wi-Fi (HTTP POST or MQTT).

**Dashboard features:**
- Live gauges: water level %, soil moisture %, temperature, humidity
- Color-coded risk indicators for Flood, Drought, and Heavy Rainfall
- Human-readable explanation text per reading
- Time-series charts for trend analysis
- Alert history log

> The cloud is **not required** for local alerts. Buzzer and LEDs fire from the Receiver ESP32 regardless of internet connectivity.

**Recommended platforms:** Grafana + InfluxDB, ThingsBoard, Node-RED, or any HTTP/MQTT dashboard.

---

## 🤖 TinyML Model

- **Framework:** Edge Impulse
- **Input feature:** `water_pct` (single float, 0.0 – 100.0)
- **Output classes:** `NORMAL`, `WARNING`, `DANGER`
- **Deployment:** Exported as Arduino library, flashed directly onto Node 1 ESP32
- **Training data:** Water level readings collected across all three tier conditions
- **Model type:** Classification (neural network or decision tree via Edge Impulse)

---

## 📁 Project Structure

```
HydroSense/
├── node1_flood/
│   ├── node1_flood.ino          # Main sketch for Node 1
│   └── ei_model/                # Edge Impulse exported library
│
├── node2_soil/
│   └── node2_soil.ino           # Main sketch for Node 2
│
├── receiver/
│   └── receiver.ino             # Main sketch for Receiver Node
│
├── dashboard/
│   └── dashboard_config/        # Cloud dashboard config files
│
├── docs/
│   ├── HydroSense.pptx          # Project presentation
│   └── architecture_diagram.png
│
└── README.md
```

---

## ⚡ Setup & Flashing

### Prerequisites
- Arduino IDE (v1.8+ or v2.x)
- ESP32 board package installed
- Required libraries:
  - `LoRa.h` or `RadioHead`
  - `DHT sensor library` (Adafruit)
  - `Edge Impulse Arduino Library` (exported from Edge Impulse project)
  - `ArduinoJson`
  - `WiFi.h` (built-in ESP32)
  - `HTTPClient.h` (built-in ESP32)

### Steps

**1. Flash Node 1 (Flood TinyML)**
```
- Open node1_flood/node1_flood.ino in Arduino IDE
- Import Edge Impulse library: Sketch → Include Library → Add .ZIP Library → select ei_model.zip
- Select board: ESP32 Dev Module
- Upload
```

**2. Flash Node 2 (Soil/Climate)**
```
- Open node2_soil/node2_soil.ino
- Select board: ESP32 Dev Module
- Upload
```

**3. Flash Receiver**
```
- Open receiver/receiver.ino
- Set your Wi-Fi credentials and cloud endpoint URL in the config section
- Select board: ESP32 Dev Module
- Upload
```

### LoRa Pin Connections (SX1278 → ESP32)

| SX1278 Pin | ESP32 Pin |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SCK | GPIO 18 |
| MISO | GPIO 19 |
| MOSI | GPIO 23 |
| NSS (CS) | GPIO 5 |
| RST | GPIO 14 |
| DIO0 | GPIO 26 |

> Pin mapping may vary by board variant. Verify before flashing.

---

## 📦 JSON Payload Formats

### Node 1 → Receiver
```json
{
  "id": "N1",
  "type": "flood",
  "water_pct": 72.3,
  "tier": "HIGH"
}
```

### Node 2 → Receiver
```json
{
  "id": "N2",
  "type": "soil",
  "soil_pct": 31.5,
  "temp_c": 34.0,
  "hum_pct": 52.0
}
```

### Receiver → Cloud
```json
{
  "flood_tier": "DANGER",
  "flood_water_pct": 72.3,
  "drought_tier": "MEDIUM",
  "soil_pct": 31.5,
  "temp_c": 34.0,
  "hum_pct": 52.0,
  "rainfall_tier": "LOW",
  "explanation": "Flood DANGER: water at 72.3%. Drought MEDIUM: soil moisture 31.5%, temp 34°C.",
  "timestamp": "2026-05-01T10:30:00"
}
```

---

## 💡 Novelty

1. **Triple-Hazard in One System** — Simultaneously monitors flood, drought, and heavy rainfall from a single integrated low-cost network
2. **TinyML on Node 1 + Rule-Based Edge on Receiver** — Two-tier AI: neural network for flood classification, rule engine for drought and rainfall
3. **Receiver-Centric Safety** — All alerts fire locally from the Receiver ESP32; system works even when cloud or internet is down
4. **Explainable Dashboard** — Shows not just a risk tier but plain-language reasoning derived from live sensor values

---

## ⚠️ Limitations

- Prototype tested in lab/indoor conditions; not yet calibrated in real river or agricultural field settings
- Drought and rainfall rules are threshold-based; edge cases may need tuning
- DHT11 has limited accuracy (±2°C, ±5% RH); SHT31 would improve results
- Single LoRa channel; scaling to many nodes needs frequency/time slot management
- Heavy rainfall rule thresholds need field validation

---

## 🚀 Future Work

- Add SMS / WhatsApp alert integration for last-mile community notification
- Add rain gauge and water flow sensor for more accurate rainfall detection
- Train dedicated TinyML models for drought and rainfall directly on Receiver or separate node
- Solar-powered nodes for fully off-grid permanent deployment
- Multi-node mesh network to cover large river basins or agricultural zones
- Mobile app for community-level alert subscription

---

## 👥 Team
| Sharvari Wakde - ML Engineer | and team of 3 more people handling hardware and system design

**Event:** Hacksagon 2026
**Institution:** [AVM-IIIT GWALIOR]

---

## 📄 License

This project is open-source and submitted as part of Hacksagon 2026.
Feel free to fork, modify, and build upon it for community resilience applications by giving credits .

---

> *"Edge intelligence for community resilience — where the cloud cannot reach."*

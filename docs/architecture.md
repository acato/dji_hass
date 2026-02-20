# dji_hass System Architecture

> **Role:** ARCHITECT
> **Date:** 2026-02-19
> **Status:** Proposal
> **Version:** 0.3.0

---

## 1. Executive Summary

**dji_hass** is a Home Assistant integration that enables alarm-triggered aerial perimeter inspection using DJI Mavic 2 drones.

The system implements **human-in-the-loop autonomy**: Home Assistant automates everything — alarm detection, safety checks, mission selection, dock preparation — but a Part 107 Remote Pilot in Command (RPIC) explicitly authorizes each flight with a single tap. This satisfies FAA requirements while delivering a functionally autonomous security drone response.

Three subsystems work together:

1. **Physical Dock** — weatherproof enclosure with ESPHome controller, keeps drone staged and batteries ready
2. **Android Bridge** — dedicated Android device running DJI Mobile SDK v4, translating between DJI proprietary protocols and MQTT/RTMP
3. **HA Integration** — `custom_components/dji_hass/`, orchestrating the entire workflow as native HA entities, services, and automations

**Design constraint:** The Mavic 2 is Phase-0 hardware (aging platform, 2018). The dock and software are designed to be aircraft-agnostic so they survive a future drone upgrade.

---

## 2. Regulatory Framework

### 2.1 Applicable Law

Drone operation in the U.S. is governed by federal law (FAA), not state law. This use case — property security triggered by an alarm — is **commercial/operational**, meaning **14 CFR Part 107** applies (not recreational hobby rules).

### 2.2 What Part 107 Requires

| Requirement | Status |
|-------------|--------|
| FAA Part 107 certificate (RPIC) | Required |
| FAA registration | Required |
| Remote ID broadcast | Required (module or firmware) |
| Anti-collision strobe for night ops | Required (visible 3 statute miles) |
| Visual Line of Sight (VLOS) | Required (RPIC or visual observer must see drone) |
| Fly under 400 ft AGL | Yes (missions at 80-120 ft) |
| Unrestricted (Class G) airspace | Yes (property is in Class G) |
| Airspace authorization (LAANC) | Not needed for Class G |

### 2.3 The Critical Constraint: No Autonomous Launch

FAA prohibits unsupervised autonomous flight. A Remote Pilot in Command must:
- Be responsible for the flight
- Be able to intervene immediately
- Explicitly authorize takeoff

**The system design satisfies this with a single-tap authorization step.** The pilot receives an actionable notification, reviews pre-flight conditions (displayed by HA), and taps "LAUNCH." Everything else is automated.

This is the same compliance pattern used by DJI Dock, Skydio Dock, and Percepto — marketed as "autonomous" but legally "human-in-the-loop."

### 2.4 VLOS for This Property

The property is ~300 ft x 150 ft, flat, 1 acre. With the dock on the shed roof and the pilot at the house, VLOS is maintainable for all planned mission corridors. The 50+ trees (some >100 ft) mean missions must follow **constrained corridors** that remain visible from the pilot's position, not a simple perimeter orbit.

### 2.5 Washington State Specifics

WA adds minimal flight restrictions beyond FAA:
- Privacy: avoid surveillance where people have reasonable expectation of privacy (neighbor yards/windows)
- Property overflight: perimeter patrol must avoid crossing into neighbors' airspace
- State parks require permission; private residential land is fine

---

## 3. Operational Concept

### 3.1 The Alarm Response Workflow

```
Alarm Sensor (PIR, gate, camera AI)
    │
    ▼
Home Assistant Automation
    │
    ├── Safety Gate Check
    │   ├── Wind < 15 mph?
    │   ├── No rain?
    │   ├── Battery > threshold?
    │   ├── GPS lock confirmed?
    │   ├── Dock connected?
    │   ├── Drone connected?
    │   └── Not already airborne?
    │
    ├── Select Mission Profile
    │   ├── Driveway sensor → front sweep
    │   ├── Backyard motion → rear orbit
    │   ├── Full alarm → full perimeter loop
    │   └── Manual → investigate waypoint
    │
    ├── Prepare Dock
    │   └── Open lid (if closed)
    │
    ▼
Actionable Push Notification to RPIC
    ┌─────────────────────────────────┐
    │ 🚨 Perimeter Alert — East Fence │
    │ Wind OK · GPS OK · Battery 85% │
    │ Mission: Full Perimeter         │
    │                                 │
    │  [LAUNCH DRONE]    [IGNORE]     │
    └─────────────────────────────────┘
    │
    ▼ (RPIC taps LAUNCH — this is the legal compliance moment)
    │
Home Assistant fires dji_hass.execute_mission
    │
    ├── Start live stream
    ├── Execute waypoint mission
    ├── Display live feed in HA dashboard
    ├── Record video (SD card + media server)
    │
    ▼
Mission completes → auto RTH → dock lid closes
    │
    ▼
Audit log: trigger, authorization time, mission, outcome
```

**Target timeline: alarm → airborne in ~30-60 seconds** (with RPIC authorization). The 2-minute SLA is comfortable.

### 3.2 Drone Selection

**Mavic 2 Zoom is the primary security aircraft.** The optical zoom allows staying higher and farther from obstacles while still capturing detail. This improves safety margin and VLOS reliability.

**Mavic 2 Pro is secondary** — used for redundancy, daylight high-quality capture, or when the Zoom is unavailable.

### 3.3 Mission Design for This Property

With 50+ trees (some >100 ft), missions are **constrained corridor sweeps**, not a simple perimeter orbit.

| Mission | Description | Altitude |
|---------|-------------|----------|
| `front_sweep` | Driveway + front edge | 80 ft (clear corridor) |
| `rear_sweep` | Back fence line | 80 ft |
| `east_edge` | East property boundary | 110 ft (taller trees) |
| `west_edge` | West property boundary | 80 ft |
| `full_perimeter` | All 4 edges sequentially | Variable per segment |
| `corner_ne` / `nw` / `se` / `sw` | Quick corner investigation | 80-110 ft |

Altitude is a **safety spec** (tree clearance + margin), not a fixed number. Camera angle compensates: gimbal at -30 to -45 degrees provides border coverage from higher altitudes.

---

## 4. Physical Dock Design

### 4.1 Purpose

The dock provides: environmental survivability, deterministic staging, battery readiness, and a motorized lid controlled by Home Assistant.

It is NOT a DJI Dock equivalent — it cannot auto-charge or auto-launch. It is a **"rapid supervised deployment station"** that emulates ~95% of dock behavior through automation + human authorization.

### 4.2 Enclosure Specification

| Component | Material | Rationale |
|-----------|----------|-----------|
| Outer enclosure | NEMA 4X polycarbonate or stainless cabinet | Weatherproof, corrosion-resistant |
| Lid panel | Aluminum sheet with internal ribs + drip edge | Strong, lightweight, rain shedding |
| Inner liner | Cement board + steel tray under drone bay | Noncombustible (LiPo fire mitigation) |
| Fasteners | Stainless steel (isolated from aluminum) | Galvanic corrosion prevention |
| Seals | EPDM gasket + compression latch | Weathertight, UV-resistant |
| Insulation | Polyisocyanurate (foil-faced), isolated from battery bay | Thermal management |

### 4.3 Environmental Control

Seattle Eastside climate: mild but wet. Condensation is a bigger threat than rain.

| System | Purpose | Implementation |
|--------|---------|----------------|
| Heating | Keep batteries 10-30 C | PTC heater pad (reptile heater class), NOT direct battery contact |
| Ventilation | Condensation/humidity control | 12V fan + filtered intake, dew-point logic |
| Temperature sensing | Interior + battery zone + ambient | DS18B20 or BME280 sensors |
| Humidity sensing | Dew point management | BME280 |
| Smoke/heat detection | LiPo fire early warning | Smoke detector in lid void |
| Drain | Prevent water pooling | Weep holes with insect mesh |

### 4.4 Lid Mechanism

| Component | Spec |
|-----------|------|
| Actuator | 12-24V DC linear actuator, clevis mount |
| Limit switches | Redundant open + closed microswitches |
| Pad clear sensor | ToF or IR beam — prevents closing onto drone |
| Manual override | Physical button + emergency stop |

### 4.5 ESPHome Dock Controller

The dock runs on an **ESP32 with ESPHome**, exposing entities to Home Assistant. Safety interlocks are enforced **locally on the controller**, not in HA — HA sends intents, the ESP32 enforces conditions.

**State machine (firmware-level):**
```
CLOSED → OPENING → OPEN → CLOSING → CLOSED
```

**Safety interlocks (on-controller, not in HA):**
- Cannot open unless power is healthy
- Cannot close unless motors disarmed AND pad-clear sensor confirms landed
- Auto-close timeout with abort on obstruction
- Charger power relay cut on smoke/overtemp (hardware, not software)
- Motion timeout: if actuator runs too long, stop and flag fault

**Entities exposed to HA:**

| Entity | Type | Notes |
|--------|------|-------|
| `cover.drone_dock_lid` | Cover | Open/close/stop |
| `sensor.dock_temperature` | Temperature | Interior |
| `sensor.dock_battery_zone_temp` | Temperature | Near battery/drone |
| `sensor.dock_humidity` | Humidity | Interior |
| `binary_sensor.dock_lid_open` | Binary sensor | Limit switch |
| `binary_sensor.dock_lid_closed` | Binary sensor | Limit switch |
| `binary_sensor.dock_pad_clear` | Binary sensor | ToF/IR |
| `binary_sensor.dock_smoke` | Binary sensor | Smoke detector |
| `switch.dock_heater` | Switch | PTC heater relay |
| `switch.dock_fan` | Switch | Ventilation fan relay |
| `switch.dock_charger_power` | Switch | Smart outlet / relay for OEM charger |
| `sensor.dock_power_status` | Sensor | Mains/UPS status |

### 4.6 Placement

On the shed roof. Requirements:
- Clear vertical column above (no overhanging branches)
- Clear lateral clearance ~30-50 ft for takeoff/landing
- Clear approach lane for return-to-home (no branches in the direction of final approach)
- Power from shed below
- WiFi coverage from house network

### 4.7 Power Architecture

```
Mains (from shed) → UPS → 12/24V DC supply
                          ├── Actuator rail (fused)
                          ├── Compute/sensor rail (fused)
                          └── Heater/fan rail (fused)

Separate smart outlet → DJI OEM charger → battery hub
(HA controls the outlet; never modify DJI charging electronics)
```

UPS is important: brownouts during storms are common — exactly when you want the system most.

---

## 5. Battery Lifecycle Management

### 5.1 The Core Problem

Mavic 2 LiPo batteries degrade quickly when stored at high charge. They self-discharge (generating heat), bloat after relatively few cycles, and are designed for intermittent recreational use — not persistent readiness.

**Design mindset: batteries are consumables** (like tires). Plan for annual replacement.

### 5.2 Charge Strategy

| Role | SOC Target | Location | Rotation |
|------|------------|----------|----------|
| Hot standby (installed in drone) | 80-85% | In dock | Rotated weekly |
| Ready spare | 55-65% (storage band) | In charging hub inside dock | Promoted to standby weekly |
| Charging / cooling | Cycling | Charging hub | As needed |

**Inventory:** 3 batteries per drone minimum (6 total for two drones).

### 5.3 Automated Maintenance (via HA)

| Automation | Trigger | Action |
|------------|---------|--------|
| Weekly rotation reminder | Schedule (Sunday AM) | Notify pilot to swap batteries |
| Charge maintenance | Standby drops below 75% | Enable charger power outlet for bounded window |
| Thermal gating | Dock temp outside 5-40 C | Disable charger power |
| Quarterly deep cycle | Schedule (quarterly) | Remind pilot to full cycle all packs |
| Swelling/degradation check | Every rotation | Visual inspection checklist notification |

### 5.4 Expected Lifespan

| Strategy | Expected Useful Life |
|----------|---------------------|
| Always at 100% | 3-6 months |
| Managed standby at 80-85% | 9-15 months |
| Rotated storage regime | 12-24 months |

### 5.5 DJI Battery Settings

- Set auto-discharge delay to 1-3 days (default ~10 days) so packs self-drain to storage level faster if not flown
- This prevents prolonged high-SOC aging

### 5.6 What NOT To Attempt

- Permanent powered drone in dock
- DIY charging contacts on the aircraft
- Robotic battery swapping
- Unattended overnight charging cycles
- Third-party batteries in a dock scenario (highest failure item in unattended deployments)

---

## 6. DJI SDK Reality Check

### 6.1 What's Available for Mavic 2

| SDK | Platform | Mavic 2 Support | Server-Side? |
|-----|----------|-----------------|--------------|
| Mobile SDK v4 | Android, iOS | **Yes** | No — requires physical device + RC USB |
| Mobile SDK v5 | Android, iOS | **No** (Mavic 3+ only) | No |
| Windows SDK | Windows 10+ UWP | **Yes** | Partial — needs Windows box + RC USB |
| Onboard SDK | Linux (C++) | **No** (Matrice only) | Yes, but not for Mavic 2 |
| Cloud API | REST/MQTT/WS | **No** (Enterprise only) | Yes, but not for Mavic 2 |

### 6.2 What Mobile SDK v4 Provides

| Capability | API | Detail |
|------------|-----|--------|
| Takeoff / Land / RTH | `FlightController` | Programmatic takeoff, landing with confirmation, go-home |
| Virtual Stick | `FlightController` | Pitch/roll/yaw/throttle at up to 50 Hz |
| Waypoint Missions | `WaypointMissionOperator` | Up to 99 waypoints, 15 actions each, curved/straight paths |
| Hotpoint Missions | `HotpointMissionOperator` | Orbit a point of interest |
| Camera Control | `Camera` | Photo, video record, exposure (ISO, aperture f/2.8-f/11, shutter), RAW/JPEG |
| Gimbal Control | `Gimbal` | 3-axis, pitch -90 to +30 deg, multiple modes |
| Telemetry | `FlightControllerState` | 10 Hz: GPS, altitude, velocity, heading, flight mode, motors, wind |
| Battery | `Battery` | Charge %, voltage, current, temp, cycles, remaining mAh |
| Signal Strength | `AirLink` | Uplink/downlink quality |
| Live Video | `VideoFeeder` + `LiveStreamManager` | H.264 RTMP push (720p, 1-5 Mbps) |
| Media Download | `MediaManager` | Download photos/videos from SD card |

### 6.3 What's NOT Available

- **No RTSP** — only in MSDK v5 (doesn't support Mavic 2)
- **No server-side control** — always requires Android/iOS/Windows intermediary
- **No headless SDK** — DJI SDK requires a UI context on Android (workarounds exist)
- **99-waypoint limit** — flight controller hardware constraint
- **Live feed is 720p** — not the 4K recording resolution
- **No in-aircraft battery charging** — no USB/auxiliary charge path exists
- **No auto-launch** — motors must be armed via controller/app

---

## 7. Software Architecture

### 7.1 High-Level Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         HOME ASSISTANT                            │
│                                                                   │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐   │
│  │   ESPHome Dock       │  │       dji_hass Integration       │   │
│  │   Controller         │  │                                  │   │
│  │                      │  │  ┌──────────┐ ┌──────────────┐  │   │
│  │  cover.dock_lid      │  │  │ MQTT     │ │  Service     │  │   │
│  │  sensor.dock_temp    │  │  │ Coord.   │ │  Handlers    │  │   │
│  │  switch.dock_heater  │  │  └────┬─────┘ └──────┬───────┘  │   │
│  │  binary.dock_smoke   │  │       │              │           │   │
│  │  ...                 │  │  ┌────┴──────────────┴────────┐  │   │
│  └──────────┬───────────┘  │  │     MQTT Client (aiomqtt)  │  │   │
│             │ ESPHome API  │  └────────────┬───────────────┘  │   │
│             │              │               │                  │   │
│  ┌──────────┴───────────┐  │  Sensors │ Binary │ Camera │ DT  │   │
│  │  Mosquitto Broker    │◄─┤──────────────────────────────────┤   │
│  └──────────┬───────────┘  └──────────────────────────────────┘   │
│             │                                                     │
│  ┌──────────┴───────────┐                                        │
│  │  Media Server         │  (go2rtc / mediamtx)                   │
│  │  RTMP in → WebRTC out │                                        │
│  └──────────────────────┘                                        │
└────────────┬──────────────────────────────────────────┬──────────┘
             │ MQTT (tcp/1883)                          │ RTMP (tcp/1935)
             │ LAN / WiFi                               │
┌────────────┴──────────────────────────────────────────┴──────────┐
│                     ANDROID BRIDGE APP                             │
│                     (dedicated Android device + RC)                │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                   DJI Mobile SDK v4                       │    │
│  │  FlightController │ Camera/Gimbal │ LiveStreamManager     │    │
│  └──────────┬────────────────┬───────────────┬──────────────┘    │
│             │                │               │                    │
│  ┌──────────┴────────────────┴───────────────┴──────────────┐    │
│  │                 MQTT Service Layer                         │    │
│  │  - Telemetry publish (10 Hz → 1 Hz configurable)          │    │
│  │  - Command subscribe + execute + respond                   │    │
│  │  - Mission cache + translate JSON → DJIWaypointMission     │    │
│  │  - RTMP push to media server                               │    │
│  │  - LWT: "offline" on disconnect                            │    │
│  └──────────────────────────┬───────────────────────────────┘    │
│                              │ USB                                │
│                    ┌─────────┴─────────┐                         │
│                    │  DJI Remote Ctrl   │                         │
│                    └─────────┬─────────┘                         │
│                              │ OcuSync 2.0                       │
│          ┌───────────────────┴───────────────────┐               │
│          │            PHYSICAL DOCK               │               │
│          │  ┌───────────────────────────────┐     │               │
│          │  │  Mavic 2 Zoom (primary)       │     │               │
│          │  │  or Mavic 2 Pro (secondary)   │     │               │
│          │  └───────────────────────────────┘     │               │
│          │  ESPHome ESP32: lid, sensors, heater   │               │
│          └────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────────┘
```

### 7.2 Component Responsibilities

#### Android Bridge App

**Purpose:** Translates between DJI proprietary SDK and standard MQTT/RTMP.

**Technology:** Kotlin, DJI Mobile SDK v4, Eclipse Paho MQTT client

**Runtime:** Dedicated Android device (old phone, Android mini-PC), always connected to RC via USB, always on WiFi. Runs as an Android Foreground Service with persistent notification and auto-restart.

**Responsibilities:**
- DJI SDK registration and product connection lifecycle
- Subscribe to MQTT command topics, execute DJI SDK calls, publish responses
- Publish telemetry to MQTT (downsampled from 10 Hz to 1 Hz; burst 10 Hz during missions)
- Push live video via RTMP to media server
- Cache mission definitions from retained MQTT messages
- Translate JSON mission format → `DJIMutableWaypointMission`
- Validate missions against SDK constraints before upload
- Auto-reconnect on connection loss (MQTT and DJI SDK)
- MQTT Last Will and Testament: `"offline"` on disconnect

**Hardware:**

| Component | Purpose | Example |
|-----------|---------|---------|
| Android device | Run bridge app | Old phone (Android 7+), Android mini-PC |
| USB OTG cable | Connect Android to RC | USB-C or micro-USB OTG |
| DJI Remote Controller | Bridge to aircraft | Mavic 2 RC |
| WiFi | Connect to HA network | Same LAN |
| Power supply | Keep Android powered | USB charger (always plugged in) |

#### dji_hass HA Integration

**Purpose:** Exposes drone + dock as native HA entities and services. Orchestrates the alarm → authorization → flight workflow.

**Technology:** Python 3.12+, aiomqtt, Home Assistant Core 2024.2+

#### ESPHome Dock Controller

**Purpose:** Physical dock management with local safety interlocks.

**Technology:** ESP32, ESPHome, 12/24V actuator, sensors

**Key principle:** Safety logic runs on the ESP32, not in HA. HA sends intents ("open lid"), the controller enforces preconditions.

---

## 8. MQTT Topic Design

### 8.1 Topic Namespace

All topics under `dji_hass/{drone_id}/` where `drone_id` is the aircraft serial number.

### 8.2 Telemetry (Bridge → HA)

QoS 0 for high-frequency data, QoS 1 for state changes.

```
dji_hass/{drone_id}/telemetry/flight        (1 Hz)
{
  "lat": 47.6062, "lon": -122.3321,
  "alt": 45.2,                    // meters relative to takeoff
  "heading": 127.5,               // degrees
  "speed_x": 2.1, "speed_y": -0.5, "speed_z": 0.0,
  "ground_speed": 2.16,           // m/s computed
  "flight_mode": "GPS",           // ATTI, GPS, SPORT
  "motors_on": true, "is_flying": true,
  "gps_signal": 5,                // 0-10
  "satellite_count": 14,
  "wind_warning": 0,              // 0=none, 1=moderate, 2=strong
  "timestamp": 1739980800
}

dji_hass/{drone_id}/telemetry/battery       (0.2 Hz)
{
  "charge_percent": 78,
  "voltage_mv": 15200, "current_ma": -2100,
  "temperature_c": 32,
  "remaining_mah": 2800, "full_charge_mah": 3600,
  "discharge_cycles": 42,
  "flight_time_remaining_s": 1200,
  "timestamp": 1739980800
}

dji_hass/{drone_id}/telemetry/gimbal        (1 Hz)
{ "pitch": -45.0, "roll": 0.2, "yaw": 127.5, "mode": "YAW_FOLLOW" }

dji_hass/{drone_id}/telemetry/camera        (on change)
{
  "mode": "RECORD_VIDEO",
  "is_recording": true, "recording_time_s": 45,
  "sd_card_remaining_mb": 28500,
  "iso": 100, "shutter_speed": "1/500", "aperture": 2.8
}

dji_hass/{drone_id}/telemetry/signal        (1 Hz)
{ "uplink_quality": 85, "downlink_quality": 92 }
```

### 8.3 State (Bridge → HA)

QoS 1, `retain: true`.

```
dji_hass/{drone_id}/state/connection     "online" | "offline"  (LWT = "offline")
dji_hass/{drone_id}/state/flight         "landed" | "airborne" | "returning_home" | "landing"
dji_hass/{drone_id}/state/mission        { "status": "idle|uploading|executing|paused|completed|error",
                                           "mission_id": null, "progress": 0.0,
                                           "current_waypoint": 0, "total_waypoints": 0, "error": null }
dji_hass/{drone_id}/state/stream         { "is_streaming": true, "rtmp_url": "rtmp://...",
                                           "resolution": "720p", "bitrate_kbps": 3000 }
```

### 8.4 Commands (HA → Bridge)

QoS 1. Request/response pattern with correlation IDs.

```
dji_hass/{drone_id}/command/{action}
  { "id": "uuid", "params": { ... } }

dji_hass/{drone_id}/command/{action}/response
  { "id": "uuid", "success": true, "error": null, "data": { ... } }
```

**Available commands:**

| Category | Commands |
|----------|----------|
| Flight | `takeoff`, `land`, `confirm_landing`, `return_to_home`, `cancel_rth` |
| Mission | `execute_mission` (params: `mission_id`), `pause_mission`, `resume_mission`, `stop_mission` |
| Camera | `take_photo`, `start_recording`, `stop_recording`, `set_camera_mode` |
| Gimbal | `set_gimbal` (params: `pitch`, `mode`), `reset_gimbal` |
| Stream | `start_stream` (params: `rtmp_url`), `stop_stream` |
| System | `set_home` (params: `lat`, `lon`) |

### 8.5 Mission Definitions (HA → Bridge)

Retained MQTT messages cached by the bridge:

```
dji_hass/{drone_id}/missions/{mission_id}   (retained)
{
  "id": "full_perimeter",
  "name": "Full Perimeter Sweep",
  "speed_mps": 5.0,
  "finish_action": "GO_HOME",
  "heading_mode": "AUTO",
  "flight_path_mode": "CURVED",
  "waypoints": [
    { "lat": 47.6062, "lon": -122.3321, "alt": 24.4,
      "speed_mps": 5.0, "gimbal_pitch": -45.0,
      "stay_ms": 2000, "actions": ["START_TAKE_PHOTO"] },
    { "lat": 47.6065, "lon": -122.3318, "alt": 33.5,
      "speed_mps": 3.0, "gimbal_pitch": -90.0,
      "stay_ms": 0, "actions": ["START_RECORD"] }
  ]
}
```

---

## 9. HA Integration Design

### 9.1 Entities

#### Sensors

| Entity ID | Device Class | Unit | Source |
|-----------|-------------|------|--------|
| `sensor.{name}_battery` | `battery` | `%` | `telemetry/battery → charge_percent` |
| `sensor.{name}_altitude` | `distance` | `m` | `telemetry/flight → alt` |
| `sensor.{name}_ground_speed` | `speed` | `m/s` | `telemetry/flight → ground_speed` |
| `sensor.{name}_gps_satellites` | — | — | `telemetry/flight → satellite_count` |
| `sensor.{name}_signal_uplink` | — | `%` | `telemetry/signal → uplink_quality` |
| `sensor.{name}_signal_downlink` | — | `%` | `telemetry/signal → downlink_quality` |
| `sensor.{name}_battery_temperature` | `temperature` | `°C` | `telemetry/battery → temperature_c` |
| `sensor.{name}_flight_mode` | — | — | `telemetry/flight → flight_mode` |
| `sensor.{name}_heading` | — | `°` | `telemetry/flight → heading` |
| `sensor.{name}_flight_time_remaining` | `duration` | `s` | `telemetry/battery → flight_time_remaining_s` |
| `sensor.{name}_mission_status` | — | — | `state/mission → status` |

#### Binary Sensors

| Entity ID | Device Class | Source |
|-----------|-------------|--------|
| `binary_sensor.{name}_connected` | `connectivity` | `state/connection` |
| `binary_sensor.{name}_airborne` | — | `state/flight ∈ {airborne, returning_home}` |
| `binary_sensor.{name}_motors_on` | — | `telemetry/flight → motors_on` |
| `binary_sensor.{name}_recording` | — | `telemetry/camera → is_recording` |
| `binary_sensor.{name}_streaming` | — | `state/stream → is_streaming` |

#### Camera

| Entity ID | Source |
|-----------|--------|
| `camera.{name}_live` | Media server stream URL |

#### Device Tracker

| Entity ID | Source |
|-----------|--------|
| `device_tracker.{name}` | `telemetry/flight → lat, lon` |

### 9.2 Services

| Service | Description | Fields |
|---------|-------------|--------|
| `dji_hass.execute_mission` | Upload and execute waypoint mission | `mission_id` (required) |
| `dji_hass.return_to_home` | Command RTH | — |
| `dji_hass.takeoff` | Take off and hover | — |
| `dji_hass.land` | Land at current position | — |
| `dji_hass.take_photo` | Capture single photo | — |
| `dji_hass.start_recording` | Begin video recording | — |
| `dji_hass.stop_recording` | Stop video recording | — |
| `dji_hass.start_stream` | Start RTMP live stream | — |
| `dji_hass.stop_stream` | Stop live stream | — |
| `dji_hass.set_gimbal` | Set gimbal pitch angle | `pitch` (-90 to +30) |
| `dji_hass.pause_mission` | Pause executing mission | — |
| `dji_hass.resume_mission` | Resume paused mission | — |
| `dji_hass.stop_mission` | Abort mission (hover) | — |

### 9.3 Coordinator Design

The coordinator is **MQTT-subscription based** (not polling):

```python
class DjiMqttCoordinator:
    """Subscribes to MQTT topics and maintains drone state."""

    def __init__(self, hass, entry, mqtt_client):
        self.data = {
            "connection": "offline",
            "flight": {}, "battery": {}, "gimbal": {},
            "camera": {}, "signal": {}, "mission": {}, "stream": {},
        }
        self._listeners = []
        self._mqtt = mqtt_client
        self._last_heartbeat = None

    async def async_start(self):
        """Subscribe to all drone topics."""
        await self._mqtt.subscribe(f"dji_hass/{self._drone_id}/#")

    async def _on_message(self, topic, payload):
        """Route MQTT messages to data buckets, notify entity listeners."""

    async def async_send_command(self, action, params=None):
        """Publish command, wait for correlation-ID response (10s timeout)."""
```

### 9.4 Config Flow

```
Step 1: Connection
  ├── MQTT Broker Host (default: core-mosquitto)
  ├── MQTT Port (default: 1883)
  ├── MQTT Username / Password
  └── Drone ID (auto-discovered from dji_hass/+/state/connection)

Step 2: Media Server (optional)
  ├── Media Server Type (go2rtc / mediamtx / none)
  └── RTMP Ingest URL

Step 3: Validation
  ├── Test MQTT connection
  ├── Check bridge heartbeat
  └── Verify media server reachability
```

### 9.5 Alarm-Triggered Automation (with Human Authorization)

```yaml
automation:
  - alias: "Drone Security Patrol on Alarm"
    trigger:
      - platform: state
        entity_id: alarm_control_panel.home
        to: "triggered"
    condition:
      # Safety gates
      - condition: state
        entity_id: binary_sensor.mavic2_connected
        state: "on"
      - condition: numeric_state
        entity_id: sensor.mavic2_battery
        above: 30
      - condition: state
        entity_id: binary_sensor.mavic2_airborne
        state: "off"
      - condition: state
        entity_id: binary_sensor.dock_smoke
        state: "off"
      - condition: numeric_state
        entity_id: sensor.dock_temperature
        above: 5
        below: 40
      # Add weather integration checks (wind, rain) here
    action:
      # Open dock lid
      - service: cover.open_cover
        entity_id: cover.drone_dock_lid
      - wait_for_trigger:
          - platform: state
            entity_id: binary_sensor.dock_lid_open
            to: "on"
        timeout: "00:00:30"

      # Actionable notification to RPIC (THE LEGAL COMPLIANCE STEP)
      - service: notify.mobile_app_pilot_phone
        data:
          title: "Perimeter Alert"
          message: >
            Alarm triggered. Battery {{ states('sensor.mavic2_battery') }}%.
            Wind OK. Dock open. Mission: full_perimeter.
          data:
            actions:
              - action: "LAUNCH_DRONE"
                title: "LAUNCH DRONE"
              - action: "IGNORE"
                title: "Ignore"

      # Wait for pilot authorization (max 2 minutes)
      - wait_for_trigger:
          - platform: event
            event_type: mobile_app_notification_action
            event_data:
              action: "LAUNCH_DRONE"
        timeout: "00:02:00"
        continue_on_timeout: false

      # Pilot authorized — execute
      - service: dji_hass.start_stream
      - service: dji_hass.execute_mission
        data:
          mission_id: "full_perimeter"

      # Notify with live feed
      - service: notify.mobile_app_pilot_phone
        data:
          title: "Drone Airborne"
          message: "Executing full perimeter sweep."
          data:
            image: "/api/camera_proxy/camera.mavic2_live"

      # Wait for mission completion
      - wait_for_trigger:
          - platform: state
            entity_id: sensor.mavic2_mission_status
            to: "completed"
        timeout: "00:10:00"

      # Cleanup
      - service: dji_hass.stop_stream
      - delay: "00:00:30"
      - service: cover.close_cover
        entity_id: cover.drone_dock_lid
```

---

## 10. Feature Feasibility Matrix

| Proposed Feature | Feasible? | How | Limitations |
|------------------|-----------|-----|-------------|
| Alarm-triggered mission | **Yes** | HA automation + RPIC tap | Requires human authorization (Part 107) |
| Execute predetermined mission | **Yes** | Waypoint missions via MSDK v4 | Max 99 waypoints |
| Stream video from drone | **Yes** | RTMP via LiveStreamManager → media server | 720p live (not 4K), RTMP only |
| Record video | **Yes** | Camera API | 4K on SD card |
| Take snapshots | **Yes** | Camera API | Full resolution RAW+JPEG |
| Get flight status | **Yes** | FlightControllerState at 10 Hz | GPS, altitude, speed, heading, etc. |
| Get battery level | **Yes** | Battery callback | %, voltage, current, temp, cycles |
| Return to base | **Yes** | FlightController.startGoHome() | Configurable RTH altitude |
| Notify with video | **Yes** | Camera entity + notification | Snapshot or stream URL |
| Physical dock | **Yes** | Custom NEMA enclosure + ESPHome | Manual battery rotation still required |
| Auto battery charging | **No** | Mavic 2 cannot charge in-aircraft | External charger on smart outlet only |
| Fully autonomous launch | **No** | FAA Part 107 prohibits it | Human-in-the-loop required |
| Server-side control | **No** | No Linux SDK for Mavic 2 | Android bridge required |

---

## 11. Failure Modes and Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Bridge loses WiFi | MQTT LWT → "offline" | HA marks unavailable. Bridge auto-reconnects. DJI failsafe handles drone. |
| Bridge app crash | MQTT LWT → "offline" | Android auto-restarts foreground service. |
| RC disconnects from drone | SDK callback | Bridge publishes "offline". Drone auto-RTH (DJI failsafe). |
| RC disconnects from Android | SDK callback | Bridge publishes "offline". Drone RTH if in mission. |
| MQTT broker down | Bridge reconnect loop | Commands blocked. Drone unaffected. |
| Media server down | Stream error callback | Stream stops. Flight unaffected. |
| Mission upload fails | SDK error → MQTT response | HA shows error. Drone stays on ground. |
| Low battery during mission | DJI auto-RTH at critical % | DJI firmware handles safety. |
| GPS loss during mission | DJI → ATTI mode | Bridge publishes mode change. DJI firmware may interrupt mission. |
| Dock lid actuator stuck | Motion timeout on ESP32 | Controller flags fault. HA alerts pilot. |
| Dock smoke sensor triggered | ESP32 hardware interlock | Charger power cut immediately (hardware relay). HA alerts. |
| Power outage | UPS → sensor reports | Dock remains closed. System unavailable until power restored. |
| Pilot doesn't authorize in time | 2-minute timeout in automation | Mission not launched. Event logged. |

---

## 12. Security Considerations

| Concern | Mitigation |
|---------|------------|
| MQTT without TLS | Use MQTT over TLS (port 8883) in production |
| Unauthorized drone commands | MQTT authentication. Bridge validates command source. |
| Physical access to bridge device | Secure location. Android device encryption. |
| DJI SDK credentials | Android Keystore, not in MQTT messages |
| Drone flyaway | Bridge enforces geofence before mission upload. RTH on signal loss (DJI default). |
| Battery safety | Bridge refuses mission below configurable threshold (default 30%). Dock smoke sensor. |
| Network isolation | Dedicated VLAN/WiFi for bridge + HA |
| Privacy (neighbors) | Missions avoid neighbor airspace. Camera angle constrained. |
| Audit trail | HA logs: trigger, authorization timestamp, mission, outcome |

---

## 13. Platform Migration Strategy

### 13.1 The Mavic 2 Problem

The Mavic 2 is an aging platform (2018). Batteries degrade and will become harder to procure. Third-party batteries are the highest failure item in unattended deployments.

**Mavic 2 is explicitly Phase-0 hardware.** It validates:
- Mission geometry and corridor safety
- HA integration and automation workflow
- Dock mechanics and environmental control
- Operational discipline and battery management
- Response time and VLOS reliability

### 13.2 Aircraft-Agnostic Dock Design

The dock is designed around **interfaces, not the drone**:

| Abstraction | What Changes on Upgrade |
|-------------|------------------------|
| Landing pad geometry | Adjustable alignment guides |
| Battery charging | Swap charger on smart outlet |
| MQTT topics | Same protocol, new `drone_id` |
| Mission definitions | Same JSON format, new waypoints |
| Android bridge app | Rebuild with new DJI SDK version |
| HA integration | Unchanged (MQTT abstraction) |

### 13.3 Future Platform Candidates

| Platform | Strengths | Notes |
|----------|-----------|-------|
| DJI Air 3/3S | Current production, long flight time (~40 min), good obstacle sensing, available batteries | Consumer firmware constraints, no dock ecosystem |
| DJI Mavic 3 Enterprise | Enterprise lifecycle, supported SDK, future dock compatibility | Significantly higher cost |
| DJI Dock 2 ecosystem | Solves all hardware problems | $15k-30k total cost, likely overkill for 1-acre residential |

### 13.4 Decision Threshold

Consider upgrading to an enterprise system only if you need:
- Zero human battery handling
- Multiple flights per hour
- Unattended overnight patrol loops
- BVLOS waiver pathway

Otherwise the DIY system achieves ~85% of enterprise capability at ~15-20% of the cost.

---

## 14. Implementation Plan

### Phase 0: Site Survey + Mission Geometry (1 day)
1. Map obstacles: tallest trees, canopy extents, no-fly wedges
2. Identify clear takeoff/landing cylinder above shed roof
3. Verify clear approach lane (no branches in RTH direction)
4. Define 2-3 mission corridors that stay visible from pilot position
5. Fly test missions manually, validate altitude/speed/camera angle
6. **Deliverable:** Property mission map + dock location decision

### Phase 1: Minimal Dock (2 weekends)
1. Source NEMA 4X enclosure + lid material
2. Install ESPHome ESP32 with temp/humidity/smoke sensors
3. Install linear actuator + limit switches
4. Wire power (UPS + fused rails)
5. Configure ESPHome entities in HA
6. Test lid open/close cycle, interlock logic
7. **Deliverable:** Drone can live outdoors safely, lid controlled from HA

### Phase 2: Android Bridge MVP (Weeks 1-3)
1. Android project + DJI Mobile SDK v4 setup
2. SDK registration + product connection handling
3. MQTT client (Paho) + LWT + telemetry publishing
4. Basic commands: takeoff, land, RTH
5. Foreground service with auto-restart
6. **Deliverable:** Bridge publishes telemetry, accepts flight commands

### Phase 3: HA Integration MVP (Weeks 2-4)
1. `custom_components/dji_hass/` scaffold
2. Config flow (MQTT + drone discovery)
3. MQTT coordinator
4. Sensor + binary sensor entities
5. Service handlers (takeoff, land, RTH)
6. **Deliverable:** Drone telemetry in HA, basic commands work

### Phase 4: Missions (Weeks 4-6)
1. Bridge: JSON → DJIWaypointMission translation
2. Bridge: Upload, execute, pause, resume, stop
3. Bridge: Mission progress reporting
4. HA: Mission services + progress entity
5. Build actual perimeter mission pack for the property
6. **Deliverable:** Waypoint missions from HA

### Phase 5: Video + Camera (Weeks 5-7)
1. Bridge: RTMP push via LiveStreamManager
2. Media server setup (go2rtc or mediamtx)
3. HA: Camera entity
4. Bridge: Camera control (photo, record) + gimbal
5. HA: Camera + gimbal services
6. **Deliverable:** Live 720p video in HA dashboard, photo/video capture

### Phase 6: Full Workflow Integration (Weeks 7-9)
1. Alarm → safety gates → notification → authorization automation
2. Dock integration (open lid before launch, close after land)
3. Battery maintenance automations
4. Audit logging
5. Multi-drone support (Zoom primary, Pro secondary)
6. **Deliverable:** End-to-end alarm response workflow

### Phase 7: Hardening (Weeks 9-11)
1. Edge case handling (all failure modes in Section 11)
2. Battery safety thresholds in bridge
3. Geofence validation before mission upload
4. MQTT TLS
5. HA integration tests (pytest)
6. Operational rehearsals (day/night, wind, rain)
7. **Deliverable:** System you trust at 3 AM in winter rain

---

## 15. Cost Estimate

### DIY System (this design)

| Component | Estimated Cost |
|-----------|---------------|
| Weatherproof enclosure + lid | $300-900 |
| Heating + ventilation | $150-400 |
| Sensors (temp, humidity, smoke, rain) | $150-300 |
| Smart power + UPS | $300-800 |
| Lid actuator + mechanism | $200-600 |
| Landing pad + alignment | $100-300 |
| Spare batteries (6-8 total, 3 per drone) | $800-1,600 |
| ESP32 + wiring + misc | $100-200 |
| Android device (old phone) | $0-150 |
| Anti-collision strobe | $30-80 |
| Misc fabrication | $300-800 |
| **Total** | **$2,500-6,000** |

### vs. DJI Dock 2 Enterprise

| Item | Cost |
|------|------|
| DJI Dock 2 + Matrice 3TD bundle | $15,000-17,000 |
| Installation + mounting | $1,000-5,000 |
| Enterprise batteries | $300-700 each |
| LTE / networking | $10-40/month |
| FlightHub subscription | Variable |
| **Total (3-year)** | **$20,000-40,000** |

---

## 16. Open Questions

1. **Multi-drone operations:** Two drones, one dock, or two docks? Rotation strategy?
2. **SD card media retrieval:** Automate downloading 4K footage from SD to HA/NAS? (MSDK v4 MediaManager supports it, but adds complexity.)
3. **Mission editor:** Define missions via HA UI (map), YAML, or import from DJI tools?
4. **Virtual stick:** Expose low-level control as HA service? Powerful but dangerous.
5. **Remote ID module:** Which module for Mavic 2? Firmware-enabled or external?
6. **Weather integration:** Which HA weather integration for wind/rain gating? Local anemometer vs. API?

---

## Sources

- [DJI Mobile SDK Introduction](https://developer.dji.com/mobile-sdk/documentation/introduction/mobile_sdk_introduction.html)
- [DJI Mobile SDK Missions Guide](https://developer.dji.com/mobile-sdk/documentation/introduction/component-guide-missions.html)
- [DJI WaypointMission API Reference](https://developer.dji.com/api-reference/android-api/Components/Missions/DJIWaypointMission.html)
- [DJI Cloud API Product Support](https://developer.dji.com/doc/cloud-api-tutorial/en/overview/product-support.html)
- [DJI Cloud API MQTT](https://developer.dji.com/doc/cloud-api-tutorial/en/overview/basic-concept/mqtt.html)
- [DJI Windows SDK](https://developer.dji.com/windows-sdk/)
- [RosettaDrone (GitHub)](https://github.com/RosettaDrone/rosettadrone)
- [14 CFR Part 107](https://www.ecfr.gov/current/title-14/chapter-I/subchapter-F/part-107)

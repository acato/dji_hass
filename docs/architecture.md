# dji_hass System Architecture

> **Role:** ARCHITECT
> **Date:** 2026-02-19
> **Status:** Proposal
> **Version:** 0.2.0

## 1. Executive Summary

This document designs a Home Assistant integration for DJI Mavic 2 Pro/Zoom drones. The core challenge is that **no server-side/Linux SDK exists for Mavic 2** — all viable SDKs (Mobile SDK v4, Windows SDK) require a physical device connected via USB to the DJI Remote Controller.

The architecture solves this with a **two-tier bridge pattern**: an Android app running DJI Mobile SDK v4 acts as a protocol bridge, exposing drone capabilities over MQTT and RTMP to a Home Assistant custom integration.

---

## 2. DJI SDK Reality Check

### What's Available for Mavic 2

| SDK | Platform | Mavic 2 Support | Server-Side? |
|-----|----------|-----------------|--------------|
| Mobile SDK v4 | Android, iOS | **Yes** | No — requires physical device + RC USB |
| Mobile SDK v5 | Android, iOS | **No** (Mavic 3+ only) | No |
| Windows SDK | Windows 10+ UWP | **Yes** | Partial — needs Windows box + RC USB |
| Onboard SDK | Linux (C++) | **No** (Matrice only) | Yes, but not for Mavic 2 |
| Cloud API | REST/MQTT/WS | **No** (Enterprise only) | Yes, but not for Mavic 2 |
| Payload SDK | Embedded | **No** | No |

### What Mobile SDK v4 Actually Provides for Mavic 2

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
| Live Video | `VideoFeeder` + `LiveStreamManager` | H.264 NAL units; built-in RTMP push (720p, 1-5 Mbps) |
| Media Download | `MediaManager` | Download photos/videos from SD card |

### What's NOT Available

- **No RTSP** — RTSP is only in MSDK v5 (which doesn't support Mavic 2)
- **No direct server-side control** — Always requires Android/iOS/Windows intermediary
- **No headless SDK** — DJI SDK requires a UI context on Android (workarounds exist)
- **99-waypoint limit** — Hardware constraint of the flight controller
- **Video feed is 720p/1080p** — Live stream, not the 4K recording resolution

---

## 3. System Architecture

### 3.1 High-Level Overview

```
┌──────────────────────────────────────────────────────────┐
│                    HOME ASSISTANT                         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │          dji_hass Custom Integration             │     │
│  │                                                  │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │     │
│  │  │  Config   │  │Coordinatr│  │   Service     │  │     │
│  │  │  Flow     │  │(MQTT Sub)│  │   Handlers   │  │     │
│  │  └──────────┘  └────┬─────┘  └──────┬───────┘  │     │
│  │                     │               │           │     │
│  │  ┌──────────────────┴───────────────┴────────┐  │     │
│  │  │            MQTT Client (aiomqtt)           │  │     │
│  │  └──────────────────┬────────────────────────┘  │     │
│  │                     │                            │     │
│  │  ┌─────────┐ ┌─────┴────┐ ┌──────────────────┐ │     │
│  │  │ Sensors │ │ Binary   │ │ Camera/Switch     │ │     │
│  │  │ Battery │ │ Sensors  │ │ Record/Snapshot   │ │     │
│  │  │ GPS     │ │ Airborne │ │                   │ │     │
│  │  │ Signal  │ │ Connected│ │                   │ │     │
│  │  └─────────┘ └──────────┘ └──────────────────┘ │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
│  ┌─────────────────────┐                                 │
│  │   MQTT Broker        │  (Mosquitto add-on)            │
│  └──────────┬──────────┘                                 │
│             │                                            │
│  ┌──────────┴──────────┐                                 │
│  │  Media Server        │  (mediamtx / go2rtc add-on)    │
│  │  RTMP in → WebRTC out│                                │
│  └─────────────────────┘                                 │
└─────────────┬──────────────────────────────────────┬─────┘
              │ MQTT (tcp/1883)                      │ RTMP (tcp/1935)
              │ LAN / WiFi                           │ LAN / WiFi
┌─────────────┴──────────────────────────────────────┴─────┐
│              ANDROID BRIDGE APP                           │
│              (dedicated Android device + RC)               │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │               DJI Mobile SDK v4                  │     │
│  │                                                  │     │
│  │  ┌───────────┐ ┌──────────┐ ┌────────────────┐  │     │
│  │  │ Flight    │ │ Camera/  │ │ LiveStream     │  │     │
│  │  │ Controller│ │ Gimbal   │ │ Manager        │  │     │
│  │  │           │ │          │ │ (RTMP push)    │  │     │
│  │  └─────┬─────┘ └────┬────┘ └───────┬────────┘  │     │
│  │        │            │              │            │     │
│  │  ┌─────┴────────────┴──────────────┴─────────┐  │     │
│  │  │           MQTT Service Layer               │  │     │
│  │  │  - Publishes telemetry (10 Hz → 1 Hz)     │  │     │
│  │  │  - Subscribes to command topics            │  │     │
│  │  │  - Request/response via correlation IDs    │  │     │
│  │  └──────────────────┬────────────────────────┘  │     │
│  └─────────────────────┘                            │     │
│                        │ USB                         │     │
│              ┌─────────┴─────────┐                   │     │
│              │  DJI Remote Ctrl  │                   │     │
│              └─────────┬─────────┘                   │     │
│                        │ OcuSync 2.0                 │     │
│              ┌─────────┴─────────┐                   │     │
│              │  Mavic 2 Pro/Zoom │                   │     │
│              └───────────────────┘                   │     │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Component Responsibilities

#### Component 1: Android Bridge App

**Purpose:** Translates between DJI proprietary SDK and standard MQTT/RTMP protocols.

**Responsibilities:**
- Initialize DJI SDK, manage registration and product connection lifecycle
- Subscribe to MQTT command topics, execute DJI SDK calls
- Publish telemetry to MQTT at configurable intervals (default 1 Hz, burst 10 Hz)
- Push live video stream via RTMP to the media server
- Handle SDK errors and publish status/health
- Auto-reconnect on connection loss (both MQTT and DJI SDK)
- Persist mission definitions locally for offline execution

**Technology:** Kotlin, DJI Mobile SDK v4, Eclipse Paho MQTT client

**Runtime:** Dedicated Android device (e.g., old phone, Android mini-PC), always connected to RC via USB, always on WiFi to reach HA network.

#### Component 2: dji_hass HA Integration

**Purpose:** Exposes drone capabilities as native HA entities and services.

**Responsibilities:**
- Config flow for setup (MQTT broker, bridge connection, media server URL)
- MQTT-based coordinator that subscribes to telemetry topics
- Entity platforms: sensors, binary sensors, switches, camera
- Service handlers for mission execution, RTH, camera commands
- Availability tracking based on bridge heartbeat

**Technology:** Python 3.12+, aiomqtt, Home Assistant Core 2024.2+

#### Component 3: MQTT Broker

**Purpose:** Message bus between Android bridge and HA integration.

**Runtime:** Mosquitto (standard HA add-on, already available in most setups)

#### Component 4: Media Server

**Purpose:** Receives RTMP from the bridge, serves streams to HA frontend.

**Runtime:** mediamtx or go2rtc (HA add-on). Receives RTMP, can serve HLS/WebRTC/RTSP to HA's camera entity.

---

## 4. MQTT Topic Design

### 4.1 Topic Namespace

All topics under `dji_hass/{drone_id}/` where `drone_id` is derived from aircraft serial number.

### 4.2 Telemetry (Bridge → HA)

Published by the Android bridge. QoS 0 for high-frequency data, QoS 1 for state changes.

```
dji_hass/{drone_id}/telemetry/flight
  Payload (JSON, 1 Hz):
  {
    "lat": 47.6062,
    "lon": -122.3321,
    "alt": 45.2,              // meters, relative to takeoff
    "alt_msl": 102.3,         // meters above sea level (if available)
    "heading": 127.5,         // degrees
    "speed_x": 2.1,           // m/s north
    "speed_y": -0.5,          // m/s east
    "speed_z": 0.0,           // m/s down
    "ground_speed": 2.16,     // m/s computed
    "flight_mode": "GPS",     // ATTI, GPS, SPORT, etc.
    "motors_on": true,
    "is_flying": true,
    "gps_signal": 5,          // 0-10
    "satellite_count": 14,
    "wind_warning": 0,        // 0=none, 1=moderate, 2=strong
    "timestamp": 1739980800
  }

dji_hass/{drone_id}/telemetry/battery
  Payload (JSON, 0.2 Hz):
  {
    "charge_percent": 78,
    "voltage_mv": 15200,
    "current_ma": -2100,
    "temperature_c": 32,
    "remaining_mah": 2800,
    "full_charge_mah": 3600,
    "discharge_cycles": 42,
    "flight_time_remaining_s": 1200,
    "timestamp": 1739980800
  }

dji_hass/{drone_id}/telemetry/gimbal
  Payload (JSON, 1 Hz):
  {
    "pitch": -45.0,
    "roll": 0.2,
    "yaw": 127.5,
    "mode": "YAW_FOLLOW",
    "timestamp": 1739980800
  }

dji_hass/{drone_id}/telemetry/camera
  Payload (JSON, on change):
  {
    "mode": "RECORD_VIDEO",     // SHOOT_PHOTO, RECORD_VIDEO, PLAYBACK
    "is_recording": true,
    "recording_time_s": 45,
    "is_storing_photo": false,
    "sd_card_remaining_mb": 28500,
    "iso": 100,
    "shutter_speed": "1/500",
    "aperture": 2.8,
    "ev": 0,
    "timestamp": 1739980800
  }

dji_hass/{drone_id}/telemetry/signal
  Payload (JSON, 1 Hz):
  {
    "uplink_quality": 85,      // 0-100%
    "downlink_quality": 92,    // 0-100%
    "timestamp": 1739980800
  }
```

### 4.3 State (Bridge → HA)

Published with QoS 1 and `retain: true`.

```
dji_hass/{drone_id}/state/connection
  Payload: "online" | "offline"
  (Bridge sets LWT to "offline")

dji_hass/{drone_id}/state/flight
  Payload: "landed" | "airborne" | "returning_home" | "landing"

dji_hass/{drone_id}/state/mission
  Payload (JSON):
  {
    "status": "idle",          // idle, uploading, executing, paused, completed, error
    "mission_id": null,
    "progress": 0.0,           // 0.0-1.0
    "current_waypoint": 0,
    "total_waypoints": 0,
    "error": null
  }

dji_hass/{drone_id}/state/stream
  Payload (JSON):
  {
    "is_streaming": true,
    "rtmp_url": "rtmp://192.168.1.10:1935/live/mavic2",
    "resolution": "720p",
    "bitrate_kbps": 3000
  }
```

### 4.4 Commands (HA → Bridge)

Published by HA integration. QoS 1. Bridge subscribes and executes.

All commands follow a request/response pattern using correlation IDs:

```
dji_hass/{drone_id}/command/{action}
  Payload (JSON):
  {
    "id": "uuid-correlation-id",
    "params": { ... }           // action-specific
  }

dji_hass/{drone_id}/command/{action}/response
  Payload (JSON):
  {
    "id": "uuid-correlation-id",
    "success": true,
    "error": null,
    "data": { ... }             // optional response data
  }
```

#### Available Commands

```
# Flight control
dji_hass/{drone_id}/command/takeoff          params: {}
dji_hass/{drone_id}/command/land             params: {}
dji_hass/{drone_id}/command/confirm_landing  params: {}
dji_hass/{drone_id}/command/return_to_home   params: {}
dji_hass/{drone_id}/command/cancel_rth       params: {}

# Mission control
dji_hass/{drone_id}/command/execute_mission  params: {"mission_id": "patrol_1"}
dji_hass/{drone_id}/command/pause_mission    params: {}
dji_hass/{drone_id}/command/resume_mission   params: {}
dji_hass/{drone_id}/command/stop_mission     params: {}

# Camera
dji_hass/{drone_id}/command/take_photo       params: {}
dji_hass/{drone_id}/command/start_recording  params: {}
dji_hass/{drone_id}/command/stop_recording   params: {}
dji_hass/{drone_id}/command/set_camera_mode  params: {"mode": "SHOOT_PHOTO"}

# Gimbal
dji_hass/{drone_id}/command/set_gimbal       params: {"pitch": -45, "mode": "ABSOLUTE_ANGLE"}
dji_hass/{drone_id}/command/reset_gimbal     params: {}

# Stream
dji_hass/{drone_id}/command/start_stream     params: {"rtmp_url": "rtmp://..."}
dji_hass/{drone_id}/command/stop_stream      params: {}

# System
dji_hass/{drone_id}/command/set_home         params: {"lat": 47.6, "lon": -122.3}
```

### 4.5 Mission Definitions (HA → Bridge)

Missions are uploaded as retained MQTT messages so the bridge can cache them:

```
dji_hass/{drone_id}/missions/{mission_id}
  Payload (JSON, retained):
  {
    "id": "patrol_1",
    "name": "Front Yard Patrol",
    "speed_mps": 5.0,
    "finish_action": "GO_HOME",       // GO_HOME, AUTO_LAND, NO_ACTION
    "heading_mode": "AUTO",
    "flight_path_mode": "CURVED",
    "waypoints": [
      {
        "lat": 47.6062,
        "lon": -122.3321,
        "alt": 30.0,
        "speed_mps": 5.0,
        "gimbal_pitch": -45.0,
        "stay_ms": 2000,
        "actions": ["START_TAKE_PHOTO"]
      },
      {
        "lat": 47.6065,
        "lon": -122.3318,
        "alt": 25.0,
        "speed_mps": 3.0,
        "gimbal_pitch": -90.0,
        "stay_ms": 0,
        "actions": ["START_RECORD"]
      }
    ]
  }
```

---

## 5. HA Integration Design

### 5.1 Entities

#### Sensors

| Entity ID | Device Class | Unit | Source |
|-----------|-------------|------|--------|
| `sensor.mavic2_battery` | `battery` | `%` | `telemetry/battery → charge_percent` |
| `sensor.mavic2_altitude` | `distance` | `m` | `telemetry/flight → alt` |
| `sensor.mavic2_ground_speed` | `speed` | `m/s` | `telemetry/flight → ground_speed` |
| `sensor.mavic2_gps_satellites` | — | — | `telemetry/flight → satellite_count` |
| `sensor.mavic2_signal_uplink` | — | `%` | `telemetry/signal → uplink_quality` |
| `sensor.mavic2_signal_downlink` | — | `%` | `telemetry/signal → downlink_quality` |
| `sensor.mavic2_battery_temperature` | `temperature` | `°C` | `telemetry/battery → temperature_c` |
| `sensor.mavic2_flight_mode` | — | — | `telemetry/flight → flight_mode` |
| `sensor.mavic2_heading` | — | `°` | `telemetry/flight → heading` |
| `sensor.mavic2_flight_time_remaining` | `duration` | `s` | `telemetry/battery → flight_time_remaining_s` |
| `sensor.mavic2_mission_status` | — | — | `state/mission → status` |

#### Binary Sensors

| Entity ID | Device Class | Source |
|-----------|-------------|--------|
| `binary_sensor.mavic2_connected` | `connectivity` | `state/connection` |
| `binary_sensor.mavic2_airborne` | — | `state/flight ∈ {airborne, returning_home}` |
| `binary_sensor.mavic2_motors_on` | — | `telemetry/flight → motors_on` |
| `binary_sensor.mavic2_recording` | — | `telemetry/camera → is_recording` |
| `binary_sensor.mavic2_streaming` | — | `state/stream → is_streaming` |

#### Camera

| Entity ID | Source |
|-----------|--------|
| `camera.mavic2_live` | Media server URL from `state/stream → rtmp_url` via go2rtc/mediamtx |

#### Device Tracker

| Entity ID | Source |
|-----------|--------|
| `device_tracker.mavic2` | `telemetry/flight → lat, lon` |

### 5.2 Services

```yaml
# services.yaml

execute_mission:
  name: Execute Mission
  description: Upload and execute a predefined waypoint mission
  fields:
    mission_id:
      name: Mission ID
      description: Identifier of the mission to execute
      required: true
      example: "patrol_1"
      selector:
        text:

return_to_home:
  name: Return to Home
  description: Command the drone to return to its home point

takeoff:
  name: Takeoff
  description: Command the drone to take off and hover

land:
  name: Land
  description: Command the drone to land at current position

take_photo:
  name: Take Photo
  description: Capture a single photo

start_recording:
  name: Start Recording
  description: Begin video recording on the drone camera

stop_recording:
  name: Stop Recording
  description: Stop video recording

start_stream:
  name: Start Live Stream
  description: Start RTMP live video stream to the media server

stop_stream:
  name: Stop Live Stream
  description: Stop RTMP live video stream

set_gimbal:
  name: Set Gimbal Angle
  description: Set the gimbal pitch angle
  fields:
    pitch:
      name: Pitch
      description: Gimbal pitch angle in degrees (-90 to +30)
      required: true
      selector:
        number:
          min: -90
          max: 30
          step: 1
          unit_of_measurement: "°"

pause_mission:
  name: Pause Mission
  description: Pause the currently executing mission (drone hovers in place)

resume_mission:
  name: Resume Mission
  description: Resume a paused mission

stop_mission:
  name: Stop Mission
  description: Abort the current mission (drone hovers in place)
```

### 5.3 Coordinator Design

Unlike a typical polling coordinator, dji_hass uses an **MQTT-subscription coordinator** — it doesn't poll, it receives pushed data.

```python
class DjiMqttCoordinator:
    """Subscribes to MQTT topics and maintains drone state."""

    def __init__(self, hass, entry, mqtt_client):
        self.data = {
            "connection": "offline",
            "flight": {},
            "battery": {},
            "gimbal": {},
            "camera": {},
            "signal": {},
            "mission": {},
            "stream": {},
        }
        self._listeners = []      # HA entity update callbacks
        self._mqtt = mqtt_client
        self._last_heartbeat = None

    async def async_start(self):
        """Subscribe to all telemetry and state topics."""
        await self._mqtt.subscribe(f"dji_hass/{self._drone_id}/#")
        # Start heartbeat monitor (mark unavailable if no message in 30s)

    async def _on_message(self, topic, payload):
        """Route incoming MQTT messages to the correct data bucket."""
        # Parse topic, update self.data, notify listeners
        # Entities read from self.data — they are stateless views

    async def async_send_command(self, action, params=None):
        """Publish a command and wait for response (with timeout)."""
        correlation_id = str(uuid4())
        # Publish to command topic
        # Wait for response on command/{action}/response with matching ID
        # Timeout after 10s → raise error
```

### 5.4 Config Flow

```
Step 1: Connection Setup
  ├── MQTT Broker Host (default: core-mosquitto)
  ├── MQTT Broker Port (default: 1883)
  ├── MQTT Username
  ├── MQTT Password
  └── Drone ID (auto-discovered from dji_hass/+/state/connection)

Step 2: Media Server (optional)
  ├── Media Server Type (mediamtx / go2rtc / none)
  └── RTMP Ingest URL (default: rtmp://localhost:1935/live/mavic2)

Step 3: Validation
  ├── Test MQTT connection
  ├── Check for bridge heartbeat on drone topics
  └── Verify media server reachability
```

---

## 6. Android Bridge App Design

### 6.1 Architecture

```
┌─────────────────────────────────────────────────────┐
│                 Android Bridge App                    │
│                                                      │
│  ┌────────────────┐    ┌────────────────────────┐   │
│  │  Foreground     │    │   DJI SDK Manager       │   │
│  │  Service        │    │   - Registration         │   │
│  │  (always-on)    │    │   - Product connection   │   │
│  │                 │    │   - Component access      │   │
│  └───────┬─────────┘    └───────────┬────────────┘   │
│          │                          │                 │
│  ┌───────┴──────────────────────────┴────────────┐   │
│  │              Command Router                    │   │
│  │  MQTT subscribe → parse → SDK call → respond  │   │
│  └───────┬──────────────────────────┬────────────┘   │
│          │                          │                 │
│  ┌───────┴─────────┐    ┌──────────┴────────────┐   │
│  │ Telemetry        │    │ Mission Manager        │   │
│  │ Publisher         │    │ - Persist definitions  │   │
│  │ - Flight 1Hz     │    │ - Translate JSON →     │   │
│  │ - Battery 0.2Hz  │    │   DJIWaypointMission   │   │
│  │ - Gimbal 1Hz     │    │ - Upload + execute     │   │
│  │ - Camera onChange │    │ - Progress tracking    │   │
│  │ - Signal 1Hz     │    │                        │   │
│  └──────────────────┘    └────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │           Stream Manager                      │    │
│  │  DJI LiveStreamManager → RTMP push            │    │
│  │  - Auto-reconnect on stream failure           │    │
│  │  - Report stream health via MQTT              │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### 6.2 Key Design Decisions

**Foreground Service:** The app runs as an Android Foreground Service with a persistent notification. This prevents the OS from killing it and ensures continuous operation.

**Telemetry Throttling:** DJI SDK fires telemetry at 10 Hz. The bridge downsamples to 1 Hz for MQTT publishing (configurable). A "burst mode" at 10 Hz can be activated during active missions for higher fidelity tracking.

**Mission Translation:** The bridge converts the JSON mission format (from MQTT retained messages) into `DJIMutableWaypointMission` objects. It validates against SDK constraints (99 waypoints, altitude limits) before upload.

**MQTT Last Will:** The bridge sets an MQTT Last Will and Testament (LWT) of `"offline"` on `dji_hass/{drone_id}/state/connection`. On clean connect, it publishes `"online"`.

**Configuration:** The bridge app has a minimal UI for initial setup only:
- MQTT broker address/credentials
- Drone ID
- WiFi network selection
- Then it runs headlessly as a service

### 6.3 Hardware Requirements

| Component | Purpose | Example |
|-----------|---------|---------|
| Android device | Run bridge app | Old phone (Android 7+), Xiaomi Mi Box, etc. |
| USB OTG cable | Connect Android to RC | USB-C OTG or micro-USB OTG |
| DJI Remote Controller | Bridge to aircraft | DJI RC for Mavic 2 |
| WiFi network | Connect Android to HA network | Same LAN as HA |
| Power supply | Keep Android device powered | USB charger (always plugged in) |

---

## 7. Data Flow Diagrams

### 7.1 Mission Execution Flow

```
User/Automation          HA Integration         MQTT Broker         Android Bridge        Mavic 2
      │                       │                      │                     │                  │
      │ service: execute      │                      │                     │                  │
      │ mission "patrol_1"    │                      │                     │                  │
      ├──────────────────────►│                      │                     │                  │
      │                       │ PUB command/         │                     │                  │
      │                       │ execute_mission      │                     │                  │
      │                       ├─────────────────────►│                     │                  │
      │                       │                      │ deliver             │                  │
      │                       │                      ├────────────────────►│                  │
      │                       │                      │                     │ load mission     │
      │                       │                      │                     │ from cache       │
      │                       │                      │                     │                  │
      │                       │                      │                     │ validate         │
      │                       │                      │                     │ (≤99 wps, etc.)  │
      │                       │                      │                     │                  │
      │                       │                      │                     │ SDK: loadMission │
      │                       │                      │                     │ SDK: uploadMission
      │                       │                      │                     ├─────────────────►│
      │                       │                      │                     │                  │
      │                       │                      │                     │ SDK: startMission│
      │                       │                      │                     ├─────────────────►│
      │                       │                      │                     │                  │
      │                       │                      │  PUB response       │                  │
      │                       │                      │◄────────────────────┤                  │
      │                       │◄─────────────────────┤                     │                  │
      │      success          │                      │                     │                  │
      │◄──────────────────────┤                      │                     │                  │
      │                       │                      │                     │                  │
      │                       │                      │ PUB state/mission   │                  │
      │                       │                      │ {status: executing, │                  │
      │                       │                      │  progress: 0.15,    │                  │
      │                       │◄─────────────────────│  waypoint: 2/12}    │                  │
      │  entity updates       │                      │◄────────────────────┤  (ongoing)       │
      │◄──────────────────────┤                      │                     │                  │
```

### 7.2 Video Streaming Flow

```
User/Automation          HA Integration         MQTT          Bridge           Media Server
      │                       │                  │               │                   │
      │ service: start_stream │                  │               │                   │
      ├──────────────────────►│                  │               │                   │
      │                       │ PUB command/     │               │                   │
      │                       │ start_stream     │               │                   │
      │                       ├─────────────────►│               │                   │
      │                       │                  ├──────────────►│                   │
      │                       │                  │               │                   │
      │                       │                  │               │ SDK: setLiveUrl() │
      │                       │                  │               │ SDK: startStream()│
      │                       │                  │               │                   │
      │                       │                  │               │ RTMP H.264 push   │
      │                       │                  │               ├──────────────────►│
      │                       │                  │               │   (continuous)    │
      │                       │                  │               │                   │
      │                       │                  │ PUB state/    │                   │
      │                       │                  │ stream        │                   │
      │                       │◄─────────────────│◄──────────────┤                   │
      │                       │                  │               │                   │
      │                       │ camera entity    │               │                   │
      │                       │ stream_source =  │               │                   │
      │  camera card shows    │ media_server_url │               │                   │
      │  live video           │                  │               │                   │
      │◄──────────────────────┤                  │               │                   │
```

### 7.3 Alarm-Triggered Automation Example

```yaml
# HA Automation: security_drone_patrol.yaml
automation:
  - alias: "Drone Security Patrol on Alarm"
    trigger:
      - platform: state
        entity_id: alarm_control_panel.home
        to: "triggered"
    condition:
      - condition: state
        entity_id: binary_sensor.mavic2_connected
        state: "on"
      - condition: numeric_state
        entity_id: sensor.mavic2_battery
        above: 30
      - condition: state
        entity_id: binary_sensor.mavic2_airborne
        state: "off"
    action:
      - service: dji_hass.start_stream
      - service: dji_hass.execute_mission
        data:
          mission_id: "perimeter_sweep"
      - service: notify.mobile_app
        data:
          title: "Drone Patrol Activated"
          message: "Alarm triggered. Drone executing perimeter sweep."
          data:
            image: "/api/camera_proxy/camera.mavic2_live"
      - wait_for_trigger:
          - platform: state
            entity_id: sensor.mavic2_mission_status
            to: "completed"
        timeout: "00:10:00"
      - service: dji_hass.stop_stream
```

---

## 8. Feature Feasibility Matrix

Based on the original README proposal vs. DJI SDK v4 reality:

| Proposed Feature | Feasible? | Implementation | Limitations |
|------------------|-----------|----------------|-------------|
| Execute predetermined mission | **Yes** | Waypoint missions via MSDK v4 | Max 99 waypoints per mission |
| Stream video from drone | **Yes** | RTMP via LiveStreamManager → media server | 720p live (not 4K), RTMP only (no native RTSP) |
| Record video | **Yes** | Camera API start/stop recording | Records at full resolution (4K) on SD card |
| Take snapshots | **Yes** | Camera API shoot photo | Full resolution, RAW+JPEG supported |
| Get flight status (airborne/landed) | **Yes** | FlightControllerState at 10 Hz | Rich data: mode, GPS, speed, heading, etc. |
| Get battery level | **Yes** | Battery callback with full details | %, voltage, current, temp, cycles, remaining time |
| Return to base | **Yes** | FlightController.startGoHome() | Configurable RTH altitude |
| Trigger on HA alarm | **Yes** | Standard HA automation | Requires bridge online + drone connected + sufficient battery |
| Notify with video | **Yes** | Camera entity + notification service | HA notification with camera snapshot or stream URL |
| Persist video to storage | **Partial** | Recording on SD card (4K). RTMP can be recorded by media server (720p) | SD card download requires separate media retrieval step |
| Cloud storage | **Partial** | Media server can push to cloud. SD card content needs manual or scheduled retrieval | Not real-time for 4K footage |
| Control mission (start/abort) | **Yes** | Start/pause/resume/stop mission commands | Full lifecycle control |
| Server-side (Linux) control | **No** | Requires Android/Windows bridge device | Fundamental SDK limitation |

---

## 9. Alternative Architecture: RosettaDrone (Lower Effort)

If building a custom Android app is too much initial effort, [RosettaDrone](https://github.com/RosettaDrone/rosettadrone) provides an existing MAVLink bridge:

```
[HA Integration]  ←UDP/MAVLink→  [RosettaDrone App]  ←USB→  [RC]  ←OcuSync→  [Mavic 2]
                                      ↓
                                 [RTP Video → FFmpeg → RTMP → Media Server]
```

**Pros:**
- Already works with Mavic 2 Pro/Zoom
- MAVLink is well-documented, Python libraries exist (pymavlink)
- Video forwarding via RTP already implemented
- Waypoint missions, telemetry, virtual stick all supported

**Cons:**
- MAVLink adds a translation layer (DJI → MAVLink → HA)
- Less control over telemetry format and timing
- RosettaDrone is community-maintained, may lag DJI SDK updates
- Video path requires extra FFmpeg step for RTMP conversion

**Recommendation:** Start with the custom MQTT bridge (Section 6). It's more work upfront but gives full control over the protocol, eliminates the MAVLink translation layer, and produces a cleaner HA integration. RosettaDrone is a valid fallback if the custom bridge proves too complex.

---

## 10. Security Considerations

| Concern | Mitigation |
|---------|------------|
| MQTT without TLS | Use MQTT over TLS (port 8883) in production. Mosquitto add-on supports TLS. |
| Unauthorized drone commands | MQTT authentication required. Bridge validates command source. |
| Physical access to bridge device | Bridge device should be in a secure location. Android device encryption enabled. |
| DJI SDK credentials | DJI app key stored in Android Keystore, not in MQTT messages. |
| Drone flyaway | Bridge enforces geofence validation before mission upload. RTH on signal loss is DJI default. |
| Battery safety | Bridge refuses mission execution below configurable battery threshold (default 30%). |
| Network isolation | Bridge and HA should be on a dedicated VLAN/WiFi, not guest network. |

---

## 11. Failure Modes and Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Bridge loses WiFi | MQTT LWT triggers "offline" | HA marks drone unavailable. Bridge auto-reconnects. Drone continues mission or RTH per DJI failsafe. |
| Bridge app crashes | MQTT LWT triggers "offline" | Android restarts foreground service (auto-restart). DJI failsafe handles drone. |
| RC disconnects from drone | SDK product disconnect callback | Bridge publishes "offline". Drone auto-RTH (DJI failsafe). |
| RC disconnects from Android | SDK callback | Bridge publishes "offline". Drone continues mission if in progress, then auto-RTH. |
| MQTT broker down | Bridge reconnect loop | Commands blocked. Telemetry buffered briefly then dropped. Drone unaffected. |
| Media server down | Stream error callback | Bridge stops RTMP push. Retries periodically. Drone flight unaffected. |
| Mission upload fails | SDK error callback | Bridge publishes error response. HA shows error. Drone stays on ground. |
| Low battery during mission | DJI auto-RTH at critical battery | Bridge publishes battery warning. DJI firmware handles safety. |
| GPS loss during mission | DJI switches to ATTI mode | Bridge publishes flight_mode change. Mission may be interrupted by DJI firmware. |

---

## 12. Implementation Plan

### Phase 1: Android Bridge MVP (Weeks 1-3)

1. Android project setup with DJI Mobile SDK v4
2. SDK registration and product connection handling
3. MQTT client (Eclipse Paho) with connection management and LWT
4. Telemetry publishing (flight, battery)
5. Basic commands: takeoff, land, RTH
6. Foreground service with auto-restart

**Deliverable:** Bridge that connects to Mavic 2, publishes telemetry, accepts takeoff/land/RTH.

### Phase 2: HA Integration MVP (Weeks 2-4)

1. Custom component scaffold (`custom_components/dji_hass/`)
2. Config flow (MQTT connection + drone discovery)
3. MQTT coordinator (subscribe, parse, maintain state)
4. Sensor entities (battery, altitude, speed, flight mode)
5. Binary sensor entities (connected, airborne)
6. Service handlers (takeoff, land, RTH)

**Deliverable:** Working HA integration showing drone telemetry and accepting basic flight commands.

### Phase 3: Missions (Weeks 4-6)

1. Bridge: Mission definition parsing (JSON → DJIWaypointMission)
2. Bridge: Mission upload, execute, pause, resume, stop
3. Bridge: Mission progress reporting
4. HA: Mission entity, service handlers, progress tracking
5. HA: Mission definition UI or YAML config

**Deliverable:** Waypoint missions can be defined, uploaded, and executed from HA.

### Phase 4: Video (Weeks 5-7)

1. Bridge: LiveStreamManager RTMP push
2. Bridge: Stream start/stop commands, health reporting
3. Media server setup (mediamtx or go2rtc add-on)
4. HA: Camera entity consuming media server stream
5. HA: Stream control services

**Deliverable:** Live 720p video from drone viewable in HA dashboard.

### Phase 5: Camera & Gimbal (Weeks 6-8)

1. Bridge: Camera control (photo, record, settings)
2. Bridge: Gimbal control (pitch, reset)
3. HA: Camera services (take_photo, start/stop_recording)
4. HA: Gimbal service (set_gimbal)
5. Integration with mission waypoint actions

**Deliverable:** Full camera and gimbal control from HA.

### Phase 6: Hardening (Weeks 8-10)

1. Comprehensive error handling and edge cases
2. Battery safety thresholds
3. Geofence validation
4. MQTT TLS
5. HA integration tests (pytest)
6. Bridge integration tests
7. Documentation

**Deliverable:** Production-ready integration with safety checks and test coverage.

---

## 13. Open Questions

1. **ChatGPT conversation:** The shared link (titled "Drone Mission Regulations WA") could not be accessed — it requires ChatGPT authentication. If it contains additional requirements (e.g., Washington state geofencing, specific regulation handling), those should be incorporated.

2. **Multi-drone support:** Should the architecture support multiple drones? Current design supports it (drone_id in topics) but would need multiple bridge devices.

3. **SD card media retrieval:** Should the integration support downloading photos/videos from the drone's SD card to HA storage? This is possible via MSDK v4 MediaManager but adds significant complexity.

4. **Mission editor:** Should missions be defined via HA UI (map-based editor), YAML files, or imported from DJI tools? A map-based editor is a substantial frontend effort.

5. **Virtual stick mode:** Should we expose low-level virtual stick control as an HA service, or restrict to waypoint missions only? Virtual stick is powerful but dangerous without safeguards.

---

## Sources

- [DJI Mobile SDK Introduction](https://developer.dji.com/mobile-sdk/documentation/introduction/mobile_sdk_introduction.html)
- [DJI Mobile SDK Missions Guide](https://developer.dji.com/mobile-sdk/documentation/introduction/component-guide-missions.html)
- [DJI WaypointMission API Reference](https://developer.dji.com/api-reference/android-api/Components/Missions/DJIWaypointMission.html)
- [DJI Cloud API Product Support](https://developer.dji.com/doc/cloud-api-tutorial/en/overview/product-support.html)
- [DJI Cloud API MQTT](https://developer.dji.com/doc/cloud-api-tutorial/en/overview/basic-concept/mqtt.html)
- [DJI Windows SDK](https://developer.dji.com/windows-sdk/)
- [RosettaDrone (GitHub)](https://github.com/RosettaDrone/rosettadrone)
- [DJI Cloud API Supported Devices](https://deepwiki.com/dji-sdk/Cloud-API-Doc/9.1-supported-devices)

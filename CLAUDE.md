# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**dji_hass** is a Home Assistant custom integration for launching DJI drone missions through HA automations. The scenario: on a HA trigger (alarm, person detected), send a drone on a predetermined mission, stream/record video, and monitor flight status.

**Status:** v0.0.1 — conceptual/planning phase, no source code implemented yet.

**Tested hardware:** Mavic 2 Pro, Mavic 2 Zoom only.

## Critical Constraint: DJI SDK

The only DJI SDKs supporting Mavic 2 are Windows and iOS/Android — there is no Linux/server-side SDK. This is the biggest architectural challenge. The integration needs an intermediary approach (Windows VM, mobile relay service, or reverse-engineered protocol).

## Architecture

A server component (ideally an HA add-on) bridges Home Assistant and the DJI drone:
- **HA side:** MQTT for commands/status, RTSP/RTMP for video streaming
- **Drone side:** DJI SDK (via intermediary, see constraint above)

### Planned Service APIs

| Service | Returns |
|---------|---------|
| `GetDroneFlightStatus` | `{airborne\|landed}` |
| `GetDroneBatteryLevel` | `int {0-100}` |
| `ExecuteMission(MissionID)` | void |
| `ReturnToBase` | void |
| `GetRTSPStream(port)` | H264 stream |
| `StartRecording(stream)` | void |
| `StopRecording(stream)` | void |
| `TakeSnapshot` | image |

## Target Stack

- **Python 3.12+**, fully async
- **Home Assistant Core 2024.2+**
- **pytest** for testing (80%+ coverage target)
- **ruff** for linting, **mypy** for type checking

## Target Directory Structure

```
custom_components/dji_hass/
├── __init__.py          # Setup/teardown (async_setup_entry, async_unload_entry)
├── config_flow.py       # Configuration UI
├── coordinator.py       # DataUpdateCoordinator for drone status polling
├── const.py             # Constants (DOMAIN, defaults)
├── manifest.json        # Integration metadata
├── strings.json         # UI strings
├── services.yaml        # Custom service definitions
└── <platform>.py        # Entity platforms (sensor, binary_sensor, switch, etc.)

tests/
├── conftest.py
├── test_init.py
├── test_config_flow.py
├── test_coordinator.py
└── test_<platform>.py
```

## Build & Test Commands

```bash
# Tests
pytest tests/ -v --cov=custom_components.dji_hass

# Type checking
mypy custom_components/dji_hass

# Linting
ruff check custom_components/dji_hass

# Single test file
pytest tests/test_config_flow.py -v
```

## Role-Based Development Guidance

The `.claude/` directory contains detailed role-specific guidance for HA development. Reference these when working in each capacity:

- **`.claude/init`** — Core project context, common patterns, imports (auto-loaded)
- **`.claude/architect.md`** — System design, component architecture, data flow patterns
- **`.claude/developer.md`** — Backend (Python) and frontend (TypeScript) implementation patterns, code templates
- **`.claude/sdet.md`** — Testing strategies, fixtures, coverage patterns
- **`.claude/fuzzer.md`** — Security testing, edge cases, attack vectors
- **`.claude/user_education_specialist.md`** — Documentation templates

Invoke roles with: "As ARCHITECT, ...", "As BACKEND DEVELOPER, ...", "As SDET, ...", "As FUZZER, ..."

## Key HA Integration Patterns

These are the critical patterns from the guidance files — all code should follow them:

1. **Coordinator pattern** — All API polling goes through `DataUpdateCoordinator`. Entities are stateless views into `coordinator.data`.
2. **Error handling layers** — API client raises specific exceptions → Coordinator catches and raises `UpdateFailed` → Entities handle `coordinator.data` being None.
3. **Config flow** — UI-based setup with connection validation before saving. No YAML config.
4. **Entity identity** — Every entity needs a stable `unique_id` and proper `device_info`.
5. **Async throughout** — No blocking I/O in async functions. Use `hass.async_add_executor_job()` if wrapping sync code.
6. **Resource cleanup** — Everything set up in `async_setup_entry` must be torn down in `async_unload_entry`.

## Legal Disclaimer

Drone flying is regulated. Users must comply with FAA (US) or equivalent aviation authority rules. This software is provided AS-IS with no liability for misuse or regulatory violations. Recreational flight requires line of sight. Non-recreational use requires FAA Part 107 license (US).

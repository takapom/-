# SITL External Georesolution Notes

This note records operational findings from running dictionary-free
natural-language place requests against ArduPilot SITL.

## Scope

The external georesolution path can turn a request such as:

```text
Alexander Maconochie Centreまで行って
```

into:

```text
natural language
  -> georesolver
  -> latitude/longitude
  -> SITL-home-relative local NED
  -> Mission IR
  -> Validator
  -> MAVLink commands
```

This proves the translation path can work without `local_places.json`, but it
does not guarantee the SITL vehicle is in a flyable state or that a long local
NED target is practical for the current simulator setup.

## Observed Run

The georesolver found:

```text
name: Alexander Maconochie Centre
lat: -35.3721063
lon: 149.1699976
source: nominatim
category: highway
type: bus_stop
```

Using SITL home:

```text
home_lat: -35.363261
home_lon: 149.165230
```

the local NED target became:

```text
north_m: -984.65
east_m:   432.78
altitude_m: 1.5
```

The horizontal distance is about 1.08 km. With normal harness constraints, the
mission was rejected because the target was outside the default local geofence.
With `--clear-harness-constraints`, validation passed and commands were emitted.

## Failure Mode

The first live attempt sent:

```text
arm -> takeoff -> goto_local_ned
```

with a 180 second `goto_local_ned` timeout. The vehicle moved partway:

```text
north_m: -330.19
east_m:   148.46
altitude_m: ~1.69
```

but did not reach the target before timeout.

After that, `LAND` did not complete. A force-disarm was sent to stop the SITL
state. Telemetry later showed:

```text
armed: false
mode: GUIDED
speed_mps: ~0.02
battery_remaining_pct: 23
```

A later retry appeared to hang because the harness was waiting inside
`goto_local_ned` for the target to be reached. The MAVLink UDP port was held by
that process, so another telemetry client could not bind to the same endpoint.
After terminating the waiting process, telemetry showed the vehicle was
disarmed and not moving.

## Root Causes

The important distinction is:

```text
natural language -> command generation: succeeded
SITL vehicle executing the full long-distance mission: failed
```

The observed execution failure was caused by a combination of:

- The target was about 1.08 km away from the current SITL home.
- `goto_local_ned` waits until the vehicle reaches the local NED target.
- The original 180 second timeout was too short for the observed progress.
- After timeout, the SITL state was left in a poor state.
- `LAND` did not complete from that state.
- A force-disarm stopped the vehicle but left mode/position telemetry confusing.
- A subsequent retry started from a disarmed, stale, low-battery SITL state.

The `--ignore-harness` flag only ignores harness runtime constraints. It does
not bypass ArduPilot arming state, mode state, failsafes, battery state, or
setpoint-following behavior.

## Operational Guidance

For long-distance external-georesolved targets, prefer resetting SITL before
rerunning:

```text
1. Stop the current live harness process if it is waiting inside goto_local_ned.
2. Restart ArduPilot SITL or reset the simulated vehicle state.
3. Confirm QGroundControl shows a sane disarmed/landed state.
4. Confirm battery/failsafe state is not blocking arming.
5. Run a short local mission first, e.g. A地点まで行って.
6. Then run the external-georesolved mission.
```

The harness now mitigates this failure mode with:

```text
Preflight Gate
  -> reject stale/airborne/armed SITL state before command emission
Segmented local NED route
  -> split long goto_local_ned targets into shorter setpoints
Distance-based timeout
  -> derive segment timeout from distance and expected speed
Progress monitor
  -> abort when distance to target stops decreasing
Reset-required failure state
  -> mark failed live runs with next_required_action: sitl_reset
```

If the mission is intentionally long, use one of these approaches:

```text
Option A: keep the current home and use segmented local NED execution
  --ignore-harness --goto-segment-max-distance-m 100

Option B: move/restart SITL home near the target
  This makes the local NED target short and the run more reproducible.

Option C: split the long target into intermediate local NED waypoints
  This is now the default live-execution strategy for long goto_local_ned
  commands.
```

Avoid using an effectively infinite timeout during debugging. The current
implementation still accepts explicit timeout overrides, but default live runs
use distance-based segment timeouts plus progress-stall detection.

## Useful Commands

Dry-run external resolution and command emission:

```sh
PYTHONPATH=src .venv/bin/python -m drone_ai_harness.runner \
  --nl "Alexander Maconochie Centreまで行って" \
  --allow-external-geo \
  --home-lat -35.363261 \
  --home-lon 149.165230 \
  --geo-cache .cache/geo.json \
  --clear-harness-constraints \
  --dry-run \
  --json
```

Temporary SITL-only run that ignores harness runtime constraints:

```sh
PYTHONPATH=src .venv/bin/python -m drone_ai_harness.runner \
  --nl "Alexander Maconochie Centreまで行って" \
  --allow-external-geo \
  --home-lat -35.363261 \
  --home-lon 149.165230 \
  --geo-cache .cache/geo.json \
  --ignore-harness \
  --goto-segment-max-distance-m 100 \
  --progress-stall-timeout-s 30 \
  --execute \
  --sitl-endpoint udp:127.0.0.1:14552 \
  --json
```

Telemetry check:

```sh
PYTHONPATH=src .venv/bin/python - <<'PY'
import json
from pymavlink import mavutil
from drone_ai_harness.mavlink_adapter import MavlinkAdapter

adapter = MavlinkAdapter("udp:127.0.0.1:14552", heartbeat_timeout_s=30)
adapter.connect()
heartbeat = adapter._recv_match("HEARTBEAT", 5.0)
base_mode = int(getattr(heartbeat, "base_mode", 0)) if heartbeat is not None else None
armed = bool(base_mode & mavutil.mavlink.MAV_MODE_FLAG_SAFETY_ARMED) if base_mode is not None else None
sample = adapter.sample_telemetry(timeout_s=2.0)
adapter.close()
print(json.dumps({"armed": armed, "base_mode": base_mode, "sample": sample}, ensure_ascii=False))
PY
```

## Research Takeaway

External georesolution expands what natural language can refer to, but it adds a
new boundary between semantic success and execution success:

```text
Resolved place correctly
  does not imply
SITL can complete the mission from the current home/state
```

Evaluation logs should therefore separate:

- georesolution result
- local NED target distance
- validator result
- command emission result
- SITL execution result
- vehicle state before and after execution

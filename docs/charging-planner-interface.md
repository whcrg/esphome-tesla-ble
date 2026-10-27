# Interface for external charging planners

This is the contract between the Tesla BLE schedules package ([ha/tesla_ble_schedules.yaml](./ha/tesla_ble_schedules.yaml)) and a separate charging planner or optimiser running in the same Home Assistant (for example EV Smart Charging plus your own planner automations). The user-facing description is in [preconditioning.md](./preconditioning.md).

Entity IDs assume the ESPHome device name `tesla-ble`. With another name, `tesla_ble_` in action and entity IDs changes accordingly; the package's own entities (`sensor.tesla_*`, `input_select.tesla_*`) keep their names.

## 1. Roles

| | Owns | Never does |
| --- | --- | --- |
| **Schedule package** | The car's precondition schedules, the user's charge requests attached to them (window and target), and, only in *Car charge schedules* mode, car charge schedules and the car's charge limit | In *External planner* mode: create car charge schedules at Home, or write the charge limit |
| **Planner** | When and how fast to charge, the charger, and in *External planner* mode the car's charge limit and charging amps | Create car charge schedules (they conflict, see 5); edit the package's helpers |

The mode is `input_select.tesla_charge_scheduling`: `Car charge schedules` or `External planner`. A planner should check it is `External planner` and otherwise stay passive, or warn.

## 2. Primary input: `sensor.tesla_next_departure`

A trigger-based template sensor. It updates when the car's precondition list changes, when the mode changes, every 5 minutes, and at Home Assistant start.

**State:** the next enabled departure within the **Home** zone in the next 72 hours, as an ISO 8601 timestamp (`device_class: timestamp`), or `unknown` if there is none.

**Attributes:**

| Attribute | Type | Meaning |
| --- | --- | --- |
| `all_upcoming` | list of objects | Every enabled departure in the next 72 h, at any location, sorted by `at` (see below) |
| `charge_scheduling` | string | Current mode, same as `input_select.tesla_charge_scheduling` |

**Each `all_upcoming` entry:**

| Key | Type | Meaning |
| --- | --- | --- |
| `at` | ISO datetime (local offset) | When the car should be ready (the precondition time) |
| `schedule_id` | int | ID of the car's precondition schedule. It is the creation time as a Unix timestamp, but treat it as an opaque ID |
| `name` | string | The schedule's name; `""` if none (the car does not seem to keep names) |
| `one_time` | bool | `true` if the schedule runs once |
| `location` | string | Zone entity ID the schedule belongs to (for example `zone.home`), or `other` |
| `latitude`, `longitude` | float | The schedule's location |
| `charge` | bool | `true` if the user asked for charging for this departure |
| `charge_start` | ISO datetime or `null` | Start of the requested charge window (only when `charge`) |
| `charge_end` | ISO datetime or `null` | End of the requested charge window; normally equal to `at` |
| `charge_target` | int (%) or `null` | Requested state of charge; `null` if the request has no target |
| `pending` | bool | `true` while the schedule is only queued in Home Assistant (the car was out of reach when it was saved); it is sent to the car when the car is next in range |

**Guarantees:**
- Disabled schedules are left out.
- A **one-time** schedule appears only for its single run: the first ready time after it was created or last saved from Home Assistant. The car keeps one-time schedules listed (and enabled) after they have run; the sensor drops them once that run is past. A planner should not parse one-time schedules itself.
- A **weekly** schedule appears for each matching day in the window (today and the next three days, cut at 72 h).
- `charge_start` ≤ `charge_end` ≤ `at`; a window crossing midnight is resolved to the right dates.
- Schedules the car reported, with changes still queued for the car applied (`pending: true`): the car's part is as fresh as `sensor.tesla_precondition_schedules` → `read_at` (below). A queued delete already removes the departure.

**Typical use:** the earliest entry with `location == 'zone.home'` and `charge == true` is the next charging deadline (`charge_end`) and target (`charge_target`). The planner may ignore `charge_start` (it is the user's preference, not a constraint) and may use its own target instead.

## 3. Freshness

`sensor.tesla_precondition_schedules` and `sensor.tesla_charge_schedules` carry `read_at` (ISO datetime): when the car's list last arrived. Their `last_changed`/`last_reported` also move on Home Assistant restarts and reloads, so they are **not** read times. The package re-reads the lists whenever the car is awake and in BLE range and `read_at` is over an hour old; it never wakes the car for this. A planner that needs certainty can treat data older than about 6 hours as stale (the example dashboard warns at that age).

## 4. Event: `tesla_ble_departure_changed`

Fired by `script.tesla_set_precondition_schedule` and `script.tesla_delete_schedule` (which the schedule editor uses) right after a precondition schedule is saved or deleted, before the car's updated list has been read back. When the car can't be reached the change is queued and the event carries `queued: true`; the departure appears in the sensor right away anyway. Use it to replan sooner; `sensor.tesla_next_departure` remains the source of truth and follows within seconds to a minute.

| Field | `change: saved` | `change: deleted` |
| --- | --- | --- |
| `change` | `saved` | `deleted` |
| `schedule_id` | ID (new schedules get theirs from Home Assistant, the creation time) | ID |
| `ready_time` | `HH:MM:SS` | — |
| `days` | list of `Sun`…`Sat` | — |
| `one_time`, `enabled` | bool | — |
| `charge` | bool | — |
| `charge_start`, `charge_end` | `HH:MM` or `null` | — |
| `charge_target` | int or `null` | — |
| `latitude`, `longitude` | float | — |
| `queued` | bool: `true` if the car was out of reach and the change waits in `sensor.tesla_schedule_queue` | — |

Schedules changed outside these scripts (Tesla app, the car's screen, direct ESPHome action calls) do not fire this event; they show up in the sensor after the next read.

## 5. Things the planner must keep in mind

- **No car charge schedules at Home in External planner mode.** Car-side charge schedules (and Tesla's scheduled or off-peak charging) can make the car refuse to start when the charger switches on, or start when the planner doesn't want it. The package removes its own linked ones, refuses new ones at Home and its card warns about any others at Home, but it never deletes schedules it didn't create. Charge schedules at other locations (for example at work) don't interfere: the car only uses a schedule where it was made.
- **Charge limit and amps** are the planner's to write in this mode (`number.tesla_ble_charging_limit`, `number.tesla_ble_charging_amps`). The package doesn't touch them.
- **Waking the car:** call `script.tesla_ensure_awake` before writing to the car. It wakes it if needed, waits up to 60 s and fails if the car doesn't wake, so later steps don't run against a sleeping car. Writing only when the value differs avoids needless wake-ups.
- **Departure location:** the car only runs a schedule when parked at its location, so departures with `location != 'zone.home'` usually matter only when the car is there.

## 6. How the request is stored (for reference)

The car has no field for a charge request (and does not keep precondition schedule names), so requests live in Home Assistant: `sensor.tesla_charge_requests`, attribute `requests`, a mapping from precondition schedule ID (as a string) to `{start: "HH:MM", end: "HH:MM", target: int or null, seen: ISO datetime}`. It is updated from `tesla_ble_departure_changed` and kept across restarts; a request whose schedule has been missing from the car's list for 7 days is dropped. A planner should read `sensor.tesla_next_departure`, which resolves these into dates, rather than this sensor.

## 7. Car charge schedules mode (for completeness)

With `Car charge schedules`, the package keeps a car charge schedule named `pc:<precondition id>` for each departure with a charge request (same days, one-time, enabled and location; charging in the window), and sets the car's charge limit to `charge_target` when the window starts. A planner should not run in this mode; if it does, it should only read.

## Preconditioning and preconditioning schedules

This project lets Home Assistant (HA) **start/stop preconditioning now**, and **view**, **add**, **edit** and **delete** the car's preconditioning (departure) schedules. These are the same schedules you see under *Schedule → Precondition* in the Tesla app. It also supports the older single **scheduled departure** setting.

The commands are the BLE equivalents of the ones in Tesla's [vehicle-command](https://github.com/teslamotors/vehicle-command) (`add-precondition-schedule`, `remove-precondition-schedule`, `scheduled-departure`, `climate-on`).

In the action names below, `tesla_ble` is your device name with `-` replaced by `_` (eg **esphome.tesla_ble_start_preconditioning**).

> [!IMPORTANT]
> Reading preconditioning schedules requires a version of the [TeslaBLE library](https://github.com/PedroKTFC/tesla-ble) that decodes them (the `precondition-schedules` change, which gives the nanopb `PreconditioningScheduleState` a fixed size). With an older library, setting and deleting schedules still work but reading them logs a warning instead of firing the event.

### Start/stop preconditioning now
| Action | Fields | Notes |
| --- | --- | --- |
| `start_preconditioning` | `temperature` (°C, 15-28, `0` keeps the current setting) | Sets driver and passenger temperature, then turns climate on. Wakes the car. |
| `stop_preconditioning` | | Turns climate off. |

The existing **Climate** switch and **Defrost** (max preconditioning) switch also still work.

### Battery warming
Tesla's protocol has no separate "warm the battery" command (neither BLE nor the Fleet API). The car heats the battery itself when it preconditions in the cold: preconditioning schedules, scheduled departure, climate on, and most strongly **Defrost** (max preconditioning). Two binary sensors show what the car is actually doing:
- **Battery heater**: on while the battery heater runs.
- **Preconditioning**: on while the car is preconditioning (cabin and/or battery).

Both come from the climate data that is polled anyway, so they add no extra traffic to the car.

### Preconditioning schedules
Like charging schedules, setting is only allowed when the build-time option `allow_setting_schedules` is non-zero (see [charging_schedules.md](./charging_schedules.md)). Reading and deleting are always allowed.

| Action | Fields |
| --- | --- |
| `set_precondition_schedule` | `schedule_id` (empty = new, an existing ID = edit/replace), `name`, `days_of_week` (bitmask Sun = 1 … Sat = 64), `precondition_time` (minutes after midnight, the time the car should be ready), `one_time`, `enabled`, `latitude`, `longitude` |
| `delete_precondition_schedule` | `schedule_id` |
| `get_precondition_schedules` | |

There is also a **Get precondition schedules** button and a **Delete a precondition schedule** text control (both disabled by default).

After a set or delete the list is read back automatically. The list arrives as the event `esphome.tesla_precondition_schedules_updated` with fields `count`, `schedules` (JSON: `{"precondition_schedules": [...]}`), `version` and `comment`. Each schedule has `id`, `name`, `days`, `*days`, `precondition_time`, `*precondition_time` (HH:MM), `one_time`, `enabled`, `latitude`, `longitude`. Fields starting with `*` are derived.

A schedule only runs when the car is parked at its location. The Tesla app sets that to wherever the car is when the schedule is made, and doesn't show it. The example card and editor show it as the name of the matching Home Assistant zone (within the zone's radius, or 500 m for small zones), or as *Other, 47 km from Home*.

The schedule location matters: the car only runs a schedule when it is parked near that latitude/longitude, so use the location where the car will be (eg `zone.home`).

> [!NOTE]
> A BLE response is limited to 400 bytes, enough for roughly ten schedules of each type. If you have more, the response cannot be decoded.

### Scheduled departure (older single setting)
`set_scheduled_departure` fields: `enabled` (false clears it), `departure_time` (minutes after midnight), `preconditioning_policy` and `off_peak_policy` (0 = off, 1 = all week, 2 = weekdays only), `off_peak_end_time` (minutes after midnight). Cars on recent software use preconditioning schedules instead.

### Home Assistant package and dashboard
Everything needed on the Home Assistant side is in two files you can paste as they are:

| File | What it contains |
| --- | --- |
| [ha/tesla_ble_schedules.yaml](./ha/tesla_ble_schedules.yaml) | A [package](https://www.home-assistant.io/docs/configuration/packages/): the two schedule sensors, the scripts, and the schedule editor (helpers and two automations). Install steps are at the top of the file. |
| [ha/tesla_ble_dashboard.yaml](./ha/tesla_ble_dashboard.yaml) | A summary card for your car view and an editor subview. Only built-in cards are used. |

Both assume the ESPHome device name `tesla-ble` (entities `*.tesla_ble_*`, actions `esphome.tesla_ble_*`). If yours differs, search-and-replace `tesla_ble_` in both files.

#### Sensors
- **Tesla charge schedules** / **Tesla precondition schedules**: the state is the number of schedules; the `schedules` attribute holds the list exactly as the car reports it (see the event fields above). They update whenever the lists are read: automatically after every add, edit or delete, and by the *refresh when the car is awake* automation whenever the car is awake and in BLE range and the last read is over an hour old (it never wakes the car itself). The `read_at` attribute is the time of the last confirmed read; the example card warns when it is older than 6 hours.

#### Scripts
All scripts that talk to the car call **Tesla - ensure awake** first. It presses **Wake up** if the car is asleep, waits up to 60 s for **Asleep** to turn off, and stops with an error if the car does not wake, so nothing is sent to a sleeping car.

| Script | Use |
| --- | --- |
| `script.tesla_ensure_awake` | Wake the car and wait for it (used by all the others). |
| `script.tesla_refresh_schedules` | Read both schedule lists. |
| `script.tesla_precondition_now` | Start preconditioning now at a given temperature. |
| `script.tesla_set_precondition_schedule` | Add, or edit by `schedule_id`, a preconditioning schedule: time, day names, one time, enabled, location (zone, or latitude/longitude). |
| `script.tesla_set_charge_schedule` | The same for a charging schedule, with start/end times. |
| `script.tesla_delete_schedule` | Delete one schedule (`kind` + `schedule_id`). |

`script.tesla_set_precondition_schedule` and `script.tesla_set_charge_schedule` also take an optional `name`; a precondition schedule name with `ch:HHMM-HHMM` asks for charging (see *Charging for a departure*).

These are also the building blocks for your own automations, for example adding a one-time preconditioning schedule from a calendar event.

#### Schedule editor
The dashboard subview lets you edit schedules without knowing their IDs:

1. Pick a schedule in the **Schedule** dropdown (entries look like `🌡️ 07:30 · Mon, Tue, Wed, Thu, Fri · Weekly · #1792681416`), or pick **➕ New precondition schedule** / **➕ New charge schedule**.
2. The fields below fill in with its values: time(s), one time, enabled, a toggle per day and, for a precondition schedule, its charge window (see below).
3. Pick the **Location**: any of your Home Assistant zones (Home first). A schedule outside every zone also offers **📍 Keep: Other, 47 km from Home**, so editing it doesn't move it.
4. **Save to car** edits the selected schedule or adds the new one. **Delete selected** removes only the selected schedule, after a confirmation.

How it works: the *sync list* automation rebuilds the dropdown whenever a schedule sensor changes, and keeps the current selection if it still exists. The *load selection* automation copies the picked schedule into the helpers. The schedule ID is the `#…` suffix of the dropdown entry, so there is no separate ID field to get out of step.

> [!NOTE]
> The car keeps a one-time schedule in its list after it has run. Delete old ones now and then: the list is limited to about ten schedules of each type (see above). `sensor.tesla_next_departure` (below) already ignores one-time schedules that have run.

#### Charging for a departure
A precondition schedule can also ask for charging before the departure. In the editor, turn on **Also charge for this departure** and set **Charge from**, **Charge until** and **Charge to** (the target %, which starts at the car's current charge limit). The default window is the 6 hours before the ready time, and it moves with the ready time when you change that. The request is stored in the schedule's own name as `ch:HHMM-HHMM@NN` (any existing name is kept in front, for example `HA shift | ch:0100-0700@80`), so it lives in the car together with the schedule.

What happens with the request depends on **Charge scheduling** (`input_select.tesla_charge_scheduling`):

| Setting | What happens |
| --- | --- |
| **Car charge schedules** | The *keep linked charge schedules in step* automation keeps a car charge schedule named `pc:<precondition id>` with the same days, one-time, enabled and location, charging in the window. Editing or deleting the precondition schedule updates or removes it. A car charge schedule can't hold a charge limit, so when the window starts the *set charge limit* automation sets the car's limit to the target. Linked charge schedules are marked 🔗 on the card and are left out of the editor's dropdown. |
| **External planner** | No car charge schedules are kept: linked ones are removed, the editor won't create new ones, and the card warns if the car still has others. The charge limit is left to the planner. Car charge schedules (and scheduled or off-peak charging) can make the car refuse to start, or start, against an external planner. The request is published for the planner instead (below). |

Storing the request needs a TeslaBLE library that supports precondition schedule names (PedroKTFC/tesla-ble#21).

#### For external charging planners
**`sensor.tesla_next_departure`** (device class timestamp) resolves the precondition schedules into concrete departures, so a planner doesn't need to understand day bitmasks, one-time rules or locations:

- **state:** the next enabled departure in the Home zone within 72 hours, or unknown.
- **`all_upcoming`:** every enabled departure in the next 72 hours, sorted, each with `at`, `schedule_id`, `name` (without the tag), `one_time`, `location` (a zone entity ID or `other`), `latitude`, `longitude`, `charge` (true/false), and `charge_start` / `charge_end` (ISO times) and `charge_target` (%) when charging was asked for.
- **`charge_scheduling`:** the current setting.

One-time schedules count only for their single run: the first ready time after they were created (their ID is the creation time). The sensor updates when the schedule list changes and every 5 minutes.

The editor also fires **`tesla_ble_departure_changed`** after a precondition schedule is saved (`change: saved`, with its times, days, one-time, enabled, location and charge window) or deleted (`change: deleted`). A planner is free to use only the departure time and ignore or override the charge window and target.

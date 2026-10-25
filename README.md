# Dino's ESPHome Spa

An ESPHome configuration for Balboa-controlled spas that adds a **Ready At** heat-up planner. You tell it when you want to get in, and it works out when to start heating. It learns how fast *your* spa heats up, so its estimates improve over time.

It runs entirely on the ESP32, so it keeps working if Home Assistant is down, and everything shows up in Home Assistant automatically.

📖 **Write-up:** [A Ready At heat-up planner for my Balboa spa with ESPHome](https://www.matthews.casa/ready-at-heat-up-planner-balboa-spa-esphome/) on [Dean's Workbench](https://www.matthews.casa)

> ### Built on esphome-balboa-spa by Brian Feucht
> All the hard work of talking to the spa is done by **[brianfeucht/esphome-balboa-spa](https://github.com/brianfeucht/esphome-balboa-spa)**, Brian Feucht's ESPHome external component. It reads the Balboa controller's serial protocol and provides the thermostat, jets, lights, filters, fault log and everything else. This repository is just a configuration that sits on top of it. Please star his project, read his README and report component issues there.
>
> His component builds on the original UART reader from [Dakoriki/ESPHome-Balboa-Spa](https://github.com/Dakoriki/ESPHome-Balboa-Spa).

---

## Contents

- [What you get](#what-you-get)
- [What you need](#what-you-need)
- [Installing](#installing)
- [Rest, Ready, High and Low: what the spa modes mean](#rest-ready-high-and-low-what-the-spa-modes-mean)
- [The Ready At planner](#the-ready-at-planner)
  - [Using it](#using-it)
  - [Every Ready At entity explained](#every-ready-at-entity-explained)
  - [How it works](#how-it-works)
  - [Status messages](#status-messages)
- [The esphome-balboa-spa options](#the-esphome-balboa-spa-options)
- [Other settings in this config](#other-settings-in-this-config)
- [Using it from Home Assistant](#using-it-from-home-assistant)
- [Known issues and tips](#known-issues-and-tips)
- [Credits and licence](#credits-and-licence)

---

## What you get

Everything from esphome-balboa-spa, configured and named consistently:

- **Thermostat**: water temperature, target temperature, Rest/Ready mode and High/Low range
- **Jets** (as a 2-speed fan: Off / Low / High), **lights**, **blower**
- **Status**: heater, circulation pump, rest mode, high range, connected
- **Filter cycles**: view and set filter 1 and 2 start times and durations, disable filter 2
- **Spa clock** with a Sync button, **fault log** (code, message, when) and **reminders** with Clear
- Wi-Fi signal (dBm and %) and a Restart button

Plus the additions in this repository:

- **Ready At**: set a time, switch the schedule on, and the spa heats up so it's ready when you are
- **A learned heat-up rate**: after every heat-up of an hour or more it updates its "minutes per 0.5°C", blending in each new run so it follows the seasons
- **A fresh temperature check**: when you switch Ready At on, the spa briefly runs its pump to get a real water temperature before planning
- **Last Heat-up**: a plain-English record of the last heating run, e.g. `31.0 to 38.5°C in 4h 53m, 19.5 min per 0.5°C, 24 Oct 05:29 (learned)`
- **It recalculates immediately** when you change the target temperature or Ready At time

## What you need

- A Balboa spa controller (the kind esphome-balboa-spa supports: see [his README](https://github.com/brianfeucht/esphome-balboa-spa))
- An ESP32 board wired to the spa's control board via an RS-485 interface. This config uses a **Lolin (WEMOS) S2 Mini** with TX on GPIO38 and RX on GPIO40. Change the `esp32:` and `uart:` sections for your hardware.
- [ESPHome](https://esphome.io) 2026.8 or newer (the ESPHome Builder add-on in Home Assistant is the easiest way)

For wiring and hardware, follow the esphome-balboa-spa project and its community. This repository doesn't cover it.

## Installing

1. In ESPHome Builder, create a new device for your ESP32 (or open your existing spa device).
2. Replace its YAML with **[spa.yaml](spa.yaml)** from this repository.
3. Open **Secrets** (top-right menu in ESPHome Builder) and add the values from **[secrets.example.yaml](secrets.example.yaml)**: your Wi-Fi name and password, an API encryption key and a password for the fallback hotspot.
4. Check these sections match your setup:
   - `esp32:` board, and `uart:` pins
   - `balboa_spa:` → `spa_temp_scale` (`C` or `F`, as your spa is set)
   - `time:` → `sntp` → `timezone`
   - The entities: remove any your spa doesn't have, or add extra jets and lights (see [the options](#the-esphome-balboa-spa-options))
5. Press **Validate**, then **Install**.
6. In Home Assistant, accept the newly discovered ESPHome device.

---

## Rest, Ready, High and Low: what the spa modes mean

Balboa spas have two independent settings, and esphome-balboa-spa maps them onto the thermostat:

| Spa setting | Thermostat in Home Assistant | What it does |
|---|---|---|
| **Ready** | mode **Heat** | Heats whenever the water drops below the set temperature. Checks the temperature regularly. |
| **Rest** | mode **Off** | Only heats during filter cycles. Saves energy, but the water temperature is only measured while the pump runs. |
| **High range** | preset **Home** | The higher temperature range (up to 40°C / 104°F). |
| **Low range** | preset **Eco** | The lower range. The spa keeps a separate set temperature for each range. |

"Off" doesn't turn the spa off. It's Rest mode. If your target is above the low range's limit (around 37°C on many spas), you'll need High range.

---

## The Ready At planner

### Using it

1. Set **Ready At** to the time you want to get in, 24-hour, e.g. `19:30`.
2. Make sure the spa's target temperature is what you want.
3. Leave the spa in **Rest** and switch **Ready At Schedule** on.

That's it. The spa briefly switches to Ready to take a fresh water temperature, works out the start time (shown in **Heating Starts At**), and goes back to Rest. At the start time it switches to Ready and heats. It repeats every day until you switch the schedule off.

### Every Ready At entity explained

**Controls**

| Entity | Type | What it does |
|---|---|---|
| **Ready At** | text, `HH:MM` | The time you want the spa ready. 24-hour, e.g. `07:00` or `19:30`. Changing it re-plans straight away. Kept through restarts. |
| **Ready At Schedule** | switch | Turns the planner on or off. Switching it on runs the [temperature check](#the-fresh-temperature-check). Switching it off cancels any check in progress and stops planning. It doesn't switch the spa out of Ready if heating has already started. Kept through restarts. |

**Information**

| Entity | Type | What it shows |
|---|---|---|
| **Ready At Status** | text sensor | What the planner is doing right now (see [Status messages](#status-messages)). |
| **Heating Starts At** | text sensor | When heating will start, `HH:MM`, or `--` when the schedule is off. |
| **Heat-up Time Needed** | sensor, minutes | How long the next heat-up is expected to take, including the margin. |
| **Last Heat-up** | text sensor | The most recent heating run of 10 minutes or more: start and end temperature, time taken, the minutes per 0.5°C it measured, when it finished, and whether it was long enough to learn from. Kept through restarts. |

**Settings** (shown under *Configuration* on the device page in Home Assistant)

| Entity | Default | Range | What it does |
|---|---|---|---|
| **Heat-up Minutes per 0.5°C** | 20 min | 5–90 | How long your spa takes to raise the water by half a degree. This is the starting guess. It's updated automatically after every heating run of an hour or more, and you can also set it yourself. |
| **Cool-down °C per Hour** | 0.3 °C/h | 0–2 | How fast the water is assumed to cool while the spa rests. Used to estimate the temperature between readings. Raise it in cold weather or if your cover is poor, lower it if the spa is often ready too early. |
| **Ready At Margin** | 15 min | 0–120 | Extra time added to every heat-up, so it's ready a little early rather than late. |
| **Back to Rest After** | 0 h | 0–12 | Hours after the Ready At time to switch the spa back to Rest. `0` means stay in Ready. For example, `3` with Ready At `19:00` returns to Rest at 22:00. |

### How it works

**Estimating the water temperature.** In Rest mode the spa only measures the water while the pump is running, so the reading can be hours old. The planner remembers the last reading it trusts: one taken while the pump was running, while heating, in Ready mode, or right after the temperature changed. Between readings it subtracts **Cool-down °C per Hour** for each hour that has passed. If there's no reading at all yet, it assumes the water is 6°C short of the target.

**Working out the start time.**

```
half-degrees short = (target − estimated temperature) ÷ 0.5, rounded up
time needed       = half-degrees short × Heat-up Minutes per 0.5°C + Ready At Margin
heating starts at = Ready At − time needed
```

It recalculates every minute, and immediately when the target temperature, Ready At time, cool-down rate, margin or back-to-rest setting changes. When the time left until Ready At is no more than the time needed, it switches the spa to Ready. If you set Ready At too late for a full heat-up, it starts straight away and gets as close as it can.

#### The fresh temperature check

When you switch **Ready At Schedule** on and the spa is in Rest:

1. It waits until the ESP32 is talking to the spa (up to 2 minutes), so it never toggles Rest/Ready blind.
2. It switches to **Ready**, which makes the spa run its circulation pump.
3. It waits for the pump to start (up to 2 minutes), then another 60 seconds so the sensor is reading the actual water.
4. It takes that temperature as the trusted reading and switches back to **Rest**, unless the new reading shows it's already time to heat, in which case it stays in Ready.

The whole check takes about 1–3 minutes. While it runs, the status says *Checking water temperature…*. If the spa is already in Ready, the check is skipped because the reading is already fresh. It doesn't run when the ESP32 starts up with the schedule already on, so it won't fight other automations that run at boot.

#### Learning the heat-up rate

Every time the heater runs, the planner notes the starting temperature and time, and keeps updating the latest temperature and time while it heats. When the heater stops:

- **Runs of 10 minutes or more** are recorded in **Last Heat-up**.
- **Runs of 60 minutes or more that raised the water by at least 0.5°C** also update **Heat-up Minutes per 0.5°C**. The new value is 70% the old figure and 30% this run's measurement (limited to 5–90 minutes), rounded to the nearest 0.5. Seasonal changes come through over a few runs, without one odd run throwing it out.

The run in progress is saved, so a restart of the ESP32 in the middle of a heat-up doesn't lose it. The run's length is measured to the last moment it was actually seen heating, so time the ESP32 was offline isn't counted.

**Temperature units.** The planner works in steps of 0.5 of whatever unit your spa uses. With `spa_temp_scale: F`, the rate is effectively minutes per 0.5°F and the cool-down rate is °F per hour. Set the starting values to suit.

### Status messages

| Ready At Status | Meaning |
|---|---|
| `Off` | Ready At Schedule is switched off. |
| `Waiting for the clock` | The ESP32 hasn't got the time from the internet yet (shortly after boot). |
| `Waiting for the spa` | The ESP32 isn't receiving data from the spa controller. Check the wiring and the *Connected* sensor. |
| `Set Ready At as HH:MM` | The Ready At text isn't a valid 24-hour time. |
| `Checking water temperature…` | The fresh temperature check is running. |
| `Will heat from 16:25` | Planned and waiting. Heating will start at the time shown. |
| `Heating, ready by 19:30` | Heating has started for today's Ready At time. |
| `Up to temperature` | The water has reached the target before the Ready At time. |
| `Enjoy! Back to Rest at 22:30` | Past the Ready At time, with **Back to Rest After** set. It will return to Rest at the time shown. |

---

## The esphome-balboa-spa options

Everything below comes from Brian's component. Each part is optional, so include only what your spa has. **[His README](https://github.com/brianfeucht/esphome-balboa-spa#readme)** is the definitive reference, with more detail, the fault code table and the latest options.

**`balboa_spa:` (the hub)**

| Option | What it does |
|---|---|
| `spa_temp_scale` | `C` or `F`, matching how your spa is set. |
| `client_id` | Optional. Overrides the automatically assigned client ID on the spa bus. |

**`climate:`** — the thermostat (target and current temperature, Rest/Ready and High/Low as [described above](#rest-ready-high-and-low-what-the-spa-modes-mean)). Use `visual:` to set the minimum, maximum and step shown in Home Assistant.

**`water_heater:`** — an alternative to `climate:` with three modes: Off (Rest), Eco (Ready + low range) and Performance (Ready + high range). Use one or the other, not both. This config uses `climate:`.

**`fan:`** — jets with speed control (Off / Low / High): `jet_1` to `jet_4`. Each accepts:

| Option | Default | What it does |
|---|---|---|
| `max_toggle_attempts` | 5 | How many times to retry if the spa ignores a change (for example, while it's heating). |
| `discard_updates` | 20 | How many spa status updates to ignore after each command, while the spa catches up. |

**`switch:`** — simple on/off controls: `jet1` to `jet4` (with the same two options), `blower`, `light`, `light2` and `filter2` (enable/disable filter cycle 2). Use either the fan or the switch for a given jet, not both.

**`light:`** — `light` and `light2` as proper Home Assistant lights. Use these *or* the `light`/`light2` switches, not both. This config uses the switch for its single light.

**`binary_sensor:`** — `blower`, `highrange`, `circulation`, `restmode`, `heatstate`, `connected` (the ESP32 is talking to the spa), `filter1_running` and `filter2_running`.

**`sensor:`** — numeric versions of `blower`, `highrange`, `circulation`, `restmode` and `heatstate`, plus the fault log: `fault_code`, `fault_total_entries`, `fault_current_entry` and `fault_days_ago`.

**`text:`** (settable, `H:MM` or `HH:MM`, 24-hour) — `spa_time`, `filter1_start_time`, `filter1_duration`, `filter2_start_time` and `filter2_duration`. They fill in from the spa automatically and update if someone changes the settings on the spa panel.

**`text_sensor:`** — `spa_time`, `filter1_config` and `filter2_config` (JSON such as `{"start":"01:00","duration":"02:00"}`, or `disabled`), `fault_message` (the fault in words), `fault_log_time`, `reminder` (e.g. *Clean Filter*, *Check pH*) and `component_version`.

**`button:`** — `sync_time` (sets the spa clock from the ESP32's time; needs `api:` and `time:`), `disable_filter2`, `request_fault_log` (refreshes the fault log) and `clear_reminder` (clears the reminder shown on the panel; press once per reminder).

## Other settings in this config

| Setting | Why |
|---|---|
| `logger: logs: BalboaSpa.CRC: NONE` | The spa's serial bus picks up electrical noise from the heater and pumps, and CRC errors are normal. This hides them while keeping other logs. |
| `uart: rx_buffer_size: 512` | Enough buffer for the busy spa bus. On the ESP-IDF framework it must be over 128. |
| `web_server:` | The ESP32's own web page and REST API on port 80. Handy for testing, and needed by apps that talk to the ESP32 directly. Add `auth:` with a username and password if your network isn't trusted. |
| `time: sntp` | The ESP32's own clock from the internet, so the planner keeps working if Home Assistant is down. The `homeassistant` time source is kept as well. |
| `wifi: ap:` with a password | If Wi-Fi is lost, the ESP32 starts a hotspot so you can reconfigure it. It's password-protected, so nobody nearby can take it over. |
| `external_components: ref: main` | Uses the latest esphome-balboa-spa. Pin it to a release tag or commit if you want every build to be identical. |
| WiFi Signal dB / Percent | Signal strength, handy because spas are often at the far end of the garden. |
| Restart button | Restarts the ESP32 only. The spa keeps running. |

## Using it from Home Assistant

Every entity appears on the device's page under Settings > Devices & services > ESPHome. Entity IDs depend on your device name. With `friendly_name: "Spa"` they look like `switch.spa_ready_at_schedule`, `text.spa_ready_at`, `sensor.spa_ready_at_status` and so on.

Example: get a phone notification when the spa is up to temperature.

```yaml
automation:
  - alias: "Spa is ready"
    triggers:
      - trigger: state
        entity_id: sensor.spa_ready_at_status
        to: "Up to temperature"
    actions:
      - action: notify.mobile_app_your_phone
        data:
          message: "The spa is up to temperature"
```

If you already have Home Assistant automations that switch the spa between Rest and Ready on a schedule, use either those or Ready At, not both, or they may undo each other.

## Known issues and tips

- **Don't use a `datetime` entity in the same config.** Adding one (for example `datetime: platform: template, type: time`) currently stops esphome-balboa-spa's Sync Spa Time button from compiling, with `expected primary-expression before '(' token` in `sync_time_button.cpp`. ESPHome's `time` namespace clashes with the C `time()` call there. That's why Ready At is a text entity. The fix is a one-line change in the component (`::time(nullptr)`). Check esphome-balboa-spa's issues for its status.
- **ESP32-S2/S3/C3 boards on the Arduino framework.** Check esphome-balboa-spa's README for the `ARDUINO_USB_CDC_ON_BOOT=0` build flag, which some ESPHome versions need on boards with native USB.
- **Temperature shows as unknown.** After a restart, the spa doesn't report a water temperature until its pump has run. In Rest mode that can be a while. Switching to Ready, or running the Ready At temperature check, gets a reading.
- **Jets or lights don't change while heating.** Some spas block changes during heating or filter cycles. The component retries automatically (see `max_toggle_attempts`).
- **Ready too early or too late?** Adjust **Cool-down °C per Hour** first, then **Ready At Margin**. Check **Last Heat-up** to see what rate it measured.

## Credits and licence

- **[esphome-balboa-spa](https://github.com/brianfeucht/esphome-balboa-spa)** by **Brian Feucht**, the ESPHome component this config is built on and which does all the communication with the spa. BSD 2-Clause licence.
- **[ESPHome-Balboa-Spa](https://github.com/Dakoriki/ESPHome-Balboa-Spa)** by Dakoriki, the original UART reader that esphome-balboa-spa is based on.
- **[ESPHome](https://esphome.io)** and **[Home Assistant](https://www.home-assistant.io)**.

This configuration and documentation are by Dean Matthews and released under the [BSD 2-Clause licence](LICENSE), the same licence as esphome-balboa-spa.

Not affiliated with or endorsed by Balboa Water Group. "Balboa" is used only to describe compatible equipment. You're controlling a heater and pumps, so use this at your own risk and keep an eye on it when you first set it up.

## More from Dean's Workbench

- **This project's write-up:** [A Ready At heat-up planner for my Balboa spa with ESPHome](https://www.matthews.casa/ready-at-heat-up-planner-balboa-spa-esphome/)
- **Website:** [www.matthews.casa](https://www.matthews.casa): articles on Home Assistant, automation, PowerShell and music
- **Other projects:** [echo-home-stream](https://github.com/axelthefatcat/echo-home-stream) (play your own audio stream on Amazon Echo) and [dinos-discogs-tagger](https://github.com/axelthefatcat/dinos-discogs-tagger) (fix music tags with PowerShell)

# NIBE Fighter 1230 heating controller

Two Raspberry Pi Pico 2 W devices running ESPHome: **nibe-controller** operates the normally open relay; **nibe-sensor** is the sole DHT11 room thermometer, powered by a Waveshare Pico-UPS-B and a 600 mAh, 3.7 V LiPo. Home Assistant provides a thermostat-style dashboard and a weekly schedule. The sensor sends temperature directly to the controller over local Wi-Fi using encrypted ESPHome UDP packet transport, and publishes temperature and humidity to Home Assistant through its own ESPHome integration.

For installation, removal of the old controller entities, battery expectations, and communication-loss tests, follow the [separate sensor migration guide](docs/separate-sensor.md).

The relay closes the NIBE Fighter 1230's **X1:3–X1:4** external adjustment input. With menu **2.5, “Kompensering yttre”**, set to **−10**, closing the contact reduces the heating-curve offset. Opening the contact restores the normal curve.

**−10 is a heating-curve offset, not necessarily a 10 °C reduction in supply temperature.** This input does not directly switch off the compressor or guarantee an immediate reduction in electrical power.

## Project status and files

The previous single-Pico setup was tested for temperature readings, relay operation, local temperature control, the Home Assistant schedule, and dashboard setup. The new two-Pico configuration has not yet been tested on hardware. A −3 °C sensor correction has been applied based on comparison with an independent thermometer. Connection to the actual heater is still awaiting testing.

| File | Purpose |
| --- | --- |
| `nibe-controller.yaml` | Relay controller configuration; receives remote temperature, with no local DHT or exported temperature/humidity sensors. |
| `nibe-sensor.yaml` | Separate Pico 2 W DHT11 thermometer, reporting every 60 seconds. |
| `docs/separate-sensor.md` | Migration, required secrets, Home Assistant cleanup, battery expectations, and bench tests. |
| `secrets.example.yaml` | Placeholder secret names for both configurations; copy into your private ESPHome secrets file. |
| `docs/enclosure.md` | Mounting, connectors, and separate controller/sensor enclosures. |
| `LICENSE` | Custom noncommercial and no-patenting license draft. |

**Experimental hardware project:** the previous setup was tested, but the two-device setup and connection to the heater still require physical verification. Earlier development notes report ESPHome 2026.9.1 schema validation and C++ source generation; that does not establish successful firmware compilation or installation. Build and run the bench tests for your own hardware.

Both configurations select `rpipico2w` (Pico 2 W). Change the board setting if you have an original Pico W. Compiled firmware is excluded because it can contain Wi-Fi credentials, passwords, and encryption keys; build your own firmware with private secrets.

## Start here

1. Gather the hardware below and keep the relay disconnected from the heater.
2. Prepare both configurations and private secrets using [Set up ESPHome](#set-up-esphome).
3. Flash both devices, then add both ESPHome integrations in Home Assistant.
4. Configure the weekly schedule and heartbeat automation.
5. Run the bench tests before connecting the heater.

If upgrading an existing single-Pico installation, follow the [migration guide](docs/separate-sensor.md) for entity cleanup and wiring changes.

```text
DHT11 → sensor Pico + UPS/battery ── encrypted local Wi-Fi ──→ controller Pico
             │                                                    │
             └─ temperature/humidity → Home Assistant             └─ relay COM/NO → NIBE
                                          │
                                          └─ schedule heartbeat → controller Pico
```

## Hardware and wiring

**The DHT11 connections below now belong to nibe-sensor. Relay connections belong to nibe-controller.** Disconnect the former controller DHT wiring with power removed. The thermometer has its own ventilated enclosure, UPS, and battery. The devices communicate over Wi-Fi; no cable or common ground is needed between them.

For a build using an off-the-shelf box and basic hand tools, see the [enclosure guide](docs/enclosure.md), including mounting details, cable routing, and clearly labelled historical concept images. No 3D printer is required.

Required hardware:

- **Two** Raspberry Pi Pico 2 W boards with headers: one controller and one sensor.
- Waveshare Pico-UPS-B and a compatible **600 mAh, 3.7 V LiPo** for the sensor; verify connector polarity and the UPS manufacturer's battery requirements.
- Three-pin DHT11 module labelled `+`, `out`/`S`, and `−`.
- A 5 V relay module with a driver circuit and isolated COM, NO, and NC contacts.
- 1 kΩ series resistor and 10 kΩ pull-down resistor.
- USB data cable, USB power supply, and suitable wiring/enclosure.

The project's relay module was tested separately: 3.3 V applied to `S` through 1 kΩ activated it; removing the signal released it. Do not assume every Arduino-kit relay module behaves the same way. A bare relay coil must not be driven directly from a Pico GPIO.

Disconnect USB **and the sensor battery** while wiring. Keep the NIBE disconnected throughout initial setup and bench testing.

| Device | Module connection | Pico connection | Physical pin |
| --- | --- | --- | ---: |
| Sensor | DHT11 `+` | 3V3 OUT | 36 |
| Sensor | DHT11 `out`/`S` | GP15 | 20 |
| Sensor | DHT11 `−` | GND | 38 |
| Controller | Relay `+` | VBUS, USB 5 V | 40 |
| Controller | Relay `−` | GND | 38 |
| Controller | Relay `S` | GP14 through 1 kΩ | 19 |

Add a **10 kΩ resistor between GP14 and GND**, on the Pico side of the series resistor:

```text
CONTROLLER PICO (USB powered)
GP14 (19) ──┬──[1 kΩ]── Relay S
            │
          [10 kΩ]
            │
GND (38) ───┴────────── Relay −

VBUS (40) ───────────── Relay +

SEPARATE SENSOR PICO (UPS/battery or USB powered)
3V3 OUT (36) ────────── DHT11 +
GP15 (20) ───────────── DHT11 out
GND (38) ────────────── DHT11 −
```

GP15 means GPIO 15, **not physical pin 15**. Each module uses the ground of its own Pico. Do not join the two devices' grounds. The VBUS connection above assumes the Pico is powered through USB. Pico GPIO signals are 3.3 V; do not apply 5 V to them.

Four-band resistor colours:

- **1 kΩ:** brown–black–red–gold.
- **10 kΩ:** brown–black–orange–gold.

The DHT11 module normally includes its data pull-up resistor. Follow the module's printed labels, since header order varies. Place the sensor away from the relay, Pico, power supply, sunlight, and drafts so it measures representative room temperature.

References: [Raspberry Pi pinouts and documentation](https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html), [DHT wiring](https://learn.adafruit.com/dht/connecting-to-a-dhtxx-sensor).

## How control works

```text
Relay closed = temperature requests reduction OR a fresh schedule command requests reduction
Relay open   = neither condition requests reduction
```

| Condition | Behaviour |
| --- | --- |
| Temperature above the threshold | Close the relay. |
| Temperature at or below threshold minus release margin | Clear the temperature request. |
| Temperature between those limits | Retain the previous temperature request. |
| Schedule active | Close the relay regardless of temperature. |
| Schedule inactive | Clear the schedule request; temperature may still keep the relay closed. |
| No schedule command for 60 seconds | Clear the schedule request and continue temperature control from nibe-sensor if readings are fresh. |
| No valid remote temperature received within 180 seconds | Clear the temperature request and thermostat current-temperature display. A fresh active schedule can still request reduction. |
| Both schedule commands and valid temperature readings unavailable | Open the relay. |
| Startup or controller power loss | Relay defaults to de-energized; COM–NO is open. On startup, local control begins after valid readings arrive. |

The controller evaluates these rules every **5 seconds**. The separate DHT11 is sampled every **60 seconds**. A sensor fault publishes an invalid reading; a 130-second sensor timeout also prevents repeatedly broadcasting an old value. The controller accepts only finite remote temperature readings. Home Assistant sends the current schedule command every **20 seconds**, including when the schedule is inactive. The command itself is the heartbeat; an open API connection alone is not a heartbeat.

The initial threshold is **21 °C**, with a **1 °C** release margin. This closes the relay above 21 °C and releases the temperature request at or below 20 °C. Lowering the threshold makes reduction more likely; raising it makes reduction less likely.

Threshold and margin are saved on the controller and reused during fallback. Settings are persisted through ESPHome's flash preferences, so allow time for a recent change to be saved before removing power. Schedule requests are not restored after a restart; Home Assistant must send a fresh command.

`reboot_timeout: 0s` in both `api:` and `wifi:` disables periodic reboots caused solely by losing those connections, allowing temperature control to continue if direct sensor packets still arrive. Home Assistant can be offline, but Wi-Fi communication between the Picos is required. It does not disable the hardware watchdog.

## Set up ESPHome

1. Install and open **ESPHome Device Builder** in Home Assistant if it is not already installed.
2. Create a device named **`nibe-controller`** and select your actual Raspberry Pi board.
3. Open its configuration editor. Copy the contents of `nibe-controller.yaml`, using the required shared transport secret described in the migration guide. Create `nibe-sensor` separately with `nibe-sensor.yaml`. Use one copy of each top-level YAML section; do not append a second `api:`, `wifi:`, `sensor:`, or `interval:` section.
4. Select the appropriate board:

   ```yaml
   rp2:
     board: rpipicow     # Original Pico W
   ```

   For a Pico 2 W, use `board: rpipico2w` instead. Both use the same physical pin mapping described here. Use a current ESPHome version supporting `rp2` and Template Climate; if validation reports an unknown platform, update Device Builder.

5. Open [secrets.example.yaml](secrets.example.yaml). In Device Builder's secrets editor, copy these entries into your private `secrets.yaml` and replace **every placeholder** with your own values:

   ```yaml
   wifi_ssid: "YOUR_WIFI_NAME"
   wifi_password: "YOUR_WIFI_PASSWORD"
   nibe_controller__encryption_key: "CONTROLLER_WIZARD_GENERATED_BASE64_KEY"
   nibe_controller__ap_password: "YOUR_CONTROLLER_HOTSPOT_PASSWORD"
   nibe_sensor__encryption_key: "SENSOR_WIZARD_GENERATED_BASE64_KEY"
   nibe_sensor__ap_password: "YOUR_SENSOR_HOTSPOT_PASSWORD"
   nibe_sensor_transport_key: "YOUR_RANDOM_SHARED_TRANSPORT_SECRET"
   ```

   Each API key must be a base64-encoded 32-byte key generated by the ESPHome device wizard. Each hotspot password must be at least eight characters. The transport secret is a separate random string, shared by both Picos. Both configurations already reference these exact names with `!secret`; no credentials belong in the device YAML files.

   Both Picos must be on a local network permitting UDP broadcasts on **port 18511**. Guest Wi-Fi, client isolation, and broadcast filtering can prevent direct communication.

6. Save and **Validate** both configurations, then build each device's firmware. Resolve all errors before installing. Keep `secrets.yaml` and generated firmware private; they are excluded by `.gitignore`.

`captive_portal:` provides a Wi-Fi setup page through the fallback hotspot if normal Wi-Fi connection fails. It is separate from the heating-control fallback.

### First flash over USB

1. Select **Install → Manual download** and build/download the `.uf2` firmware.
2. Disconnect the Pico from USB.
3. Hold **BOOTSEL** while connecting it to your computer with a USB data cable, then release the button.
4. Copy the `.uf2` file onto the **RPI-RP2** drive for a Pico W, or **RP2350** drive for a Pico 2 W.
5. The Pico restarts automatically. Leave it powered and wait for it to appear online in Device Builder.
6. Open wireless **Logs** and confirm room temperature and humidity readings on nibe-sensor and received packets / Remote Temperature Fresh on nibe-controller.

For later changes, use **Save → Validate → Install → Wirelessly**. Build fresh firmware when changing the board, Wi-Fi settings, encryption key, or control logic.

Reference: [ESPHome Raspberry Pi platform](https://esphome.io/components/rp2/).

## Add both devices to Home Assistant

1. Open **Settings → Devices & services**.
2. Configure the discovered ESPHome device. If it is not discovered, select **Add integration → ESPHome** and enter the Pico's local IP address. The default API port is **6053**.
3. Enter the API encryption key if requested. After changing an existing device's key, update the integration's key if Home Assistant requests reconfiguration.
4. Confirm the following entities appear:

| Entity name | Purpose |
| --- | --- |
| Room Temperature (NIBE Sensor) | Corrected DHT11 temperature; the only exported room-temperature sensor. |
| Room Humidity (NIBE Sensor) | DHT11 relative humidity. |
| Remote Temperature Fresh (controller) | A finite remote temperature packet has arrived within 180 seconds. |
| Reduction Temperature Threshold | Saved threshold, adjustable from Home Assistant. |
| Temperature Release Margin | Saved release margin. |
| Heating Reduction Active | Pico's commanded relay state; not physical contact feedback. |
| Home Assistant Schedule Control Active | Fresh schedule commands are arriving, regardless of whether the schedule is currently active. |
| NIBE Temperature Control | Thermostat interface for the same saved threshold. |

Add nibe-sensor as its own ESPHome integration. After flashing the controller, remove its obsolete Room Temperature and Room Humidity entities, and select the new sensor entities in dashboards and automations; see the migration guide. The controller retains an internal received-temperature value for its thermostat display, without creating a duplicate sensor entity.

The relay itself is internal to ESPHome. The old manual switch from bench testing may remain as an unavailable entity in Home Assistant; remove that obsolete entity if present. The algorithm now owns the relay.

## Create the weekly schedule

1. Open **Settings → Devices & services → Helpers → Create helper → Schedule**.
2. Name it **NIBE Heating Reduction**.
3. For reduction every night from **22:00 to 06:00**, add two blocks on every day: **00:00–06:00** and **22:00–24:00**.
4. Save and check its entity ID. The automation below expects **`schedule.nibe_heating_reduction`**. If yours differs, replace both occurrences in the automation.
5. Check Home Assistant's timezone under its general settings; this installation uses **Europe/Stockholm**. The weekly schedule follows Home Assistant's local clock.

Edit the same helper whenever you want to change the hours. There is no need to edit the automation or reflash the Pico for schedule changes.

Reference: [Home Assistant Schedule helper](https://www.home-assistant.io/integrations/schedule/).

## Create the schedule and heartbeat automation

First find the action registered by the Pico. In current Home Assistant, open **Settings → Tools → Actions** and search for **`set_schedule_reduction`**. Older versions may show **Developer tools → Actions**, or **Services** instead of Actions.

For a device named `nibe-controller`, the expected action is **`esphome.nibe_controller_set_schedule_reduction`**. Use the actual action name shown by your installation if different. This is a custom ESPHome action, not a switch entity.

Open **Settings → Automations & scenes → Create automation → Create new automation**. Open the editor menu, select **Edit in YAML**, and paste:

```yaml
alias: NIBE schedule and heartbeat
description: Refresh the Pico's schedule command every 20 seconds.
triggers:
  - trigger: time_pattern
    seconds: "/20"

  - trigger: state
    entity_id: schedule.nibe_heating_reduction

  - trigger: homeassistant
    event: start

conditions:
  - condition: template
    value_template: >
      {{ states('schedule.nibe_heating_reduction') in ['on', 'off'] }}

actions:
  - action: esphome.nibe_controller_set_schedule_reduction
    data:
      requested: "{{ is_state('schedule.nibe_heating_reduction', 'on') }}"

mode: restart
```

Save and enable the automation. It sends both `true` and `false` commands, so the heartbeat continues outside scheduled hours. An unavailable schedule prevents commands from being sent, allowing the controller to fall back to temperature control using fresh direct packets from nibe-sensor. After reconnecting, the next successful command restores schedule control.

Within approximately **25 seconds**, **Home Assistant Schedule Control Active** should turn on. Check the automation's traces if it does not. For testing in the Actions tool, the payload is `requested: true` or `requested: false`; either command expires after 60 seconds unless refreshed.

References: [ESPHome API actions](https://esphome.io/components/api/), [Home Assistant automation triggers](https://www.home-assistant.io/docs/automation/trigger/), [Home Assistant Tools](https://www.home-assistant.io/docs/tools/dev-tools/).

## Dashboard

### Thermostat card

1. Open the dashboard and select **Edit dashboard → Add card**.
2. Select **Thermostat**.
3. Choose **NIBE Temperature Control** as its entity.
4. Set the card name to **NIBE Heating**, then save.

The card shows measured room temperature and a dial for the reduction threshold. The dial is synchronized with **Reduction Temperature Threshold**. **Auto** is the interface's operating mode, not confirmation that the NIBE compressor is running. The card does not enable or disable the schedule.

### Schedule and status card

Add an **Entities** card containing:

- **NIBE Heating Reduction** — the weekly schedule helper.
- **Heating Reduction Active** — the commanded relay state.
- **Home Assistant Schedule Control Active** — heartbeat status.
- **Temperature Release Margin** — the adjustable margin.
- **Remote Temperature Fresh** — direct sensor reception status.
- Optionally, **Room Humidity** from **NIBE Sensor**.

To change schedule hours, edit the helper under **Settings → Devices & services → Helpers**. Tapping its dashboard entry opens its details; some Home Assistant versions also expose editing there.

Reference: [Home Assistant Thermostat card](https://www.home-assistant.io/dashboards/thermostat/).

## Sensor comparison and correction

Place an independent air thermometer beside the DHT11, away from heat sources and drafts. Allow both to settle for **20–30 minutes**, then compare several readings. An infrared thermometer measures surfaces and is not a direct air-temperature reference.

The project's independent thermometer read **24 °C** while the DHT11 read **27 °C**, so the temperature sensor in **nibe-sensor.yaml** uses:

```yaml
temperature:
  name: "Room Temperature"
  id: room_temperature
  filters:
    - offset: -3.0
    - timeout: 130s
```

The correction runs on nibe-sensor before direct packet transmission and Home Assistant reporting, so the controller and Home Assistant use the same reading. Keep this correction in one place; do not subtract it again in Home Assistant. A single comparison establishes an offset at that temperature, not accuracy across the full range. Reassess the offset if the difference changes or the sensor is moved.

## Bench tests before connecting the heater

Keep the NIBE disconnected during these tests.

1. **Startup:** restart the Pico and confirm the relay initially stays released. It can subsequently close if temperature or a fresh schedule command requests reduction.
2. **Temperature release:** with the schedule inactive, set the threshold to at least the corrected temperature plus the release margin. The relay should open within about 5 seconds.
3. **Temperature activation:** lower the threshold below the corrected temperature. The relay should close within about 5 seconds.
4. **Schedule:** set the threshold high enough to release temperature control, then temporarily add a schedule block covering the current time. The relay should close regardless of temperature. Remove the block and confirm it releases.
5. **Heartbeat fallback:** activate a schedule block, keep the temperature request inactive, and disable the heartbeat automation. After approximately 60–65 seconds from its last successful command, schedule control should clear and the relay should open. Lower the threshold to confirm direct remote-temperature control still works. Re-enable the automation afterward.
6. **Sensor fallback:** with the relay initially unpowered, disconnect only the DHT11 DATA wire, then power both Picos and test without valid schedule commands. With no valid temperature reading, the relay must remain open. Also turn off nibe-sensor while temperature reduction is active and the schedule inactive: within 180–185 seconds of the last received packet, the relay must release and Remote Temperature Fresh must turn off. Reconnect wiring with power removed.
7. **Contacts:** with controller power removed, COM–NC should have continuity and COM–NO should be open. Continuity testing must be done with the contacts isolated from the NIBE. When powered and commanded on, confirm COM–NO closes; these are isolated dry contacts.

## Connect to the NIBE Fighter 1230

Have someone qualified for work inside the heater isolate its electrical supply before accessing the terminals. The relay contact wiring must remain electrically separate from the Pico's power and ground.

| Relay terminal | NIBE terminal |
| --- | --- |
| COM | X1:3 |
| NO | X1:4 |
| NC | Unused |

Use only the isolated **potential-free COM–NO contact**. Do not feed 3.3 V, 5 V, or controller ground into X1:3–X1:4.

Set menu **2.5, “Kompensering yttre”**, to **−10**. Begin with the relay released, then verify that closing it applies external compensation and opening it restores normal operation. Repeat temperature, schedule, and communication-loss tests with the installed system. Heating response takes time; a relay click is not proof of a particular supply temperature or electrical consumption.

The NIBE manual describes this external contact on printed page **18**: [Fighter 1230 installation and maintenance manual](https://vpkollen.se/uploaded/manualer/Nibe/Fighter_1230.pdf).

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Relay does not release after lowering the threshold | Lowering the threshold requests more reduction. Raise it above the room reading plus the release margin. Also check the schedule. |
| Relay stays closed outside scheduled hours | Temperature control may still be requesting reduction. The two requests are combined with OR. |
| Schedule control status is off | Confirm the automation is enabled, the Pico is online, and the action name and schedule entity ID match. Inspect automation traces. |
| Schedule control status is on while the schedule is off | Expected: this status means fresh commands are arriving, not that reduction is scheduled. |
| Thermostat card cannot find an entity | Confirm Template Climate validated and the updated firmware was installed. Select the climate entity, not the number entity. |
| Temperature stays too high | Confirm the offset is under temperature in nibe-sensor.yaml, its firmware was updated, and the sensor is away from heat sources, including the UPS. |
| Remote Temperature Fresh stays off | Confirm both Picos are flashed, the shared transport secret matches, and local UDP broadcasts on port 18511 are permitted. Guest Wi-Fi or client isolation can block this. |
| DHT11 reports errors or no readings | Check 3.3 V supply, sensor-to-Pico ground, DATA on physical pin 20, and `model: DHT11`. Keep sensor wires short. |
| YAML validation fails | Check indentation, board selection, current ESPHome version, and duplicate top-level sections. |
| Pico is unavailable in Home Assistant | Check Wi-Fi and API encryption settings. An open wireless log connection does not replace schedule heartbeat commands. |

## Future household consumption control

Consumption control is **not implemented yet**. The existing MQTT meter is awaiting replacement.

The planned rule is to request reduction when household power stays above **X kW continuously for T**, with a separate release threshold to avoid chatter. Home Assistant will need a fresh power entity in **W or kW**, not just an energy total in kWh. The eventual request will be combined with temperature and schedule using OR, while retaining direct remote-temperature control if Home Assistant commands expire and the sensor packets remain fresh.

## Disclaimer

This is an experimental DIY project provided “as is,” without warranties of any kind. Substantial parts of this code is produced with AI. It has not been certified or fully tested with the NIBE Fighter 1230. It is not affiliated with or endorsed by NIBE, Raspberry Pi, or Waveshare.

You are responsible for verifying compatibility, wiring, configuration, and safe operation. Incorrect installation or unexpected behaviour may damage equipment, interrupt heating, or cause injury. Have a qualified professional perform work inside the heat pump, and complete the documented bench tests before connecting it.

This controller adjusts heating through an external input; it does not replace the heat pump’s built-in controls or safety protections, and it does not guarantee energy savings.

To the extent permitted by applicable law, the authors accept no liability for damage or loss arising from use of this project.

## License

This project uses a [custom noncommercial and no-patenting license draft](LICENSE). It permits noncommercial use and modification subject to its conditions, and prohibits seeking patents or utility models covering the project or its derivatives. Commercial use requires separate written permission.

This is a source-available project, not an OSI-approved open-source license. The custom terms require legal review before relying on their enforceability; they do not themselves invalidate patents or bind parties who have not accepted or exercised the license permissions. Linked third-party documentation, hardware designs, and dependencies retain their own licenses.

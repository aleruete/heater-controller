# Move the thermometer to NIBE Sensor

The new thermometer is a **Pico 2 W**, with the existing **DHT11 on GP15**, a **Waveshare Pico-UPS-B**, and a **600 mAh, 3.7 V LiPo**. The original Pico retains the relay.

## Chosen behaviour

- `nibe-sensor` is the only physical room thermometer and the only device exporting Room Temperature and Room Humidity to Home Assistant.
- DHT readings are taken every **60 seconds**. This is a useful starting interval for room heating, where temperature normally changes slowly.
- The sensor publishes to Home Assistant through its encrypted native API and sends temperature directly to `nibe-controller` through encrypted ESPHome UDP packet transport. No Home Assistant forwarding automation or MQTT broker is required.
- Packet transport sends sensor updates as they occur and repeats the current state every 60 seconds. These repeat packets improve recovery after a missed packet or controller restart; they are not new measurements.
- The controller retains its threshold, release margin, thermostat card, and schedule action. Its received temperature is **internal**, so no duplicate temperature sensor entity is exported. The thermostat's current-temperature display comes from the remote sensor.
- A non-finite remote temperature clears temperature-based reduction on the next 5-second control evaluation. If packets stop, the last usable reading expires after **180 seconds**; reduction clears within **180–185 seconds of the last received usable packet**. The thermostat stops displaying that old temperature.
- DHT read errors publish an invalid reading. A **130-second timeout** on the sensor is an additional safeguard against rebroadcasting an old value if sensor updates stop. Receipt time measures communication freshness; it is not an absolute measurement timestamp.
- Home Assistant schedule commands still expire after 60 seconds. A fresh active schedule can request reduction even when temperature is unavailable. With neither request valid, the relay opens.
- With Home Assistant offline, temperature control still operates if both Picos can communicate over Wi-Fi. Loss of the Wi-Fi network removes this fallback.

## Battery expectations

The capacity unit is **mAh**, not mA. The nominal stored energy is approximately **0.6 Ah × 3.7 V = 2.22 Wh**.

Reporting less often does not put this ESPHome Pico into deep sleep. ESPHome's standard deep-sleep component currently supports other platforms, but **not RP2/Pico**. Both the Pico and its Wi-Fi connection remain powered in these configurations. Changing from one minute to five minutes can reduce transmissions, but should not be assumed to provide a large runtime improvement.

Use a fully charged battery and measure actual runtime with USB disconnected. If you measure average current at the **battery**, ideal runtime is approximately:

| Example measured battery current | Ideal runtime from 600 mAh |
| --- | ---: |
| 20 mA | 30 hours |
| 50 mA | 12 hours |
| 100 mA | 6 hours |

These are arithmetic examples, **not measurements or predictions for this build**. Actual usable capacity and cutoff reduce runtime. Battery-side current already includes the UPS's conversion loss; current measured on a different voltage rail cannot be used directly in this table. The UPS includes an INA219 monitor, but battery telemetry is not enabled in this configuration.

Start at **60 seconds**. If long unattended battery operation is essential, a separate power-saving design is needed; increasing this interval alone is insufficient. Keep the DHT away from the Pico, UPS, and battery charging heat. Recheck its existing −3 °C calibration in the new position.

References: [ESPHome DHT](https://esphome.io/components/sensor/dht/), [ESPHome deep sleep](https://esphome.io/components/deep_sleep/), [Waveshare Pico-UPS-B](https://www.waveshare.com/wiki/Pico-UPS-B).

## Prepare both configurations

1. Back up the configurations currently running in ESPHome Device Builder privately. Build fresh firmware for each device; this repository does not distribute personal firmware builds.
2. Keep the controller's existing device name and network settings when replacing its configuration with `nibe-controller.yaml`. Rotate the previously exposed controller API key and fallback-hotspot password in your private secrets file. Update the Home Assistant integration's API key after flashing; keeping the device name preserves its identity.
3. Create a second Device Builder device named **`nibe-sensor`**, select **Pico 2 W**, and use `nibe-sensor.yaml`. If the wizard has already created this device, edit it instead of creating a duplicate.
4. In Device Builder's secrets editor, add:

   ```yaml
   # Existing shared Wi-Fi credentials:
   wifi_ssid: "YOUR_WIFI_NAME"
   wifi_password: "YOUR_WIFI_PASSWORD"

   # Controller credentials (rotate values exposed in earlier revisions):
   nibe_controller__encryption_key: "CONTROLLER_WIZARD_GENERATED_BASE64_KEY"
   nibe_controller__ap_password: "YOUR_CONTROLLER_HOTSPOT_PASSWORD"

   # New sensor's key generated by the ESPHome device wizard:
   nibe_sensor__encryption_key: "SENSOR_WIZARD_GENERATED_BASE64_KEY"
   nibe_sensor__ap_password: "YOUR_SENSOR_HOTSPOT_PASSWORD"

   # One random shared secret, exactly the same for BOTH Picos:
   nibe_sensor_transport_key: "YOUR_RANDOM_SHARED_TRANSPORT_SECRET"
   ```

   The API key must be a base64-encoded 32-byte key. The transport secret is a separate random string; it is not the API key. The fallback password must contain at least eight characters. Do not commit real secrets to this repository.
5. Validate both device configurations with **ESPHome 2026.9.1 or newer**. Both files select `rpipico2w`; adjust the controller's board if its actual hardware differs.

Both Picos must use a local network allowing UDP broadcasts on **port 18511**. Guest Wi-Fi, client isolation, separate VLANs, or broadcast filtering can prevent direct communication even while both devices appear online in Home Assistant. Start with both on the same ordinary Wi-Fi network. The shared secret encrypts temperature packets; the sensor also enables a rolling code to reject previously received packets.

## Install and move the wiring

Keep the relay contacts disconnected from the heater during initial tests.

1. Disconnect USB and the sensor battery before disconnecting the DHT wiring from the controller. Connect it only to the new sensor Pico:

   | DHT11 pin | Sensor Pico connection | Physical pin |
   | --- | --- | ---: |
   | `+` | 3V3 OUT | 36 |
   | `out` / `S` | GP15 | 20 |
   | `−` | GND | 38 |

2. Leave the original controller relay on GP14 with its existing wiring.
3. Validate and build `nibe-sensor`, then use the initial USB/BOOTSEL UF2 installation procedure in the README. An already provisioned sensor can be installed wirelessly if OTA is configured and it is online.
4. Add **NIBE Sensor** through **Settings → Devices & services → ESPHome**, using its own API key. Confirm that Room Temperature and Room Humidity update about once a minute.
5. Install the updated `nibe-controller.yaml` on **nibe-controller**, using the rotated controller credentials and updating Home Assistant's API key. Confirm **Remote Temperature Fresh** turns on and the thermostat displays the new sensor's temperature. Allow about one reporting interval for first reception.
6. Apply the bench tests below before reconnecting the heater. The threshold and margin use the controller's existing saved values; initial defaults do not overwrite saved settings.

## Make Home Assistant forget the old thermometer

The new firmware removes the controller's old Room Temperature and Room Humidity entities from its API. Home Assistant may retain their registry entries as unavailable; it does not automatically redirect references to the new sensor.

1. After the controller has reconnected with the updated firmware, open **Settings → Devices & services → Entities** and filter by **nibe-controller**.
2. Identify its obsolete **Room Temperature** and **Room Humidity** entities. Check the owning device carefully: delete only the old controller entities, not the new NIBE Sensor entities. Use each obsolete entity's settings to delete it once Home Assistant reports it is no longer provided. If deletion is unavailable, reload the controller's ESPHome integration and confirm the updated firmware is actually running, then retry.
3. Replace every old temperature or humidity reference in dashboard cards, automations, scripts, and helpers with the corresponding **NIBE Sensor** entity. Check their actual entity IDs in Home Assistant; likely IDs are `sensor.nibe_sensor_room_temperature` and `sensor.nibe_sensor_room_humidity`, but names or existing collisions can change them.
4. Keep **NIBE Temperature Control**, its thermostat card, **Reduction Temperature Threshold**, **Temperature Release Margin**, and the existing schedule/heartbeat automation. These still belong to the controller. The thermostat receives its measured temperature internally from the new sensor.
5. Add **Remote Temperature Fresh** to the status card. This indicates usable direct packets, independently of the sensor's Home Assistant connection.

Deleting obsolete entity registry entries does not erase historical recorder data. No history/database cleanup is required for this migration.

References: [Home Assistant ESPHome integration](https://www.home-assistant.io/integrations/esphome/), [ESPHome packet transport](https://esphome.io/components/packet_transport/), [ESPHome packet transport sensor](https://esphome.io/components/sensor/packet_transport/).

## Bench verification

1. **Startup without sensor:** leave the sensor off, ensure the schedule is inactive, restart the controller, and confirm the relay stays released and Remote Temperature Fresh is off.
2. **Remote temperature:** power the sensor, wait for a reading, and compare its HA temperature with the controller's thermostat current temperature. There must be no exported controller Room Temperature or Room Humidity entity in the updated API.
3. **Threshold/hysteresis:** with the schedule inactive, lower the threshold below the reading and confirm reduction activates. Raise it to at least the reading plus the release margin and confirm release within about five seconds. A reading between threshold and threshold-minus-margin retains the existing temperature request.
4. **Stable temperature:** leave the devices running at a steady temperature for more than three minutes. Remote Temperature Fresh must remain on: identical repeated temperature values are still usable reports.
5. **Sensor power loss:** while temperature reduction is active and the schedule inactive, turn off nibe-sensor. Within 180–185 seconds of the last usable packet, the relay releases, Remote Temperature Fresh turns off, and the thermostat's measured temperature becomes unavailable. Restoring the sensor restores control after new packets arrive.
6. **DHT failure:** remove sensor power, disconnect DATA, and restart the sensor with the schedule inactive. Confirm invalid measurements do not request reduction. Reconnect wiring only with power removed.
7. **Home Assistant loss:** while direct reception is working, stop Home Assistant or disconnect only its server from the network. Leave both Picos and Wi-Fi powered. After the schedule's 60-second timeout, changing the threshold across the remote temperature must still affect the relay.
8. **Schedule independence:** with no usable temperature, a fresh active schedule can close the relay. Disable the heartbeat automation and confirm release within 60–65 seconds of its last command. Re-enable it afterward.

Earlier development notes report ESPHome 2026.9.1 schema validation and C++ source generation with temporary test secrets. Revalidate the current files with your private secrets before installing. Firmware compilation, installation, battery runtime, radio delivery, and these physical bench tests remain to be performed on the devices.

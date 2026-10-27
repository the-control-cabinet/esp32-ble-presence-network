# 04 · Runbook: Renaming ESPHome Boards and Cleaning Up HA

[← Back to README](../README.md)

Three proxies were renamed to match their rooms. Each rename has knock-on effects in Home Assistant. This runbook is the checklist that came out of doing it.

## Before flashing
- Change `name:` / `friendly_name:` and the entity names. Keep the **API key value** the same: if the secret's *name* changes, copy the old value across, so HA doesn't need re-keying.
- **No `//` comments inside lambdas.** On ESPHome 2026.10 a `//` comment in a lambda ended compilation with `expected '}' at end of input`.

## After flashing

| Step | Where | Why |
|---|---|---|
| 1. If the dashboard shows the board offline, use **Logs → Wirelessly**, or `wifi: use_address:` temporarily | ESPHome Builder | mDNS for the new hostname can lag |
| 2. **Ignore** the stale "discovered" card with the old name | HA → ESPHome | It points at a hostname that no longer exists |
| 3. **Reconfigure** the existing entry's host to `<new-name>.local` | HA → ESPHome | HA kept connecting to the old name (no "Home Assistant connected" in the device log until this was done) |
| 4. Rename the **device** if HA kept the old display name, then **reload Bermuda** | HA → Devices / Bermuda | Bermuda caches scanner names |
| 5. Set the device's **area** | HA → Devices | Bermuda uses proxy areas for room presence |
| 6. Delete the old **MQTT device** | HA → MQTT | MQTT discovery created a separate copy under the old name |
| 7. Re-pair any device that **connected through** the renamed board | Device / integration | Bonds and connection ownership change |
| 8. Clear **orphaned entities** (red "no longer provided"), checking **Related** first | HA → Entities | ESPHome unique IDs derive from entity names, so renamed entities are new, and the old ones are orphaned |

## MQTT hygiene
- **Boards that use MQTT only for the scanner:** `discovery: false`, so there are no duplicate entities.
- **Boards that don't use MQTT at all:** remove the `mqtt:` block (one fewer connection at boot, and one fewer suspect in [doc 03](03-incident-esphome-2026-10-watchdog.md)).
- **If a deleted MQTT device keeps coming back:** the firmware still has MQTT, or the broker holds **retained discovery messages**. Clear them by publishing an empty **retained** payload to each `homeassistant/…/<node>/…/config` topic (HA → MQTT → Configure → Publish).

## HA configuration
- Scanner sensors: rename the **`name:`** only; keep `unique_id` and the topics ([config/ha-mqtt-sensors.yaml](../config/ha-mqtt-sensors.yaml)).
- Recorder: `entity_globs: sensor.ble_device_list*` (needs a restart).

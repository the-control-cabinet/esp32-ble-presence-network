# 01 · Proxy Roles: One Connection Proxy per Floor, Everything Else Passive

[← Back to README](../README.md)

## Why roles
An ESPHome Bluetooth proxy can do two different jobs:
- **Forward advertisements** (passive). This is all Bermuda presence and most BLE sensors need.
- **Make connections** (`bluetooth_proxy: active: true`). Some devices (appliances, personal-care devices) must be connected to and read via GATT, sometimes with pairing.

Connections cost radio time, memory and complexity. Letting every board connect means HA may hand a device to whichever board happens to have the best signal at that moment, so ownership is unpredictable and harder to debug. The fleet was therefore given **explicit roles**, like assigning controllers on a BMS network.

## Fleet (11 ESP32 boards + the host adapter = 12 Bluetooth adapters in HA)

| Floor | Active-connection proxy | Passive proxies (+ discovery logger) | Lean monitoring boards |
|---|---|---|---|
| Ground | Family room / kitchen (interim; moves to the understairs plant board later) | Lounge, play room | Understairs (UFH/cooker monitoring), utility (water metering) |
| 1st | Bedroom 2 (also hosts a dedicated bridge for a device needing LE Secure Connections pairing) | Main bedroom, bedroom 3, office | |
| 2nd | Studio | | |
| n/a | | | Boiler plant board: **no Bluetooth** (fully allocated I/O); camera board: no Bluetooth |

Room names are generic in this repo.

## Profiles
See [config/proxy-profiles.yaml](../config/proxy-profiles.yaml).

| Profile | `bluetooth_proxy: active` | Scan type | Scanner | MQTT |
|---|---|---|---|---|
| A. Active-connection proxy | true | active | yes | scanner only (`discovery: false`) |
| B. Passive proxy + logger | false | active (to capture names) | yes | scanner only (`discovery: false`) |
| C. Lean monitoring board | false | passive | no | **none** |

**Design rules:**
- **Monitoring boards stay lean.** Their primary job is plant or metering data, so they carry no optional radio or MQTT workload. That's the same reason BMS plant controllers aren't used as general-purpose gateways.
- **Passive mode doesn't affect presence.** Bermuda works from advertisements and their RSSI, not connections.
- **Connections go where the devices are.** The ground-floor connection role moves to the understairs board once the ESPHome stability issue ([doc 03](03-incident-esphome-2026-10-watchdog.md)) is fixed, not before: a plant-monitoring board shouldn't take on the riskiest workload.

## Verification
- Bermuda's scanner page: every proxy's last advertisement under about 2 s.
- HA's Bluetooth integration: 12 adapters (11 ESP32 + host).
- Connected devices: the coffee machine connects through the ground-floor connection proxy (poll, read, disconnect about every 3 s); the bridged personal-care device pairs through the bedroom-2 bridge.

# 02 · Discovery Logger v2: Same Output, a Fraction of the Work

[← Back to README](../README.md) · Config: [config/ble-scanner-dedup.yaml](../config/ble-scanner-dedup.yaml)

## Purpose
HA's own Bluetooth discovery cards **disappear when a device moves out of range**, so there's no way to look back and see what was around, and which devices might be worth integrating. The logger keeps an **accumulating, de-duplicated list** (address type, address, name) per board and publishes it every 10 minutes, without writing a constant stream into HA's database.

## v1 → v2

| | v1 (`std::vector<std::string>`) | v2 (`std::map<uint64_t, std::string>`) |
|---|---|---|
| Work per advertisement (tens to hundreds per second) | Build 3-4 strings, then a **linear search** of up to 200 string compares | **One map lookup**; return immediately if already seen |
| Work per new device | Push, and when full **erase the first element**, shifting 199 strings | Build the entry string once; when full, erase one node (O(log n)) |
| Heap behaviour | Constant churn of temporary strings | Allocations only for genuinely new devices |
| Output | `{"devices":["[PUB]AA:BB:…|Name", …]}` + count | **Identical**: same topics, same format |

## Active scanning, deliberately
Many devices only send their **name** in a scan response, which is requested by active scanning. Names are what make the list useful ("that's an appliance with an integration"), so the logger boards keep `active: true` scanning. The lean monitoring boards use passive scanning.

## HA side
- Count and list sensors per board ([config/ha-mqtt-sensors.yaml](../config/ha-mqtt-sensors.yaml)).
- The list sensors are excluded from the recorder with a glob (`sensor.ble_device_list*`), which covers new boards automatically.
- `mqtt: discovery: false` on the ESPs: the scanner uses MQTT only as a transport, so the boards no longer create a duplicate MQTT copy of every entity alongside the native API.

## Lessons
- **Make the common case cheap.** Nearly every advertisement is a repeat, so the fast path must be a lookup, not string work.
- **Key by the natural identifier** (the 64-bit address), not by a formatted string.
- **Behaviour is preserved:** the same topics and payload, so nothing downstream had to change.

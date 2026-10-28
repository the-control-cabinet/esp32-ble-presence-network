# Changelog

## [2.1.1] - 2026-10-28
- Fleet table: VPin cabinet board added (11 ESP32 proxies in total).

## [2.1.0] - 2026-10-27
- Calibration exercise (planned): survey, global ref_power/attenuation, per-receiver offsets, walk test, acceptance criteria, re-calibration triggers (doc 05); record-sheet templates in data/.

## [2.0.0] - 2026-10-27
- Proxy roles: one connection proxy per floor, passive proxies, lean monitoring boards (doc 01, config/proxy-profiles.yaml).
- Discovery logger v2 (std::map; work only for new devices; active scanning for names) (doc 02).
- Incident write-up: task-watchdog resets on ESPHome 2026.10.0, elimination table, upstream reports, status open (doc 03).
- Rename and clean-up runbook: HA reconfigure, Bermuda, MQTT discovery off, retained-message clearing, orphaned entities, recorder glob (doc 04).
- HA sensor and recorder examples updated.

## [1.0.0] - 2026-10-27
- Architecture, room presence (Bermuda, private BLE), on-device de-duplicating scanner (YAML), HA sensors, issues (memory crash, keepalive, JSON attributes), roadmap.

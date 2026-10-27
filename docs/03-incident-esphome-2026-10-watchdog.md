# 03 · Incident: Task-Watchdog Resets on ESPHome 2026.10.0 (Open, Upstream Fix Pending)

[← Back to README](../README.md)

## Symptom
Proxy boards rebooting every few minutes. The device log after each reboot shows:
```
*** CRASH DETECTED ON PREVIOUS BOOT ***
Reason: Task wdt
Crashed core: 1
PC: esp_cpu_wait_for_intr (backtrace: esp_vApplicationIdleHook → prvIdleTask)
```
Built with ESPHome **2026.10.0** on ESP-IDF **5.5.6**, ESP32-S3.

## What the data showed
- **Heap is healthy:** about 150-157 KB free, with ~100 KB blocks available, and stable right up to each reset.
- **The main loop is healthy:** loop time 18-24 ms up to the reset (spikes only at boot).
- **The backtrace is the idle task on core 1.** The watchdog fired while core 1 was idle, which points at a task starving **core 0**, where the Wi-Fi and Bluetooth stacks run, rather than at ESPHome's own loop.
- **Timing:** typically within 1-4 minutes of boot, during API/MQTT/BLE start-up activity (one boot logged `mqtt took a long time (517 ms)` and a 49 s API handshake).

## Elimination (one change at a time)

| Hypothesis | Test | Result |
|---|---|---|
| Heavy per-advertisement scanner lambda | Rewrote it (v2, [doc 02](02-scanner-v2.md)) | Still resets: **not the cause** (also crashed on v1) |
| Memory exhaustion | Debug heap sensors | Flat and plentiful: **not the cause** |
| ESPHome main loop blocking | Debug loop-time sensor | Normal until the reset: **not the cause** |
| MQTT discovery start-up burst | `discovery: false` | Still resets: **not the cause** |
| Active BLE connections | `bluetooth_proxy: active: false` on the worst board | Still resets: **not the cause** on that board |
| Board-specific (renaming, configuration) | Same signature on a second, un-renamed board running different firmware options | **Same crash:** not board-specific config |

## Upstream
The same signature (`Task wdt` in `esp_vApplicationIdleHook`, crashed core 1, ESP32-S3) is reported against **ESPHome 2026.10.0** on the ESPHome issue tracker by several users, including Bluetooth-proxy boards and boards with slow MQTT connects (e.g. esphome/esphome #21716, #21488; a similar report exists on 2026.9.1). A related report traces a stall to the BLE stack bring-up waiting on the Bluetooth task without a timeout.

## Current status: OPEN
- **Working assumption:** a regression in the ESPHome 2026.10.0 / ESP-IDF 5.5.6 Wi-Fi and Bluetooth stack under proxy load, with the worst-affected board also on the weakest Wi-Fi link (−65 to −67 dBm against −48 to −51 dBm elsewhere).
- **Impact:** short presence gaps on the affected floor; the board's discovery list resets on each reboot. No hardware risk: the watchdog is doing its job.
- **Mitigations available (not yet applied):** extend the watchdog (`CONFIG_ESP_TASK_WDT_TIMEOUT_S: "15"`); improve the board's Wi-Fi link; a serial capture to name the starving task.
- **Plan:**
  1. Compare uptime histories across active, passive and lean boards.
  2. When a fixed release lands, update **one board first**, confirm stable uptime, then roll out.
  3. Hold other upgrades meanwhile.

## Lessons
- **Instrument before guessing.** Adding heap and loop-time sensors ruled out three hypotheses within an hour.
- **One variable per flash.** Each test changed exactly one thing.
- **Read the backtrace for *where*, not just *that*.** An idle-task backtrace on the crashed core points away from the application loop.
- **Check upstream early.** Matching a signature to open issues stops you redesigning a working configuration to chase a platform bug.

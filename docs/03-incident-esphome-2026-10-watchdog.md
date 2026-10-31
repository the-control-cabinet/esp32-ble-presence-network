# 03 · Incident: Task-Watchdog Resets on ESPHome 2026.10.x (Open, Upstream Fix Pending)

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

## Update: 48-hour comparison and 2026.10.1 (29-31 Oct 2026)

**Uptime over 48 hours (all on 2026.10.0 at the start):**

| Board | Role | Behaviour |
|---|---|---|
| Six passive boards + control room | Passive / no BLE | Stable (~45 h) |
| Family Room / Kitchen | Active (Nespresso) | **Reset every 7-11 h** |
| Studio | Active (2nd floor) | **Boot-loop bursts** (resets every few minutes for hours), with one stable 25 h stretch |
| 1st-floor active board | Active (shaver bridge) | Stable for 45 h |
| Play Room | Passive (confirmed `active: false`) | Stable; one reset shared with two other boards at the moment 2026.10.1 was installed |

**Findings:**
- **Not memory.** Family Room's heap settles to ~137 KB free / ~85 KB max block after boot and stays flat until each reset: no leak, no fragmentation. Studio's heap is equally flat in its stable stretch and volatile only during boot-loop bursts.
- **Not Wi-Fi** on Family Room (−40 dBm) or Studio (−45 dBm).
- **Connection churn:** Family Room makes an active connection to the coffee machine **about once a minute** (connect, ~4 s, disconnect), all day. The 1st-floor active board, connecting rarely, stays up. Working hypothesis: resets follow **active-connection activity**, not the `active: true` setting.
- **ESPHome 2026.10.x crash capture** confirms **Task wdt on core 1**. Reports marked **"Captured by a different firmware build"** appear straight after an OTA update, on stable passive boards too, with the same PC: they are the **update-time watchdog** and are ignored. Only a crash report **without** that line is evidence of the in-service fault.
- **2026.10.1** does not list a fix for the open upstream issue (#21716).

**Instrumentation added:** a `Reset Reason` debug text sensor on the three active boards (Studio, 1st floor, Family Room) (the heap and loop-time sensors were already present). Not rolled out to stable boards.

**Next:** collect the next in-service crash (decoded backtrace) and reset reasons; then run Studio **passive for 24-48 h** to test the active-connection hypothesis; report results upstream.

## Lessons
- **Instrument before guessing.** Adding heap and loop-time sensors ruled out three hypotheses within an hour.
- **One variable per flash.** Each test changed exactly one thing.
- **Read the backtrace for *where*, not just *that*.** An idle-task backtrace on the crashed core points away from the application loop.
- **Check upstream early.** Matching a signature to open issues stops you redesigning a working configuration to chase a platform bug.

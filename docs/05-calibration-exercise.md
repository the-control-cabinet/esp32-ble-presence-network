# 05 · Calibration Exercise: Measuring Each Receiver and Commissioning Room Presence (Planned)

[← Back to README](../README.md) · Record sheets: [data/receiver-calibration.csv](../data/receiver-calibration.csv) · [data/presence-walk-test.csv](../data/presence-walk-test.csv)

**Status: not yet done.** The proxies were installed and have been working on defaults; no receiver has been measured. This is the commissioning exercise that closes that gap, run like a BMS sensor calibration: survey, reference measurements, offsets, then a functional test with acceptance criteria.

## Why calibrate
Bermuda converts signal strength to distance with a path-loss model:

```
distance = 10 ^ ((ref_power − RSSI) / (10 × attenuation))
```

- **`ref_power`** is the RSSI expected at **1 m**. It's set globally, with a **per-scanner offset** and an optional **per-device override**.
- **`attenuation`** is the environment factor: how fast the signal falls with distance in this house.

Two errors matter:
1. **Receivers differ.** Different boards, enclosures, mounting heights and nearby metal mean the *same* device at the *same* distance gives different RSSI at different proxies. Uncorrected, presence is biased towards the most sensitive receiver: a device next to a "deaf" proxy can be assigned to the room next door.
2. **Absolute distance is wrong** without a measured `ref_power` and `attenuation`, so `max_radius` (how far a device can be from a proxy and still count as being in its area) can't be set meaningfully.

## Equipment
- **A reference transmitter with fixed transmit power.** Ideally a dedicated BLE beacon or tag, or a spare ESP32 set up as a beacon. Phones vary their power and rotate addresses, so they're poor references (use them for the walk test, not the calibration).
- A tape measure, and something to hold the transmitter at proxy height (a box or tripod).
- HA open on the transmitter's Bermuda **Distance** sensor, so you can watch its attributes (per-scanner RSSI).

## Procedure

### Step 0: survey (record in `receiver-calibration.csv`)
For every proxy, record:
- room, mounting height, enclosure type
- anything metal or water within 0.5 m (radiators, mirrors, fish tanks, appliances)
- power source, Wi-Fi RSSI
- the board's role ([doc 01](01-proxy-roles.md))

This often explains outliers before you measure anything.

### Step 1: global reference (one "golden" proxy)
1. Choose a proxy with clear line of sight in an uncluttered room.
2. Place the transmitter **1.0 m** away, at the same height, in the same orientation every time (mark it).
3. Watch the per-scanner RSSI for **2-3 minutes** and record the **median**. Ignore spikes; BLE hops between three advertising channels, which each read slightly differently.
4. Set Bermuda's global **`ref_power`** to that median.
5. Move the transmitter to **2 m, 4 m and 6-8 m** in line of sight. Record the median RSSI at each, and adjust **`attenuation`** until the reported distance matches the tape measure (typically between 2 and 4 indoors).

### Step 2: per-receiver offsets (every proxy)
For **each** proxy in turn:
1. Transmitter at **1.0 m**, same height and orientation as Step 1.
2. Record the median RSSI over 2 minutes.
3. **Offset = golden-proxy RSSI at 1 m − this proxy's RSSI at 1 m.** Enter it as that scanner's offset in Bermuda's configuration.
4. Re-check: with the offset applied, the reported distance at 1 m should read about 1 m (±0.3 m).

Do the active-connection proxies too. Do the unstable proxy (see [doc 03](03-incident-esphome-2026-10-watchdog.md)) **after** its firmware fix, since a board that reboots mid-measurement gives meaningless medians.

### Step 3: functional walk test (record in `presence-walk-test.csv`)
Carry a phone tracked via private BLE, and stand for **60 s** at each test point:
- the centre of every room,
- each doorway and room boundary,
- known problem spots (stair landings, open-plan joins, rooms above and below each other).

Record the **expected area** against **Bermuda's area**, plus how long it took to settle.

### Step 4: tune and re-test
- **`max_radius`:** now that distances are real, set it so a device beyond a sensible room size isn't claimed.
- **Smoothing and timeouts:** trade response speed against flicker at boundaries.
- **Floor bleed:** where the room above or below wins, check the offsets for those two proxies, or move or lower one proxy.
- **Coverage gaps:** points where no proxy is within range indicate where an extra proxy is needed.
- Repeat Step 3 for the points that failed.

## Acceptance criteria
| Measure | Target |
|---|---|
| Distance at 1 m after offsets (every proxy) | 1.0 ± 0.3 m |
| Room centres correct | 100 % |
| Doorways and boundaries correct | ≥ 80 % (boundaries are inherently ambiguous) |
| Correct floor | 100 % at room centres |
| Settle time on entering a room | ≤ 30 s |

## Re-calibration triggers
Moving a proxy, changing its enclosure or power supply, a new board, furniture changes near a proxy, or a firmware change affecting the radio. Re-run Step 2 for that proxy, then a short walk test.

## Beyond this
- Bermuda's current area decision is driven largely by **relative** signal between proxies, which is why per-receiver offsets matter more than the absolute numbers.
- Full 2D/3D positioning with **fingerprinting** (recorded RSSI signatures at known spots) is available in a fork of Bermuda (*BLE Trilateration*). It's a candidate for the 3D-mapping roadmap item once the receivers are calibrated.

## Skills shown
Sensor calibration against a reference, offset derivation, a path-loss model fit, functional testing against acceptance criteria, and documented re-calibration triggers: the same discipline as commissioning BMS temperature or CO₂ sensors.

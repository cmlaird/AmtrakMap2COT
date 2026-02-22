# Amtrak Train Tracker for ATAK

A Node-RED flow that pulls live Amtrak (and VIA Rail) train positions from [TransitDocs](https://transitdocs.com) and pushes them to ATAK as Cursor on Target (CoT) events. Trains are displayed on the ATAK map with color-coded icons reflecting their on-time status.

---

## Features

- Live train positions with heading and speed
- Color-coded icon overlays based on delay status
- Next stop with early/late status in the CoT remarks
- Active Amtrak alert messages displayed inline
- Service disruption flagging
- Automatic 30-second refresh via Node-RED inject node

---

## Color Status Legend

| Color | Meaning |
|-------|---------|
| 🟢 Green | On time (< 5 min late) |
| 🟡 Yellow | Late (5–29 min late) |
| 🔴 Red | Very late (30–59 min late) |
| ⚫ Black | Extremely late (60+ min late) |
| ⚪ Gray | Service disruption reported |

---

## Requirements

- [Node-RED](https://nodered.org/)
- ATAK or WinTAK with network connectivity to your Node-RED instance
- [Generic Icons](https://github.com/) iconset installed in ATAK
  - Iconset UID: `ad78aafb-83a6-4c07-b2b9-a897a8b6a38f`
  - Icon: `Shapes/rail.png`

---

## Node-RED Flow

### Nodes

```
[Inject (30s)] → [HTTP Request] → [Function] → [TCP Out]
```

### Node Configuration

**Inject Node**
- Repeat: `interval`
- Every: `30 seconds`

**HTTP Request Node**
- Method: `GET`
- URL: `https://asm-backend.transitdocs.com/map`
- Return: `a parsed JSON object`

**Function Node**
- Paste the contents of `function.js`

**TCP Out Node**
- Address: Your TAK server or ATAK device IP
- Port: `8089`
- Upload your certs to your server
- Make sure Verify Server is unchecked
---

## CoT Output Format

Each train is emitted as a CoT XML event with type `a-f-G-E-V-T` (friendly ground train):

```xml
<event version="2.0" uid="TRAIN-2026-02-22_AMTK_99" type="a-f-G-E-V-T" ...>
  <point lat="40.742966" lon="-73.946077" hae="9999999.0" ce="9999999.0" le="9999999.0"/>
  <detail>
    <contact callsign="99 - Northeast Regional"/>
    <track course="225" speed="20.81"/>
    <remarks>BOS → NPN | Speed: 46.53 mph | Next Stop: NWK (14 min early) | 🚨 Delay Notification: ...</remarks>
    <usericon iconsetpath="ad78aafb-83a6-4c07-b2b9-a897a8b6a38f/Shapes/rail.png"/>
    <color argb="-256"/>
  </detail>
</event>
```

---

## Delay Detection Logic

Delay status is determined in the following priority order:

1. **Alert text** — parses `"approximately X minutes late"` directly from Amtrak's alert messages
2. **Last ACTUAL departure variance** — reads the most recently departed stop's actual vs. scheduled time
3. **Next ESTIMATED stop variance** — fallback for trains early in their journey with no actuals yet

---

## Thresholds

Thresholds are defined at the top of `function.js` and can be adjusted:

```javascript
const LATE_MIN = 5;           // Yellow
const VERY_LATE_MIN = 30;     // Red
const EXTREMELY_LATE_MIN = 60; // Black
```

---

## Data Source

Train positions and schedule data are sourced from the TransitDocs public API:

```
https://asm-backend.transitdocs.com/map
```

This endpoint returns live position, speed, heading, stop-level variance, and delay alerts for all active Amtrak and VIA Rail trains.

---

## Notes

- Trains without a `location` field are filtered out and not sent as CoT events
- The CoT stale time is set to 120 seconds — if the inject interval is changed, update `staleCot()` accordingly
- VIA Rail trains are included in the feed and will appear on the map alongside Amtrak trains

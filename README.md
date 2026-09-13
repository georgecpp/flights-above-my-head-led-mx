# Flights Above My Head LED Matrix

A self-hosted LED flight-information display that detects aircraft passing above a configured location and shows detailed flight, route, timing and aircraft information.

The system runs its backend on a OnePlus Nord AC2003 K3s node and sends a compact display model to an ESP32-controlled HUB75 RGB LED matrix.

## Project goals

For each relevant aircraft, the display should show:

* commercial flight number;
* aircraft callsign;
* airline;
* departure airport;
* destination airport;
* scheduled and actual departure times;
* scheduled and estimated arrival times;
* calculated flight time remaining;
* flight progress;
* aircraft type and model;
* aircraft registration;
* current altitude;
* current speed and heading;
* distance from the configured location;
* whether the aircraft is approaching, overhead or departing.

The system must continue behaving predictably when flight information is incomplete, the provider is unavailable or the local network is temporarily disconnected.

## System architecture

```text
Flightradar24 flight data
           |
           v
+-----------------------------+
| Flight-data provider        |
|                             |
| - geographic search         |
| - detailed flight lookup    |
| - response normalization    |
+-----------------------------+
           |
           v
+-----------------------------+
| Overhead flight engine      |
|                             |
| - distance calculation      |
| - closest-approach estimate |
| - flight prioritization     |
| - state classification      |
+-----------------------------+
           |
           v
+-----------------------------+
| Display API                 |
|                             |
| - versioned JSON contract   |
| - cached last-known state   |
| - health and stale status   |
+-----------------------------+
           |
           +----------------------+
           |                      |
           v                      v
+--------------------+   +--------------------+
| Browser simulator  |   | ESP32 firmware     |
|                    |   |                    |
| Pixel-accurate     |   | HUB75 rendering    |
| matrix preview     |   | Wi-Fi client       |
+--------------------+   +--------------------+
                                  |
                                  v
                         +--------------------+
                         | RGB LED matrix     |
                         +--------------------+
```

## Deployment architecture

The application backend is packaged as an ARM64 OCI container and deployed to the home K3s cluster.

```text
GitHub repository
       |
       v
GitHub Actions
       |
       v
ARM64 image in GHCR
       |
       v
homelab-gitops
       |
       v
K3s on OnePlus Nord
       |
       v
ESP32 on local network
```

Infrastructure is maintained separately:

* [`nord-control-plane`](https://github.com/georgecpp/nord-control-plane) provisions the OnePlus Nord, Linux userspace and K3s cluster.
* [`homelab-gitops`](https://github.com/georgecpp/homelab-gitops) contains the Kubernetes deployment configuration.
* This repository contains only the application, simulator, firmware and hardware documentation.

## Flight-data strategy

The initial implementation uses publicly accessible Flightradar24 endpoints commonly used by personal and educational flight-display projects.

The integration works in two stages.

### 1. Nearby-flight discovery

A geographic bounding box around the configured location is queried to discover active aircraft and their Flightradar24 identifiers.

The initial implementation is expected to use a request based on:

```text
https://data-cloud.flightradar24.com/zones/fcgi/feed.js
```

### 2. Flight-detail enrichment

After selecting the most relevant aircraft, its Flightradar24 identifier is used to retrieve detailed information from:

```text
https://data-live.flightradar24.com/clickhandler/?flight={flight_id}
```

The detailed response can provide information such as:

* flight number and callsign;
* airline;
* origin and destination;
* aircraft type and registration;
* scheduled departure and arrival;
* actual departure;
* estimated arrival;
* altitude, position, speed and heading.

Time remaining, flight progress, distance and closest approach are calculated locally.

## Provider status

The initial Flightradar24 integration is unofficial and may change or stop working without notice.

To contain that risk:

* provider-specific code is isolated behind an internal interface;
* raw provider objects never reach the firmware;
* missing fields are treated as normal;
* schema changes are detected through tests and monitoring;
* the last valid display model can be retained temporarily;
* an official Flightradar24 API provider can be added later without changing the display protocol.

The project is intended for personal and educational use. Users are responsible for complying with the applicable [Flightradar24 Terms of Service](https://www.flightradar24.com/terms-of-service).

## Flight-selection logic

The application does not simply display the geographically nearest aircraft.

For every candidate, the backend will:

1. reject grounded aircraft;
2. reject stale or invalid positions;
3. calculate horizontal distance from the configured location;
4. derive a motion vector from heading and ground speed;
5. estimate the closest point of approach;
6. determine whether the aircraft is approaching or departing;
7. rank candidates by proximity, trajectory and data freshness;
8. retain the selected flight briefly to prevent display flicker.

Possible flight states are:

```text
APPROACHING
OVERHEAD
DEPARTING
STALE
NO_FLIGHT
PROVIDER_OFFLINE
```

Selection radius, closest-approach threshold and retention time are configurable.

## Timing semantics

The application distinguishes between scheduled, actual and estimated times.

| Field                    | Meaning                                   |
| ------------------------ | ----------------------------------------- |
| `scheduled_departure_at` | Published departure time                  |
| `actual_departure_at`    | Actual or detected takeoff/departure time |
| `scheduled_arrival_at`   | Published arrival time                    |
| `estimated_arrival_at`   | Current estimated arrival                 |
| `remaining_seconds`      | Estimated arrival minus current time      |
| `elapsed_seconds`        | Current time minus actual departure       |
| `progress_percent`       | Estimated completed portion of the flight |

All timestamps are stored and transmitted in UTC.

The matrix converts them to the configured display timezone, for example:

```text
Europe/Bucharest
```

When an actual or estimated timestamp is unavailable, the field is omitted rather than replaced with misleading information.

## Display design

The preferred hardware target is a 64×64 HUB75 RGB matrix because it can show a complete flight card.

A 64×32 panel can also be supported using rotating pages.

### Example 64×32 sequence

Page 1:

```text
RO 703
IAS > OTP
```

Page 2:

```text
DEP 19:42
ETA 20:28
```

Page 3:

```text
LEFT 00:16
B738 YR-BGM
```

Page 4:

```text
35,000 FT
8.2 KM NW
```

### Idle display

When no relevant aircraft is detected, the matrix may show:

* current time;
* a scanning animation;
* server and network status;
* the most recent flight count;
* a configurable idle message.

### Error display

Provider or network failures should produce a clear degraded state rather than a blank screen.

Examples:

```text
DATA STALE
LAST 2M AGO
```

```text
SERVER
OFFLINE
```

## Display API

The ESP32 consumes a small, versioned, provider-independent JSON document.

Example:

```json
{
  "schema": 1,
  "generated_at": "2026-09-13T17:55:00Z",
  "expires_at": "2026-09-13T17:55:45Z",
  "display_timezone": "Europe/Bucharest",
  "state": "APPROACHING",
  "flight": {
    "provider_id": "38a384da",
    "flight_number": "RO703",
    "callsign": "ROT703",
    "airline": {
      "name": "TAROM",
      "iata": "RO",
      "icao": "ROT"
    },
    "origin": {
      "name": "Iasi International Airport",
      "iata": "IAS",
      "icao": "LRIA"
    },
    "destination": {
      "name": "Henri Coanda International Airport",
      "iata": "OTP",
      "icao": "LROP"
    },
    "scheduled_departure_at": "2026-09-13T16:35:00Z",
    "actual_departure_at": "2026-09-13T16:42:00Z",
    "scheduled_arrival_at": "2026-09-13T17:25:00Z",
    "estimated_arrival_at": "2026-09-13T17:28:00Z",
    "remaining_seconds": 1980,
    "elapsed_seconds": 4380,
    "progress_percent": 69,
    "aircraft": {
      "type": "B738",
      "model": "Boeing 737-800",
      "registration": "YR-BGM",
      "icao24": "4A08F2"
    },
    "position": {
      "latitude": 47.1234,
      "longitude": 27.5678,
      "altitude_ft": 35000,
      "ground_speed_kts": 447,
      "heading_deg": 284,
      "distance_km": 8.2,
      "closest_approach_km": 1.7,
      "closest_approach_seconds": 52
    }
  }
}
```

The example is synthetic and is not a captured Flightradar24 response.

## Planned technology

### Backend

* Python
* FastAPI
* Pydantic
* HTTPX
* pytest
* ARM64 OCI container
* Kubernetes health probes

### Firmware

* ESP32
* C++
* PlatformIO
* HUB75 DMA display driver
* Wi-Fi configuration
* HTTP client
* watchdog and reconnect handling

### Simulator

* TypeScript
* Vite
* HTML Canvas
* pixel-accurate 64×32 and 64×64 layouts
* playback of synthetic flight scenarios

## Repository structure

```text
.
├── backend/
│   ├── src/
│   │   ├── api/
│   │   ├── display/
│   │   ├── domain/
│   │   ├── providers/
│   │   │   └── flightradar24/
│   │   └── selection/
│   └── tests/
├── firmware/
│   ├── include/
│   ├── src/
│   ├── test/
│   └── platformio.ini
├── simulator/
│   ├── src/
│   └── tests/
├── shared/
│   ├── fixtures/
│   └── schemas/
├── hardware/
│   ├── bom.md
│   ├── wiring.md
│   └── enclosure/
├── docs/
│   ├── architecture/
│   └── decisions/
├── .github/
│   └── workflows/
└── README.md
```

## Reliability requirements

The backend must:

* apply connection and response timeouts;
* poll at a conservative configurable interval;
* retrieve details only when necessary;
* cache selected-flight enrichment;
* tolerate missing optional fields;
* retry transient failures with exponential backoff and jitter;
* detect stale position data;
* retain the previous valid display briefly;
* avoid rapidly switching between similarly ranked flights;
* expose readiness, liveness and provider-health endpoints;
* avoid logging credentials or precise coordinates.

The firmware must:

* reconnect to Wi-Fi automatically;
* reconnect to the backend automatically;
* reject unsupported schema versions;
* validate required payload fields;
* continue displaying the last valid payload until it expires;
* clearly indicate stale or offline data;
* use a hardware or software watchdog;
* recover without manual intervention after power loss.

## API efficiency

The application minimizes external requests by:

* querying only a small geographic area;
* using a configurable polling interval;
* enriching only the selected flight;
* caching flight details by provider flight ID;
* caching stable airport, airline and aircraft metadata;
* calculating countdowns locally;
* avoiding repeated enrichment while the selected flight is unchanged;
* backing off after errors or rate limiting.

## Configuration

Runtime configuration will be supplied using environment variables or Kubernetes secrets.

Planned configuration:

```text
HOME_LATITUDE
HOME_LONGITUDE
SEARCH_RADIUS_KM
OVERHEAD_THRESHOLD_KM
PREDICTION_WINDOW_SECONDS
FLIGHT_RETENTION_SECONDS
FR24_POLL_INTERVAL_SECONDS
DISPLAY_TIMEZONE
DISPLAY_WIDTH
DISPLAY_HEIGHT
LOG_LEVEL
```

Coordinates and credentials must never be committed.

An `.env.example` file will document required values using non-sensitive placeholders.

## Privacy and test-data policy

The public repository must not contain:

* exact installation coordinates;
* Wi-Fi credentials;
* API credentials;
* internal IP addresses;
* Kubernetes credentials;
* device identifiers;
* unmodified raw provider responses.

Automated tests use manually created synthetic flight data.

If real responses are used temporarily during development, they must remain outside Git and be removed after the required analysis.

## Development roadmap

### Phase 0 — Infrastructure

* [ ] Recover the OnePlus Nord AC2003
* [ ] Install and verify LineageOS
* [ ] Enable container support
* [ ] Install the single-node K3s cluster
* [ ] Establish the GitOps deployment path

### Phase 1 — Flight-data spike

* [ ] Query nearby aircraft using a test bounding box
* [ ] retrieve detailed information for one flight ID
* [ ] identify consistently available fields
* [ ] document missing-field behavior
* [ ] create synthetic response fixtures
* [ ] establish a conservative polling interval

### Phase 2 — Backend domain

* [ ] Define the provider-independent `Flight` model
* [ ] Implement the Flightradar24 adapter
* [ ] Implement time normalization
* [ ] Implement distance calculations
* [ ] Implement closest-approach prediction
* [ ] Implement candidate ranking and retention
* [ ] Implement caching and failure handling

### Phase 3 — Display contract

* [ ] Define JSON Schema for display API version 1
* [ ] Implement the current-flight endpoint
* [ ] Implement health endpoints
* [ ] Implement stale and offline states
* [ ] Add contract tests

### Phase 4 — Simulator

* [ ] Build the 64×32 simulator
* [ ] Build the 64×64 simulator
* [ ] Implement rotating display pages
* [ ] Implement synthetic scenario playback
* [ ] Finalize fonts, colors and animations

### Phase 5 — Hardware

* [ ] Select the LED panel
* [ ] Select the ESP32 controller and HUB75 adapter
* [ ] Validate the power supply
* [ ] Document wiring
* [ ] Implement firmware networking
* [ ] Implement matrix rendering
* [ ] Validate watchdog and reconnect behavior
* [ ] Design the enclosure

### Phase 6 — Deployment

* [ ] Build the ARM64 backend image
* [ ] Publish it to GHCR
* [ ] Add Kubernetes manifests to `homelab-gitops`
* [ ] Deploy the backend to the Nord
* [ ] Connect the ESP32 to the deployed backend
* [ ] Test reboot and network-failure recovery

## Inspiration

This project is inspired by open-source personal flight displays including:

* [smartbutnot/flightportal](https://github.com/smartbutnot/flightportal)
* [rzeldent/esp32-flightradar24-ttgo](https://github.com/rzeldent/esp32-flightradar24-ttgo)
* [cfbender/aerovision](https://github.com/cfbender/aerovision)
* [JeanExtreme002/FlightRadarAPI](https://github.com/JeanExtreme002/FlightRadarAPI)

## Status

Repository planning complete.

Development begins after the initial OnePlus Nord control-plane bootstrap.

# IoT Cyber Range

A virtual IoT security lab: simulated ESP32 devices (Wokwi) representing a
facility access-control system, communicating over MQTT, with a controlled
attack → detection → alert → response pipeline built around Docker Compose.

## Concept

Three IoT devices simulate a small facility access-control system:

- **ESP32 #1 — Badge/RFID controller** (`badge-controller`, completed):
  publishes access events to `facility/badge1/access`
- **ESP32 #2 — Motion sensor** (`motion-sensor`, planned): publishes readings
  under `facility/motion1/`
- **ESP32 #3 — Lock/siren actuator** (`lock-actuator`, planned): subscribes
  to commands on `facility/lock1/cmd` to lock/unlock or trigger an alarm

All devices communicate over MQTT through a Dockerized Mosquitto broker.

The security layer provides a controlled **attacker** environment, a
non-executing **honeypot**, a rule-based **detection** service, local event
storage and a live **dashboard**. An operator can request a broker-account
block; applying it updates the local ACL and restarts the broker.

Full pipeline target: IoT devices → attack → monitoring → detection →
logging → alert → investigation → optional response.

## Architecture

[Attacker container] --> [MQTT Broker (Mosquitto, Docker, localhost only)]
^ ^ ^
[ESP32 #1] [ESP32 #2] [ESP32 #3] (Wokwi)
(done) (planned) (planned)
|
v
               [Security monitor (IDS subscriber + local SQLite/API)]
                         ^             ^
                         |             |
                  [Broker audit]   [Honeypot event log]
                         |
                   [Dashboard]


## Components

- **esp32/** — Wokwi projects, one folder per device
  - `badge-controller/` — **completed**: push button (GPIO4) simulates a
    badge scan, red LED (GPIO2), PubSubClient library, publishes to
    `facility/badge1/access`
  - `motion-sensor/`, `lock-actuator/` — **not started yet**
- **Broker/** — `mosquitto.conf`, `acl.conf`, `passwordfile`
- **Backend/** — MQTT subscriber + API (**not started yet**)
- **Lab/** — isolated honeypot, IDS, local security dashboard and response script
- **Attacker/** — on-demand lab-only MQTT and honeypot exercises
- **screenshots/** — evidence figures referenced in `Notes.md`
- **docker-compose.yml** — orchestrates the broker, honeypot and security
  monitor; the teammate-owned application backend remains separate

## Tech stack

- ESP32 simulation: Wokwi
- ESP32 firmware library: PubSubClient (Arduino)
- MQTT broker: Eclipse Mosquitto (Dockerized), username/password auth + ACLs
- Public tunnel: bore (open-source, bore.pub relay)
- Backend: teammate-owned; not implemented in this security layer
- Event storage: local SQLite used by the IDS/dashboard service
- Detection: Python rule-based MQTT and log monitor using Paho MQTT
- Honeypot: bounded fake TCP maintenance console; input is recorded, never executed
- Dashboard: local HTML/JavaScript event and alert feed
- Attacker tooling: Python/Paho scripts restricted to the Compose lab network

## Known technical constraint

Wokwi's simulated ESP32 network is not on the same LAN as our Docker
containers — it needs a real, reachable broker address. We expose the
Dockerized Mosquitto broker via bore (https://github.com/ekzhang/bore), an
open-source TCP tunnel to the public bore.pub relay, so Wokwi firmware can
connect to it over the internet, entirely free and with no payment
information required (we moved off ngrok because its free tier now requires
a credit/debit card for TCP tunnels). Note: bore.pub URLs/ports change on
every restart — update the address in the ESP32 sketch each session.
Forwarded traffic is not encrypted by default (per bore's own
documentation) — acceptable for this development/demo context, not
intended as production-secure transport. The bore.pub free relay has shown
intermittent connection instability under testing — documented as a known
limitation of the free community tunnel.

Each device's MQTT credentials (e.g. `badge1`) are scoped via ACLs to only
publish on that device's own topic. Since the `badge-controller` Wokwi
project may be shared/public, this credential is treated as low-privilege
by design rather than secret — leaking it only allows fake badge-scan
events, nothing more.

The security lab network is Docker-internal. The MQTT port and dashboard
are published only on `127.0.0.1`; the attacker and honeypot ports are not
published to the host or internet. The command-injection exercise sends
command-like text to a fake service that records it without invoking a
shell. This is a teaching lab, not a production IDS or containment system.

## Setup

1. Create the local broker users with `Broker/add-user.ps1`; save the
   printed credentials in the git-ignored `Broker/passwords.txt`. Preserve
   the `badge1` password configured in the Wokwi sketch.
2. Create the monitor's least-privilege secret from the `ids` entry in
   `Broker/passwords.txt`:

   ```powershell
   $line = Get-Content .\Broker\passwords.txt |
       Where-Object { $_ -like 'ids:*' } | Select-Object -First 1
   if (-not $line) { throw "No ids credential found." }
   $idsPassword = ($line -split ':', 2)[1]
   [System.IO.File]::WriteAllText(
       (Join-Path (Get-Location) 'Broker\ids.password'),
       $idsPassword,
       [System.Text.UTF8Encoding]::new($false)
   )
   ```

3. Start the broker, honeypot, and IDS/dashboard:

   ```powershell
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30 mqtt-broker security-monitor honeypot
   ```

4. Open `http://127.0.0.1:8080` for the event and alert dashboard. It is
   separate from the teammate-owned application backend.
5. Run a bounded exercise from the repository root:

   ```powershell
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Lab\run-scenario.ps1 -Scenario recon
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Lab\run-scenario.ps1 -Scenario command-injection
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Lab\run-scenario.ps1 -Scenario sensor-spoof
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Lab\run-scenario.ps1 -Scenario lateral-movement
   ```

   These scripts contact only the internal Compose broker and honeypot.
   The recon scenario checks that anonymous MQTT access is rejected; the
   command-injection string is logged by a fake service and never run.
6. To apply a dashboard block request, start the host-side watcher in a
   separate PowerShell window. It edits the lab ACL and restarts Mosquitto:

   ```powershell
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Lab\response.ps1 -Action Watch
   ```

   Or block/unblock one local account directly with `-Action Block
   -User motion1` / `-Action Unblock -User motion1`. A broker restart
   disconnects all lab clients. Stop the watcher with Ctrl+C.
7. When needed for Wokwi, expose the localhost-bound broker through Bore
   from the host with `bore local 1883 --to bore.pub`; use the current
   `bore.pub:<PORT>` in the sketch. Do not publish Docker's MQTT port on
   all host interfaces.

## Status

- [x] Repo structure created
- [x] `mosquitto.conf` written (listener 1883, ACL + password file referenced)
- [x] `acl.conf` written (per-device topic restrictions)
- [x] Mosquitto added to `docker-compose.yml` + tested locally (pub/sub confirmed)
- [x] Real MQTT authentication added (`passwordfile`, `allow_anonymous false`)
- [x] Public tunnel set up (bore.pub) and externally tested with auth/ACLs
- [x] First ESP32 (badge-controller) built and publishing to broker through bore
- [ ] Remaining 2 ESP32 firmwares (motion-sensor, lock-actuator)
- [ ] Backend event ingestion (teammate-owned)
- [x] Attacker container + bounded recon exercise
- [x] Detection rules (unauthorized MQTT, command-like honeypot input, sensor anomalies)
- [x] Honeypot integrated into local event logging
- [x] Sensor-spoofing and honeypot-to-broker lateral-movement scenarios
- [x] Basic local dashboard (live event feed + alerts)
- [x] Response action (operator-approved local account block)
- [x] Controlled lab integration tests (attacks, detections, dashboard, response)
- [ ] Full end-to-end test with the remaining devices and teammate-owned backend
- [ ] Demo rehearsal
- [ ] Report written

See `Notes.md` for the detailed, dated build log.


## Team / Work Distribution

**Ghada — Infrastructure & Cybersecurity Lead**

Already completed (core project infrastructure):
- Designed the overall IoT Cyber Range architecture
- Created the repository structure
- Wrote `Broker/mosquitto.conf` (MQTT listener on port 1883, ACL and
  password file references)
- Wrote `Broker/acl.conf` with per-device MQTT topic restrictions
- Added Mosquitto to `docker-compose.yml` and tested local MQTT pub/sub
  communication
- Implemented real MQTT authentication with `passwordfile` and disabled
  anonymous access (`allow_anonymous false`)
- Set up the public Bore TCP tunnel and externally tested MQTT
  communication through it with authentication and ACLs
- Built the first ESP32, `badge-controller`, and verified end-to-end MQTT
  publishing through Bore

Remaining work:
- Attack/defense testing against all three device firmwares and the teammate-owned backend
- Final security integration and demo rehearsal

**Arij — IoT Device & Data Lead**

- Implement the remaining two ESP32 firmwares (`motion-sensor`,
  `lock-actuator`) and their MQTT integration
- Device-side testing and integration
- Backend MQTT event ingestion, database schema, and event
  storage/processing
- Integration of all three ESP32 devices with the backend


## Timeline (6 weeks)

- Week 1: repo + broker + first ESP32 connectivity — **done**
- Week 2: all 3 ESP32 firmwares + backend ingesting events — **in progress
  (badge-controller done)**
- Week 3: attacker container + recon attack + first detection rule + basic dashboard
- Week 4: command-injection attack + honeypot
- Week 5: sensor-spoofing scenario + anomaly rule + lateral-movement correlation
- Week 6: response action, polish, demo rehearsal, buffer
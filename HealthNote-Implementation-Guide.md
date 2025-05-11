# HealthNote Watch – Implementation Guide (v0.1)

_This living document tracks design decisions, task breakdowns, and progress for the **HealthNote** fork of Espruino.  Update it as milestones are reached or requirements evolve._

---
## 1. Vision & Scope

**Goal:** Ship a privacy-preserving smartwatch firmware (codename `HEALTHNOTE_OS-0.1`) that collects core health metrics and publishes them to Nostr using the emerging **NIP-101h** family of specs.

* **Hardware** – NRF52840-based watch (Bangle.js 2 reference design)
* **Metrics v0.1** – Heart-rate, step count, battery level
* **Firmware role** – Capture → encrypt sensitive values (NIP-44) → sign → hand off to phone via BLE
* **Phone companion** – Sync buffer, verify, (optionally) re-sign with user key, and relay to Blossom/private/public Nostr relays

_Out of scope_: Music, LoRa/Meshtastic, Lightning, Weather, etc.

---
## 2. Architecture Snapshot

```mermaid
flowchart TD
    subgraph Watch
        S1[Sensor drivers] --> EN[NIP-101h encoders]
        EN --> CR[NIP-44 encryption]
        CR --> SG[Device-key signature]
        SG --> BUF[24 h ring buffer]\nStorage
        BLE[BLE GATT service] -->|EVENTS characteristic| PH
    end
    subgraph Phone
        PH[HealthNote companion]
        PH --> VR[Verify + optional re-sign]
        VR --> RL[Relay dispatcher]
        RL -->|wss| Blossom & public relays
        VR --> SDK[HealthNote Stats SDK]
    end
```

**Split-key + NIP-26 Delegation**: The watch owns a *device key* and stores a delegation certificate (signed by the user's key) that authorises kinds 32000-32999 for a multi-year window. Every event includes the `delegation` tag, so relays treat it as originating from the user. No pubkey rewriting is needed after pairing.

---
## 3. Milestones & Status

| ID | Description | Owner | ETA | Status |
|----|-------------|-------|-----|--------|
| M1 | Fork + rebrand; firmware boots (`HEALTHNOTE_OS-0.1`) | FW |  | 🚧 |
| M2 | Blank firmware (stock apps stripped) boots | FW |  | ☐ |
| M3 | Sensors stream into Storage as NIP events | FW |  | ☐ |
| M4 | Keypair generated, NIP-44 encrypt + sign works | FW |  | ☐ |
| M5 | BLE service enumerates (`HealthNote Sync`) | FW |  | ☐ |
| M6 | Phone drains buffer & publishes to relay | App |  | ☐ |
| M7 | Minimal UI & QR pairing screen | FW |  | ☐ |

_Update the table as milestones progress._

---
## 4. Firmware Work Breakdown

### 4.1 Rebrand & Cosmetic Tweaks
* Update `boards/BANGLEJS2.py`:
  * `VERSION` → `HEALTHNOTE_OS-0.1`
  * Support URL → `https://healthnote.watch/help`
* Replace boot bitmap (240×240 mono, ≤ 7 kB)

### 4.2 Strip Default Apps
* Remove all `.info` except `boot.js`, `settings.js`
* Expected flash savings ≈ 200 kB

### 4.3 Metric Encoders (NIP-101h)
| Metric | Sensor callback | NIP kind | Required tags | Encrypted field(s) |
|--------|-----------------|----------|---------------|--------------------|
| Steps  | `Bangle.on('step')` every event | **32011** | `d`, `measurement_value`, `measurement_time`, `unit_standard` | `measurement_value` |
| Heart-rate | `Bangle.on('HRM')` every 2 s | **32013** | `d`, `measurement_value`, `measurement_time`, `unit_standard` | `measurement_value` |
| Battery | `E.getBattery()` every 10 min | **32090** | `d`, `measurement_value`, `measurement_time`, `unit_standard` | _none_ (non-private) |

*Encoder template (pseudo-JS)*
```js
function encodeMetric(kind, val, unit) {
  return {
    kind,
    created_at : Math.floor(Date.now()/1000),
    tags : [
      ["d", E.toUUID(16)],
      ["measurement_value", String(val), unit],
      ["measurement_time", String(Date.now()/1000|0)],
      ["unit_standard", unit==="bpm"?"metric":"other"],
      ["privacy_level","private"],
      ["v","1.0.0"]
    ],
    content:"",
    pubkey: DEVICE_PUBKEY
  };
}
```

### 4.4 Key Management & Encryption
* **Device key** generated on first boot; stored in internal flash (encrypted with 6-digit PIN)
* **NIP-26 Delegation** created by the companion on first pairing; certificate is stored plain in flash and appended to every event
* **ECDH** between device-privkey ↔ *user-pubkey* (from delegation) yields shared secret for encryption
* **NIP-44** (XChaCha20-Poly1305) applied only to sensitive tag values

### 4.5 BLE GATT `HealthNote Sync` Service
* Service UUID `16B11410-0000-1000-8000-00805F9B34FB`
| Char | UUID suffix | Properties | Length | Description |
|------|-------------|------------|--------|-------------|
| PUBKEY | `1141` | Read | 33 B | Device compressed pubkey |
| EVENTS | `1142` | Indicate | ≤ 20 B | TLV chunks of signed events |
| CTRL   | `1143` | Write | ≤ 20 B | Opcodes: `SYNC`,`CLEAR`,`PING`,`SET_PK` |
| METRIC_FMT | `1144` | Read | 8 B | Bitmask of supported metrics |
| CONF   | `1145` | Write | TLV | Sampling/interval overrides |

*All characteristics require **encryption, no MITM** (see `BLE_GAP_CONN_SEC_MODE_SET_ENC_NO_MITM`).*

### 4.6 Minimal UI
* Default watch face: time + icons (steps, HR) + current values
* BTN1/BTN2 scroll pages: Summary • Sensors • Sync status • Settings
* Long-press BTN3 → QR with device pubkey + firmware hash

---
## 5. Companion App Responsibilities
1. BLE scan → discover `HealthNote Sync`
2. Read `PUBKEY`; store mapping in secure storage
3. Subscribe to `EVENTS`; assemble TLV stream
4. Verify device signature & delegation validity
5. If delegation missing or expired, prompt user → create new delegation, write via `CTRL:SET_PK`
6. Decrypt sensitive tags with NIP-44 shared secret
7. Feed decrypted JSON into `@healthnotelabs/analytics-sdk`
8. Relay raw event to: `wss://blossom.healthnote.watch` + user-configured relays
9. On success, write `CTRL:CLEAR` to prune watch buffer

---
## 6. Testing & Validation
* **Unit tests** for encoder & encryption (run under Espruino emulator)
* **Integration**: nRF Connect to verify GATT service & security level
* **End-to-end**: Publish to test relay, read back via Nostr client; ensure encrypted payloads cannot be decoded without key

---
## 7. Open Questions / TBD
* Final decision on *device-only* vs *user-level* signing – revisit M4.
* Storage pruning policy if phone unavailable (overwrite oldest? warn user?)
* Battery impact of 2 s HRM interval – might auto-throttle during low battery.

---
## 8. Change Log
* **2025-05-11** – Adopted NIP-26 delegation flow; guide updated accordingly.

---
**Contributing:**  Submit PRs against this file for any spec changes, implementation notes, or progress updates. 
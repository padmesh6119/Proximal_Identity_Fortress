# NFC//VAULT

> Zero-password physical authentication. Tap a card. You're in.

![ESP32](https://img.shields.io/badge/ESP32-WROOM--32-blue?style=flat-square&logo=espressif)
![SHA256](https://img.shields.io/badge/Crypto-SHA--256-green?style=flat-square)
![Supabase](https://img.shields.io/badge/Backend-Supabase-3ECF8E?style=flat-square&logo=supabase)
![NFC](https://img.shields.io/badge/NFC-ISO%2FIEC%2014443-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)

---

## What Is This?

NFC//VAULT replaces the username/password login form with a physical NFC card tap.

No form. No credentials. No database of passwords to breach. No phishing surface.

**Tap once → enroll. Tap again → authenticate. Session opens in under 100ms.**

The raw card UID never leaves the hardware. The ESP32 hashes it on-device with SHA-256 via mbedtls. The server only ever sees the hash.

**Live demo → [pif-green.vercel.app](https://pif-green.vercel.app)**

---

## Stats

| Metric | Value |
|---|---|
| Read latency | < 100ms |
| Passwords stored | 0 |
| Token algorithm | SHA-256 (mbedtls, on-device) |
| Phishing surface | None |
| Session TTL | 300 seconds |

---

## System Architecture

```
┌─────────────┐    I2C 400kHz    ┌──────────────┐
│  NFC Card   │ ───────────────► │  PN532 Reader│
│  (UID only) │                  └──────┬───────┘
└─────────────┘                         │ raw UID
                                        ▼
                                ┌──────────────┐
                                │    ESP32     │  SHA-256(UID) via mbedtls
                                │              │  ← raw UID never leaves here
                                └──────┬───────┘
                                       │ hashed token (serial USB)
                                       ▼
                                ┌──────────────┐
                                │ Python Bridge│  serial → HTTPS
                                └──────┬───────┘
                                       │ POST → Supabase
                                       ▼
                                ┌──────────────┐
                                │   Supabase   │  PostgreSQL + Realtime WS
                                └──────┬───────┘
                                       │ WebSocket broadcast
                                       ▼
                                ┌──────────────┐
                                │  Dashboard   │  unlock · TTL · audit log
                                └──────────────┘
```

### Hardware Layer

| Component | Spec | Role |
|---|---|---|
| ESP32-WROOM-32 | Dual-core 240MHz | Edge auth controller |
| PN532 | I2C @ 400kHz | NFC reader at 13.56MHz |
| mbedtls | SHA-256 engine | On-device hashing |
| NFC Card | ISO/IEC 14443-A | MIFARE Classic or Ultralight |

### Software Layer

| Component | Tech | Role |
|---|---|---|
| Python Bridge | pyserial + supabase-py | Serial → Supabase relay |
| Supabase | PostgreSQL + Realtime | Token store + event broadcast |
| Web Dashboard | Vanilla JS + Canvas | Live session UI |
| Identity Ghost | Particle system | Deterministic card visualization |
| Globe Canvas | HTML5 Canvas | Auth event ping map |

---

## Security Model

### Hard Properties

| Property | How It's Enforced |
|---|---|
| No raw UID transmission | SHA-256 computed on ESP32. Serial output is hex digest only. |
| No password database | `enrolled_tokens` stores hashes only. No plaintext secret. |
| No phishing surface | No login form exists. Nothing to spoof. |
| Session TTL | Sessions expire after 300s. Timeout logged to Supabase. |
| Card revocation | Revoked cards rejected at bridge layer. |

### Known Limitations (Prototype)

> This is an academic prototype. These would need to be addressed before production.

- **Replay attacks** — No challenge-response nonce. A captured token could be replayed within the TTL window.
- **Card cloning** — If a UID is leaked externally, a cloned card authenticates. Physical card security is assumed.
- **No mTLS on bridge** — Python bridge uses standard HTTPS. Production should use mutual TLS.
- **No dashboard auth** — Anyone with the URL can view the audit log and enrolled cards.

---

## Hardware Setup

### Bill of Materials

| Component | Qty | Notes |
|---|---|---|
| ESP32-WROOM-32 dev board | 1 | Any ESP32 with USB-serial |
| PN532 NFC Module | 1 | Set DIP switches to I2C before wiring |
| NFC Cards or Fobs | 2+ | MIFARE Classic 1K or Ultralight |
| Jumper wires (F-F) | 4 | For breadboard |
| USB cable | 1 | Power + serial |

### Wiring (PN532 → ESP32)

| PN532 Pin | ESP32 Pin | Notes |
|---|---|---|
| VCC | 3.3V | Do NOT use 5V |
| GND | GND | Common ground |
| SDA | GPIO 21 | I2C data (ESP32 default) |
| SCL | GPIO 22 | I2C clock (ESP32 default) |

> **Important:** Set PN532 DIP switches to I2C mode **before** powering on.
> SW1 = OFF, SW2 = ON

---

## Firmware

### Dependencies

- Arduino framework for ESP32 (PlatformIO or Arduino IDE)
- [Adafruit PN532 library](https://github.com/adafruit/Adafruit-PN532) — install via Library Manager
- `mbedtls` — bundled with ESP-IDF/Arduino-ESP32, no separate install needed

### SHA-256 On-Device

```c
mbedtls_sha256_context ctx;
mbedtls_sha256_init(&ctx);
mbedtls_sha256_starts(&ctx, 0);          // 0 = SHA-256
mbedtls_sha256_update(&ctx, uid, uidLen);
mbedtls_sha256_finish(&ctx, digest);
mbedtls_sha256_free(&ctx);

// Hex-encode digest to 64-char string
char token[65];
for (int i = 0; i < 32; i++)
    sprintf(&token[i * 2], "%02x", digest[i]);
```

### Serial Output Format

```
AUTH_OK:a3f2c1d9e8b7...       ← enrolled card tapped
ENROLL_OK:a3f2c1d9e8b7...     ← new card enrolled
AUTH_FAIL:04ab1234             ← unknown card (raw UID on fail only)
```

### Core Loop

```
1. Poll PN532 for card
2. Read UID bytes
3. Debounce (skip if same card within 2s)
4. Compute SHA-256(UID) on-device
5. Check NVS for token
   ├── Found      → Serial: AUTH_OK:<token>
   ├── Not found + enroll mode → Store to NVS → Serial: ENROLL_OK:<token>
   └── Not found + normal mode → Serial: AUTH_FAIL:<uid_hex>
```

---

## Python Bridge

### Install

```bash
pip install pyserial supabase
```

### Config

Edit the top of `bridge.py`:

```python
SERIAL_PORT  = '/dev/ttyUSB0'   # Windows: 'COM3'
BAUD_RATE    = 115200
SUPABASE_URL = 'https://your-project.supabase.co'
SUPABASE_KEY = 'sb_publishable_...'
```

### Logic

```
1. Open serial port at 115200 baud
2. Read line, strip whitespace
3. Parse prefix: AUTH_OK / ENROLL_OK / AUTH_FAIL
4. Extract token or UID after colon
5. INSERT into auth_events { event, token, created_at }
6. On ENROLL_OK → also INSERT into enrolled_tokens if not exists
7. Log result. On error: log and continue, do not crash
```

### Run

```bash
python bridge.py
```

Keep this terminal open while the ESP32 is powered. The bridge must be running for events to reach Supabase.

---

## Supabase Setup

### 1. Create Tables

Run in **SQL Editor → New Query**:

```sql
CREATE TABLE auth_events (
  id         uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  event      text NOT NULL,
  token      text,
  uid        text,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE enrolled_tokens (
  id         uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  token      text NOT NULL UNIQUE,
  created_at timestamptz DEFAULT now()
);
```

### 2. Enable Row Level Security

> **Do this before sharing the demo link publicly.**

1. Supabase dashboard → **Table Editor**
2. Click `auth_events` → toggle **RLS → ON**
3. Repeat for `enrolled_tokens`

### 3. Add Policies

Without policies, RLS locks the tables completely — even your bridge can't write. Run this:

```sql
-- Allow bridge to insert events
CREATE POLICY "allow anon insert" ON auth_events
  FOR INSERT TO anon WITH CHECK (true);

CREATE POLICY "allow anon insert" ON enrolled_tokens
  FOR INSERT TO anon WITH CHECK (true);

-- Allow dashboard to read events
CREATE POLICY "allow anon select" ON auth_events
  FOR SELECT TO anon USING (true);

CREATE POLICY "allow anon select" ON enrolled_tokens
  FOR SELECT TO anon USING (true);
```

### 4. Enable Realtime

**Database → Replication → auth_events → toggle Insert → ON → Save**

### API Key Safety

The dashboard uses a `sb_publishable_` prefixed key. This is **safe to expose in frontend code** — Supabase designed it for this purpose.

| Action | Safe? |
|---|---|
| Publishable key in frontend JS | ✅ Yes |
| Deploy to Vercel with key in source | ✅ Yes |
| Share demo link publicly | ✅ Yes (with RLS on) |
| Expose `service_role` key anywhere | ❌ Never |
| Leave RLS disabled on tables | ❌ Never |

---

## Dashboard Features

| Feature | Description |
|---|---|
| Lock Screen | Animated NFC card + radar sweep. Waits for enrolled card tap. |
| Session Countdown | SVG ring timer. 300s → amber at 120s → red at 60s. |
| Auth Event Log | Realtime log of all AUTH_OK, AUTH_FAIL, ENROLL_OK events. |
| Card DNA | Barcode visualization generated deterministically from token hash. |
| Identity Ghost | Particle system that assembles a unique shape per enrolled card. |
| Globe Map | 3D canvas globe with ping animations at reader location. |
| Enrolled Cards | Token list with use count, last-seen, nickname editor, revoke button. |
| RF Waveform | Simulated 13.56MHz signal that spikes on card read. |
| Location Override | Set fixed lat/lon for reader to control globe ping placement. |

### Deploy

```bash
# Vercel (recommended)
# Push HTML to GitHub → import at vercel.com/new → deploy

# Local
python -m http.server 8080
# open http://localhost:8080
```

---

## Running the Full System

### Checklist

- [ ] Supabase tables created
- [ ] RLS enabled on both tables
- [ ] INSERT + SELECT policies added for anon role
- [ ] Realtime enabled on `auth_events`
- [ ] ESP32 flashed with firmware
- [ ] PN532 wired correctly (DIP switches in I2C mode)
- [ ] `bridge.py` configured with correct port + Supabase credentials
- [ ] Dashboard deployed or served locally

### Startup Sequence

```
1. Connect ESP32 via USB
2. Run: python bridge.py
3. Verify: "Connected to /dev/ttyUSB0 at 115200 baud"
4. Open dashboard → topbar shows REALTIME CONNECTED
5. Tap card → lock screen responds in < 100ms
6. First tap → ENROLL_OK → card registered
7. Second tap → AUTH_OK → dashboard unlocks → TTL starts
```

### Troubleshooting

| Symptom | Fix |
|---|---|
| No serial data from bridge | Check COM port. On Linux: `ls /dev/tty*` with/without ESP32 plugged in. |
| PN532 not detected | Check DIP switches (SW1=OFF, SW2=ON). Verify SDA=GPIO21, SCL=GPIO22. |
| Dashboard stuck on CONNECTING | Check SUPABASE_URL and SUPABASE_KEY. Open DevTools → Network. |
| Bridge permission error | RLS on but INSERT policy missing. Re-run the policy SQL from Section 3. |
| No live updates on dashboard | Realtime not enabled. Database → Replication → auth_events → Insert ON. |
| AUTH_FAIL on enrolled card | Hash mismatch. Re-flash firmware and re-enroll the card. |

---

## Future Work

### Security Hardening
- **Challenge-response nonce** — Eliminate replay risk: `SHA-256(UID + server_nonce)`
- **HMAC-SHA256** — Replace plain hash with `HMAC-SHA256(UID, device_secret)` provisioned in ESP32 eFuse
- **mTLS on bridge** — Mutual TLS between bridge and Supabase backend
- **Rate limiting** — Edge function middleware to limit INSERT requests per IP

### Feature Extensions
- **Multi-reader support** — Multiple ESP32+PN532 units posting to same backend with `reader_id`
- **RBAC** — Assign roles (admin/user/guest) to enrolled cards, issue JWT on auth
- **WebAuthn fallback** — Passkey fallback for environments without NFC readers
- **Zero Trust integration** — Feed auth events into MQTT + Hyperledger Fabric trust scoring pipeline
- **OLED display** — SSD1306 for on-device feedback (padlock animation on grant, glitch on deny)
- **Relay control** — GPIO relay to physically actuate a door lock on AUTH_OK

---

## Project Structure

```
nfc-vault/
├── firmware/
│   └── main.ino          # ESP32 firmware (Arduino)
├── bridge/
│   └── bridge.py         # Python serial → Supabase bridge
├── dashboard/
│   └── index.html        # Self-contained web dashboard
└── README.md
```

---

## Built By

**Padmesh** — Embedded systems & security student  
Coimbatore, India · 2025

> *Still a student. Still building. But this one made me genuinely rethink why we're so comfortable with how broken passwords already are.*

**Demo → [pif-green.vercel.app](https://pif-green.vercel.app)**

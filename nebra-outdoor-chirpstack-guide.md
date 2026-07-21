# Nebra Outdoor Hotspot (Gen 1) → ChirpStack LoRaWAN Gateway Conversion Guide

Convert a **Nebra Outdoor Helium Hotspot (Gen 1, CM3-based)** into a production LoRaWAN gateway running **Raspberry Pi OS + ChirpStack Concentratord + MQTT Forwarder**.

> **Why this guide exists:**
>
> 1. **These units are flooding the secondhand market.** Nebra is no longer an active presence in the Helium ecosystem, and with Helium mining economics long past their peak, Nebra Outdoor hotspots are increasingly available cheap on eBay, Facebook Marketplace, and surplus channels — often for less than the cost of the enclosure and PoE hardware alone.
> 2. **Used units are difficult to keep on (or return to) the Helium network.** Hotspot ownership is bound to the original owner's Helium wallet and the onboarding/ECC key; transferring a secondhand unit requires the seller's cooperation, maker-app support, and fees — and with the manufacturer effectively gone, re-onboarding support is unreliable to nonexistent. For most used units, private LoRaWAN is the realistic second life.
> 3. **The hardware is genuinely good** — weatherproof IP67 enclosure, PoE, CM3 compute, mPCIe concentrator slot — a solid foundation for a private ChirpStack gateway.
> 4. **The required settings are documented nowhere obvious.** The Nebra Outdoor uses completely non-standard concentrator wiring, documented *only* in Nebra's own `hm-pyhelper` Python library — not in any ChirpStack documentation, forum thread, or hardware datasheet. Without these settings, the concentrator returns `chip version is 0x00` and `lgw_start failed` no matter what module, software, or config you use.

---

## TL;DR — The Three Critical Settings

| Setting | Standard/Expected Value | **Nebra Outdoor Actual Value** |
|---|---|---|
| SPI device | `/dev/spidev0.0` | **`/dev/spidev1.2`** (SPI bus 1, CS 2) |
| Reset GPIO (BCM) | 17 (RAK HAT) or 25 (MNTD) | **GPIO 38** |
| Boot overlay | none needed | **`dtoverlay=spi1-3cs`** required |

Source: [`NebraLtd/hm-pyhelper` → `hardware_definitions.py`](https://github.com/NebraLtd/hm-pyhelper/blob/master/hm_pyhelper/hardware_definitions.py), variant `nebra-outdoor1`:

```python
'nebra-outdoor1': {
    'SPIBUS': 'spidev1.2',
    'RESET': 38,
    ...
}
```

---

## Hardware Notes

| Component | Detail |
|---|---|
| Enclosure | Nebra Outdoor Hotspot Gen 1 (PoE powered) |
| Compute | Raspberry Pi CM3 (BCM2837), eMMC via removable "SD card key" module |
| Concentrator slot | mPCIe (52-pin), **SPI + GPS PPS only wired** — no USB to the slot |
| Stock concentrator | MaxiIoT **GL5712** (SX1301 + 2× SX1257) |
| Tested replacement | **RAK2287 SPI** (SX1302, salvaged from MNTD RAK v2 Blackspot) — works |
| Ethernet | USB-attached (Exar XR22800 hub w/ USB-Ethernet) — works fine on Raspberry Pi OS, **not supported by ChirpStack Gateway OS (OpenWrt)** |

### Known hardware pitfalls

1. **ChirpStack Gateway OS (OpenWrt) does not work on this board.** Its minimal kernel lacks drivers for the Exar USB-Ethernet chip, resulting in no network and no AP fallback. Use plain Raspberry Pi OS (64-bit Lite recommended).
2. **The GL5712 (SX1301) module could not be made to work** with ChirpStack Concentratord — confirmed by controlled test: with the *verified-correct* wiring (`/dev/spidev1.2`, reset GPIO 38 — the same settings on which a RAK2287 works flawlessly in the same slot), `chirpstack-concentratord-sx1301` still fails at `lgw_start`. Root cause undetermined (module fault vs. a GL5712-specific HAL incompatibility, possibly clock-source wiring); matches an unresolved [ChirpStack forum thread](https://forum.chirpstack.io/t/gateway-hardware-issue-maxiiot-gl5712-module-nebra-helium-hotspot-hat/17519). Replace it with an SX1302/SX1303 mPCIe module (RAK2287 SPI variant confirmed working end-to-end, including live uplink reception; Nebra SX1303 and Waveshare SX1303 should also work).
3. **RAK2287 comes in SPI and USB variants.** The Nebra slot only wires SPI — a USB-variant module will power up but never respond.
4. **eMMC "SD card key"**: flash it like a normal SD card via USB reader. No jumper/USB-boot procedure needed.
5. **Legacy sysfs GPIO is gone** on modern Raspberry Pi OS — old Nebra reset scripts using `/sys/class/gpio` fail. Concentratord's cdev-based reset handles this correctly.

---

## Step-by-Step Deployment

### 1. Flash Raspberry Pi OS

- **Raspberry Pi OS Lite 64-bit** (Bookworm/Trixie) to the eMMC key module via USB adapter
- Headless setup: create `userconf.txt` in the boot partition (`username:$6$hash` via `openssl passwd -6`), plus empty `ssh` file

### 2. Enable SPI buses

```bash
# Primary SPI + SPI1 with 3 chip selects (concentrator is on spidev1.2)
echo "dtparam=spi=on" | sudo tee -a /boot/firmware/config.txt
echo "dtoverlay=spi1-3cs" | sudo tee -a /boot/firmware/config.txt
sudo reboot
```

Verify after reboot — you need to see **spidev1.2**:
```bash
ls /dev/spidev*
# Expect: /dev/spidev0.0  /dev/spidev0.1  /dev/spidev1.0  /dev/spidev1.1  /dev/spidev1.2
```

### 3. Install ChirpStack Concentratord (SX1302)

Concentratord ships as a standalone `.deb` on the ChirpStack artifacts server (it is **not** the `chirpstack` apt package — that's the full network server and does not belong on the gateway):

```bash
curl -sL https://artifacts.chirpstack.io/downloads/chirpstack-concentratord/chirpstack-concentratord-sx1302_4.7.1_linux_arm64.deb -o /tmp/concentratord.deb
sudo dpkg -i /tmp/concentratord.deb
```

> Check [artifacts.chirpstack.io/downloads/chirpstack-concentratord/](https://artifacts.chirpstack.io/downloads/chirpstack-concentratord/) for the current version.

The package's `preinst` script creates the `chirpstack` system user if it doesn't already exist (the MQTT Forwarder deb carries the identical script, so install order between the two doesn't matter — whichever lands first creates the user).

Add hardware group permissions (required or you get `setup reset pins error: Permission denied`):
```bash
sudo usermod -aG spi,gpio,dialout chirpstack
```

### 4. Configure Concentratord

The deb installs **three** config files to `/etc/chirpstack-concentratord-sx1302/`, and the systemd unit loads all of them, merged in this order:

```
concentratord.toml  →  channels.toml  →  region.toml
```

> ⚠️ **Multi-file gotcha:** the three `-c` files are **concatenated and parsed as a single TOML document** — not merged. The shipped `channels.toml` and `region.toml` contain EU868 defaults, so if you put a full config in `concentratord.toml` and leave the other two populated, the duplicated `[gateway.concentrator]` table causes a hard startup panic: `Error parsing config file: ... "duplicate key"`. Blank **both** files (single-file approach below) — blanking only one still crashes. After fixing, you may also need `sudo systemctl reset-failed chirpstack-concentratord-sx1302` to clear systemd's rapid-restart lockout (`Start request repeated too quickly`).

Single-file approach — blank the region/channel files and keep everything in `concentratord.toml`:

```bash
sudo truncate -s0 /etc/chirpstack-concentratord-sx1302/channels.toml
sudo truncate -s0 /etc/chirpstack-concentratord-sx1302/region.toml
```

`/etc/chirpstack-concentratord-sx1302/concentratord.toml` (US915 FSB2 shown):

```toml
[concentratord]
log_level = "INFO"
log_to_syslog = false
stats_interval = "30s"

[concentratord.api]
event_bind = "ipc:///tmp/concentratord_event"
command_bind = "ipc:///tmp/concentratord_command"

[gateway]
antenna_gain = 0
lorawan_public = true
region = "US915"
model = "rak_2287"
model_flags = []
gateway_id = ""

# ── Nebra Outdoor Gen 1 specific ──────────────────────────
com_dev_path = "/dev/spidev1.2"
sx1302_reset_pin = 38
# ──────────────────────────────────────────────────────────

[gateway.concentrator]
multi_sf_channels = [
    903900000,
    904100000,
    904300000,
    904500000,
    904700000,
    904900000,
    905100000,
    905300000,
]

[gateway.concentrator.lora_std]
frequency = 904600000
bandwidth = 500000
spreading_factor = 8

[gateway.concentrator.fsk]
frequency = 904600000
bandwidth = 125000
datarate = 50000
```

Notes:
- The explicit `[gateway.concentrator]` channel section **is required** — the `rak_2287` model profile alone does not populate a channel plan (channels show `enabled: false, freq: 0` without it).
- GNSS is disabled by default when `gnss_dev_path` is unset — leave it unset (no GPS antenna / no sky view inside the enclosure causes startup issues otherwise).
- Match `multi_sf_channels` to your network server's sub-band.

### 5. Start and verify

```bash
sudo systemctl restart chirpstack-concentratord-sx1302
sudo journalctl -u chirpstack-concentratord-sx1302 -f --no-pager
```

**Success looks like:**
```
Gateway ID retrieved, gateway_id: "0016c001XXXXXXXX"
Initializing JIT queue, capacity: 32
Creating socket for publishing events, bind: ipc:///tmp/concentratord_event
```

**Failure looks like** (wrong SPI path or reset pin):
```
Note: chip version is 0x00 (v0.0)
ERROR: Failed to set SX1250_0 in STANDBY_RC mode
called `Result::unwrap()` on an `Err` value: lgw_start failed
```

Record the Gateway ID — it is read from the SX1302 chip and is what you register in ChirpStack.

### 6. Install and configure MQTT Forwarder

The MQTT Forwarder *is* published in the ChirpStack apt repository, so add the repo and install from there:

```bash
# Add the ChirpStack signing key
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://artifacts.chirpstack.io/packages/chirpstack.key | gpg --dearmor | sudo tee /etc/apt/keyrings/chirpstack.gpg > /dev/null

# Add the repo
echo "deb [signed-by=/etc/apt/keyrings/chirpstack.gpg] https://artifacts.chirpstack.io/packages/4.x/deb stable main" | sudo tee /etc/apt/sources.list.d/chirpstack.list

# Install
sudo apt update
sudo apt install -y chirpstack-mqtt-forwarder
```

`/etc/chirpstack-mqtt-forwarder/chirpstack-mqtt-forwarder.toml`:

```toml
[backend]
enabled = "concentratord"

[backend.concentratord]
event_url = "ipc:///tmp/concentratord_event"
command_url = "ipc:///tmp/concentratord_command"

[mqtt]
server = "mqtts://YOUR-BROKER:8883"    # or mqtt://...:1883 for plain
topic_prefix = "us915"                  # MUST match the region id in your ChirpStack server config
qos = 0
clean_session = true
keep_alive_interval = "30s"
json = false
```

> ⚠️ **Topic prefix gotcha:** `topic_prefix` must be the ChirpStack **region id only** (e.g. `us915`), *not* `us915/gateway`. The forwarder's built-in topic template already appends `/gateway/{gateway_id}/event/{type}`. Setting the prefix to `us915/gateway` produces `us915/gateway/gateway/EUI/event/stats` — one level too deep — and ChirpStack (subscribed to `<region_id>/gateway/+/event/+`) never sees it. Symptom: forwarder logs show successful sends, but the gateway stays "Never seen" in the UI.
>
> Confirm the region id on your server: check the mounted `region_*.toml` (e.g. `region_us915_0.toml`) for its `topic_prefix`/`id` value. You can also verify delivery end-to-end with `mosquitto_sub -h YOUR-BROKER -p 8883 --capath /etc/ssl/certs -t '<region_id>/gateway/#' -v`.
>
> TLS note: with a public CA cert on the broker (e.g. Let's Encrypt via Caddy), leave `ca_cert`/`tls_cert`/`tls_key` unset — the system trust store handles it.

```bash
sudo systemctl enable --now chirpstack-mqtt-forwarder
```

### 7. Register in ChirpStack

Add the gateway (EUI from step 5) under your tenant in the ChirpStack UI. "Last seen" should update within ~30 seconds (stats publish on concentratord's `stats_interval`).

**Verification checklist:**
```bash
# Forwarder journal — topic must have a SINGLE /gateway/ segment:
sudo journalctl -u chirpstack-mqtt-forwarder -f --no-pager
# ✅ topic: us915/gateway/0016c001XXXXXXXX/event/stats
# ❌ topic: us915/gateway/gateway/0016c001XXXXXXXX/event/stats  (prefix misconfigured)

# Optional broker-side confirmation:
mosquitto_sub -h YOUR-BROKER -p 8883 --capath /etc/ssl/certs -t 'us915/gateway/#' -v
```

Uplink frames from in-range LoRa devices appear in the journal as `Received uplink frame, uplink_id: ...` and confirm the RF path end-to-end.

> Deployment reminder: set real coordinates in `[gateway.location]` in the concentratord toml before field installation so ChirpStack maps the gateway correctly.

---

## Optional: mTLS Gateway Authentication

The basic setup above works with an open (or username/password) broker listener. For production fleets, mutual TLS is the better model — each gateway authenticates with its own client certificate instead of shared credentials, and certificates can be individually revoked.

### Prerequisites

- Broker's TLS listener configured to require client certificates (Mosquitto: `require_certificate true` + `cafile` pointing at your CA on the 8883 listener)
- Three files per gateway: CA certificate, gateway client certificate, gateway private key
  - ChirpStack's UI can generate these per-gateway (gateway page → certificate generation) using its internal CA — the PEM blocks are shown **once**, so save them immediately. Bonus: the cert CN is the gateway EUI, enabling per-gateway Mosquitto topic ACLs.
  - Or use your own CA (openssl, step-ca, etc.)

### Install certificates on the gateway

```bash
sudo mkdir -p /etc/chirpstack-mqtt-forwarder/certs
# copy ca.crt, gateway-<EUI>.crt, gateway-<EUI>.key into that directory, then:
sudo chown -R chirpstack:chirpstack /etc/chirpstack-mqtt-forwarder/certs
sudo chmod 600 /etc/chirpstack-mqtt-forwarder/certs/*.key
sudo chmod 644 /etc/chirpstack-mqtt-forwarder/certs/*.crt
```

> Ownership matters: the forwarder runs as the `chirpstack` user (check with `grep User /lib/systemd/system/chirpstack-mqtt-forwarder.service`). A key that is `chmod 600` but owned by root fails with `Open private key: Permission denied`.

### Update the forwarder config

Add the three TLS paths to the existing `[mqtt]` section — **do not remove the `[backend]` sections**:

```toml
[backend]
enabled = "concentratord"

[backend.concentratord]
event_url = "ipc:///tmp/concentratord_event"
command_url = "ipc:///tmp/concentratord_command"

[mqtt]
server = "mqtts://YOUR-BROKER:8883"
topic_prefix = "us915"
qos = 0
clean_session = true
keep_alive_interval = "30s"
json = false

ca_cert = "/etc/chirpstack-mqtt-forwarder/certs/ca.crt"
tls_cert = "/etc/chirpstack-mqtt-forwarder/certs/gateway-<EUI>.crt"
tls_key = "/etc/chirpstack-mqtt-forwarder/certs/gateway-<EUI>.key"
```

> `ca_cert` note: this is the CA used to validate the **broker's server certificate**. If the broker cert is from a public CA (e.g. Let's Encrypt) and only your client certs are private-CA, leave `ca_cert` unset — the system trust store handles server validation.

### Restart and verify

```bash
sudo systemctl reset-failed chirpstack-mqtt-forwarder
sudo systemctl restart chirpstack-mqtt-forwarder
sudo journalctl -u chirpstack-mqtt-forwarder -f --no-pager
```

Success logs `Configuring client with TLS certificate, ...` followed by a normal MQTT connection and stats publishes; the gateway's "Last seen" resumes updating in ChirpStack.

### mTLS troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Forwarder starts but logs show UDP / port 1700 activity, no MQTT | `[backend]` sections removed while editing — forwarder silently falls back to Semtech UDP mode | Restore `[backend]` + `[backend.concentratord]` sections |
| `No such file or directory` on a cert path | Filename mismatch between config and disk (classic: `ca.cert` vs `ca.crt`) | Make `ca_cert`/`tls_cert`/`tls_key` paths match actual filenames exactly |
| `Open private key: Permission denied` | Key is `chmod 600` but owned by root, service runs as `chirpstack` | `sudo chown chirpstack:chirpstack` on the certs directory |
| Panic: `Setup MQTT client error: peer sent no certificates` | Wrong/mismatched cert + key pair, or invalid PEM files | Re-download the pair for this gateway's EUI; verify match: `openssl x509 -noout -modulus -in cert \| openssl md5` vs same on the key |
| `Start request repeated too quickly` after fixing config | systemd rapid-restart lockout from the failed attempts | `sudo systemctl reset-failed chirpstack-mqtt-forwarder` then restart |
| TLS handshake rejected by broker | Broker's `cafile` doesn't include the CA that signed the client cert, or `require_certificate` misconfigured | Verify broker listener config; test with `mosquitto_pub` using the same cert/key |

> Cert lifecycle reminder: ChirpStack-issued gateway certs expire. Track expiry dates (`openssl x509 -noout -enddate -in cert`) — an expired client cert presents as sudden handshake failures on a previously working gateway. Also note the EUI is read from the SX1302 chip, so swapping the concentrator module changes the EUI and requires a new gateway registration + new certs.

---

## Troubleshooting Quick Reference

| Symptom | Cause | Fix |
|---|---|---|
| `chip version is 0x00` + `lgw_start failed` | Wrong SPI device or reset pin | `com_dev_path = "/dev/spidev1.2"`, `sx1302_reset_pin = 38` |
| No `/dev/spidev1.2` | Missing overlay | `dtoverlay=spi1-3cs` in `/boot/firmware/config.txt` + reboot |
| `setup reset pins error: Permission denied` | chirpstack user lacks GPIO access | `sudo usermod -aG spi,gpio,dialout chirpstack` + restart service |
| Channels all `enabled: false, freq: 0` | Missing `[gateway.concentrator]` section | Add explicit channel plan (see config above) |
| Panic: `Error parsing config file: ... "duplicate key"` | Full config in `concentratord.toml` while shipped `channels.toml`/`region.toml` (EU868 defaults) still populated — files are concatenated, so tables collide | Blank **both** files (`sudo truncate -s0 ...`), then `systemctl reset-failed` + restart |
| `gateway_id must be exactly 8 bytes` (older builds) | Empty `gateway_id=""` on 4.5.x sx1301 builds | Omit the field or use 16 hex chars; not an issue on 4.7.x sx1302 |
| No Ethernet under ChirpStack Gateway OS | OpenWrt lacks Exar USB-Ethernet driver | Use Raspberry Pi OS instead |
| Works in MNTD Blackspot but not Nebra | Different wiring per board | MNTD: spidev0.0 + GPIO25. Nebra Outdoor: spidev1.2 + GPIO38 |
| Forwarder sends OK, but gateway "Never seen" in ChirpStack | `topic_prefix` includes `/gateway` (doubled topic segment) | Set `topic_prefix` to region id only (e.g. `us915`); verify topic in journal reads `us915/gateway/EUI/event/stats` |
| Web UI / Postgres errors on the gateway itself | `sudo apt install chirpstack` was run — that's the full network server, not the gateway daemon | `sudo apt remove chirpstack`; install `chirpstack-concentratord-sx1302` (step 3) |

## Board-to-Settings Cheat Sheet (Helium salvage hardware)

| Board | SPI Device | Reset GPIO (BCM) | Notes |
|---|---|---|---|
| MNTD RAK v2 Blackspot (RAK7248, Pi 4) | `spidev0.0` | 25 | gpiochip4 on Pi4/CM4 kernels |
| Nebra Outdoor Gen 1 (CM3) | `spidev1.2` | 38 | needs `spi1-3cs` overlay; gpiochip0 |
| Nebra Indoor | `spidev0.0` | 22 | per hm-pyhelper |
| Standard RAK2287 Pi HAT | `spidev0.0` | 17 | concentratord default |

> Authoritative source for all Nebra variants: [`hm-pyhelper/hardware_definitions.py`](https://github.com/NebraLtd/hm-pyhelper/blob/master/hm_pyhelper/hardware_definitions.py)

---

## Diagnostic Techniques That Cracked It

Documented because they're reusable for any "mystery board" concentrator bring-up:

1. **Raw SPI register read** (bypasses all HAL logic — distinguishes "dead bus" from "init sequence problem"):
```python
import spidev
spi = spidev.SpiDev()
spi.open(1, 2)                    # bus, chip-select
spi.max_speed_hz = 1000000
resp = spi.xfer2([0x00, 0x01, 0x00])
print([hex(x) for x in resp])     # all 0x00 = nothing responding
```

2. **3.3V rail check at the mPCIe slot** with a multimeter — separates power-delivery faults from signal-routing problems.

3. **Vendor's own hardware definitions are ground truth.** For any ex-Helium hardware, check the manufacturer's miner software repos (`hm-pyhelper`, `hm-pktfwd`, balena configs) for SPI bus, reset pin, and overlay requirements before trusting any generic profile.

4. **`chip version is 0x00` always means no SPI communication** — it is never a config-file, region, or channel-plan problem. Fix the physical/bus layer first.

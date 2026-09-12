# airrohr-firmware: what it does

A reference description of the original [airRohr firmware](../src-original/airrohr-firmware), written
as input to the ESPHome port. It focuses on the configuration this project targets — **NodeMCU v2/v3
+ SDS011 + BME280, no display** — and treats everything else as an appendix.

Sections 1–8 describe the firmware as it is. Where a behaviour will be handled differently in the
ESPHome port, a short *Porting note* says so; those decisions are collected in
[Appendix B](#appendix-b--porting-notes).

---

## 1. What airRohr is

airRohr is the GPL-3.0 Arduino firmware behind the Sensor.Community (formerly Luftdaten.info)
citizen air-quality network. A NodeMCU board with a particulate-matter sensor and usually a
temperature/humidity sensor measures the air, and every couple of minutes posts the readings to
sensor.community and other endpoints.

| | |
| --- | --- |
| Version documented here | `NRZ-2024-135` ([airrohr-firmware.ino:61](../src-original/airrohr-firmware/airrohr-firmware.ino#L61)) |
| Primary target | NodeMCU v2/v3 (ESP8266), 160 MHz, 4M flash with a 3M SPIFFS partition ([platformio.ini:37-39](../src-original/airrohr-firmware/platformio.ini#L37-L39)) |
| Secondary target | ESP32 — present in the source but marked experimental |
| Build system | PlatformIO, ~27 `nodemcuv2_*` environments that differ only in the compiled-in UI language |
| License | GPL-3.0 |

Two properties shape the whole design:

- **One binary fits all hardware.** The firmware contains drivers for every supported sensor, and the
  user ticks boxes on a local web page to say what is actually wired up. Nothing is decided at
  compile time except the language. This is the opposite of ESPHome, where the YAML decides what gets
  compiled in.
- **It is a field device on someone else's roof.** Hence the captive-portal setup flow, the automatic
  daily firmware update, the forced reboot every four weeks, and the duty-cycled fan.

## 2. The device at a glance

```
                 every 145 s
  SDS011 ──serial──┐
                   ├──► measurement cycle ──► JSON payload ──┬──► sensor.community (one POST per sensor)
  BME280  ──I²C────┘        (§4)                  (§5)       ├──► madavi.de (one combined POST)
                                                             └──► optional: openSenseMap, InfluxDB,
                                                                  custom HTTP, CSV over serial …
  local web UI (§6) ── /values /status /data.json /metrics
  SPIFFS /config.json (§7)      daily OTA check (§8)
```

## 3. The hardware configuration we target

### Wiring

Serial connections are crossed: the sensor's TX goes to the board's RX.

| Signal | NodeMCU pin | GPIO | Notes |
| --- | --- | --- | --- |
| SDS011 TX → board RX | D1 | GPIO5 | `PM_SERIAL_RX` |
| SDS011 RX ← board TX | D2 | GPIO4 | `PM_SERIAL_TX` |
| SDS011 power | VU + GND | — | 5 V, drawn from USB rail |
| BME280 SDA | D3 | GPIO0 | `I2C_PIN_SDA` |
| BME280 SCL | D4 | GPIO2 | `I2C_PIN_SCL` |
| BME280 power | 3V3 + GND | — | 3.3 V |

Source: [Readme.md:62-69,123-128](../src-original/airrohr-firmware/Readme.md#L62-L69) and
[ext_def.h:105-120](../src-original/airrohr-firmware/ext_def.h#L105-L120).

### SDS011

Driven over **SoftwareSerial at 9600 8N1**, not a hardware UART
([airrohr-firmware.ino:5918](../src-original/airrohr-firmware/airrohr-firmware.ino#L5918)) — the
ESP8266's single usable hardware UART is taken by the debug console. The firmware speaks the Nova
Fitness protocol directly: it sends `Start`, `Stop` and `ContinuousMode` command frames, then scans
the incoming stream for the `0xAA 0xC0` measurement header followed by an 8-byte body with a
checksum ([:3488-3520](../src-original/airrohr-firmware/airrohr-firmware.ino#L3488-L3520)).

At boot the firmware queries the sensor's firmware version/date, switches it to continuous reporting,
and then immediately stops it ([:5491-5498](../src-original/airrohr-firmware/airrohr-firmware.ino#L5491-L5498)).

### BME280

Probed at I²C **`0x77`, then `0x76`** if that fails. On success it is configured for **forced mode
with 1× oversampling** on temperature, pressure and humidity
([:5401-5420](../src-original/airrohr-firmware/airrohr-firmware.ino#L5401-L5420)) — the sensor sleeps
between explicit one-shot conversions rather than sampling continuously, which keeps self-heating
down.

The same code path serves BMP280 and BME280; the chip ID read back at init decides which. A BME280
reports temperature, pressure and humidity; a BMP280 reports temperature and pressure only, and
identifies itself to the API under a different pin number. See §5.

## 4. The measurement cycle

This is the part that differs most from ESPHome's model, where each sensor has its own
`update_interval` and publishes independently.

airRohr has **one cycle for the whole device**. `loop()` runs continuously and sets `send_now` when
`sending_intervall_ms` has elapsed since the last send
([:5989](../src-original/airrohr-firmware/airrohr-firmware.ino#L5989)). The default interval is
**145 000 ms** (~2.5 min, [:138](../src-original/airrohr-firmware/airrohr-firmware.ino#L138)); it is
user-configurable but floored at 5 s on load
([:1168-1171](../src-original/airrohr-firmware/airrohr-firmware.ino#L1168-L1171)).

### SDS011 is duty-cycled, not polled

The laser and fan have a limited service life, so the sensor is **off for most of the interval**:

| Phase | Duration | What happens |
| --- | --- | --- |
| Idle | interval − 20 s | Fan stopped (`SDS_cmd(Stop)`) |
| Warm-up | 15 s (`WARMUPTIME_SDS_MS`) | Fan started, readings arrive but are **discarded** |
| Reading | 5 s (`READINGTIME_SDS_MS`) | Readings accumulated into sum/min/max |
| Send | — | Average computed, payload sent, fan stopped again |

Implementation at [:3470-3565](../src-original/airrohr-firmware/airrohr-firmware.ino#L3470-L3565),
constants at [defines.h:51-53](../src-original/airrohr-firmware/defines.h#L51-L53). The sensor emits
roughly one frame per second, so a cycle collects about 5 samples. If the configured interval is
shorter than warm-up + reading (20 s), the duty cycling is skipped and the fan simply runs
continuously.

### Averaging discards the extremes

At send time, if more than two samples were collected, **the single lowest and single highest are
dropped** and the rest averaged
([:3527-3536](../src-original/airrohr-firmware/airrohr-firmware.ino#L3527-L3536)):

```cpp
if (sds_val_count > 2) {
    sds_pm10_sum = sds_pm10_sum - sds_pm10_min - sds_pm10_max;
    sds_pm25_sum = sds_pm25_sum - sds_pm25_min - sds_pm25_max;
    sds_val_count = sds_val_count - 2;
}
last_value_SDS_P1 = float(sds_pm10_sum) / (sds_val_count * 10.0f);
```

The SDS011 reports tenths of µg/m³, hence the division by 10. Fewer than three usable samples in a
cycle increments an error counter shown on the status page; zero samples means nothing is reported
for that cycle.

*Porting note: the duty cycling and the trimmed average are both reproduced in the port. They are
airRohr-specific behaviour that sensor.community's data expects, and the duty cycling is what
preserves the laser's service life — so the SDS011 is not left free-running on an ESPHome
`update_interval`.*

### BME280 is read once per cycle

Unlike the PM sensor, the BME280 has no sampling loop. `fetchSensorBMX280()` is called **inside the
`send_now` block** ([:6187-6201](../src-original/airrohr-firmware/airrohr-firmware.ino#L6187-L6201)),
so it takes exactly one forced measurement per interval and sends it unaveraged. A failed read is
reported as no value at all for that cycle, not as a stale one.

*Porting note: the port makes this a normal polled ESPHome sensor — nothing here is special beyond the
temperature offset. The one behaviour to preserve is the failure case: a failed read must be omitted
from the payload, not serialised as `NaN`.*

### Corrections

- **`temp_correction`** is a user-supplied offset that exists because sensors mounted in the standard
  drainpipe enclosure read warm. It is added at exactly three call sites — DHT22
  ([:3270](../src-original/airrohr-firmware/airrohr-firmware.ino#L3270)), BMX280
  ([:3413](../src-original/airrohr-firmware/airrohr-firmware.ino#L3413)) and DS18B20
  ([:3460](../src-original/airrohr-firmware/airrohr-firmware.ino#L3460)) — and **to temperature
  only**; humidity and pressure are never corrected.
  + It is **not** applied to HTU21D, SHT3x, SCD30 or BMP180, which store their temperature raw. The
    config page labels the field generically as "Correction in °C"
    ([:1776](../src-original/airrohr-firmware/airrohr-firmware.ino#L1776)) with nothing indicating
    the limited scope, so this reads as an oversight rather than intent.
  + The offset is applied **at the point of storage**, so the corrected value is what reaches the
    APIs, the `/values` page, and the derived `dew_point()` and `pressure_at_sealevel()`
    calculations. There is no raw-versus-corrected distinction anywhere in the firmware.
  + *Porting note: the port applies the correction to **every** temperature sensor, including the four
    airRohr misses — the inconsistency is treated as a bug to fix, not a behaviour to reproduce.*
- **`height_above_sealevel`** feeds `pressure_at_sealevel()`, the barometric reduction of the measured
  pressure to sea level ([:1360-1367](../src-original/airrohr-firmware/airrohr-firmware.ino#L1360-L1367)).
  This value is **only displayed on the local `/values` page** and is never sent to any API
  ([:2215](../src-original/airrohr-firmware/airrohr-firmware.ino#L2215)). Likewise the **dew point**
  shown for BME280 is computed locally for display only
  ([:2219-2220](../src-original/airrohr-firmware/airrohr-firmware.ino#L2219-L2220)).
  + *Porting note: both are kept, and kept **local** — surfaced through the ESPHome web server / Home
    Assistant, never added to an upload payload. Neither has a built-in ESPHome platform, so both are
    template sensors reproducing these formulas from the offset-corrected temperature.*

### Per-cycle housekeeping

After each send ([:6264-6283](../src-original/airrohr-firmware/airrohr-firmware.ino#L6264-L6283)):

- reconnect WiFi if the link dropped, counting the failure;
- reboot if the device has been up longer than **28 days** (`DURATION_BEFORE_FORCED_RESTART_MS`);
- run the OTA check if a day has passed since the last attempt.

Sends are also suppressed for the first ~65 s after boot until NTP has synced
([:5997-6003](../src-original/airrohr-firmware/airrohr-firmware.ino#L5997-L6003)).

*Porting note: the OTA check is not ported — ESPHome's own update flow replaces it.*

## 5. What gets sent, exactly

### The payload envelope

Every destination receives the same basic document
([:664](../src-original/airrohr-firmware/airrohr-firmware.ino#L664)):

```json
{"software_version": "NRZ-2024-135", "sensordatavalues":[{"value_type":"…","value":"…"}]}
```

Two details matter for a port: **every value is a JSON string, never a number**, and floats are
formatted by Arduino's `String(float)` — i.e. **two decimal places**
([utils.cpp:230-242](../src-original/airrohr-firmware/utils.cpp#L230-L242)).

### Internal value names for our configuration

| Sensor | `value_type` | Meaning | Unit |
| --- | --- | --- | --- |
| SDS011 | `SDS_P1` | PM10 | µg/m³ |
| SDS011 | `SDS_P2` | PM2.5 | µg/m³ |
| BME280 | `BME280_temperature` | temperature (incl. correction) | °C |
| BME280 | `BME280_humidity` | relative humidity | % |
| BME280 | `BME280_pressure` | **absolute** pressure, **not** reduced to sea level | Pa |

Appended once per cycle to the combined payload only: `samples` (loop iterations this cycle),
`min_micro` / `max_micro` (fastest and slowest loop pass, a health indicator), `interval`, and
`signal` (WiFi RSSI) ([:6240-6244](../src-original/airrohr-firmware/airrohr-firmware.ino#L6240-L6244)).

### sensor.community: one POST per sensor, with a virtual pin

Sensor.Community's API models each sensor as a separate "pin" on the device, so the firmware sends
**a separate request per sensor**, each carrying only that sensor's values
([:3130-3148](../src-original/airrohr-firmware/airrohr-firmware.ino#L3130-L3148),
[:6131-6199](../src-original/airrohr-firmware/airrohr-firmware.ino#L6131-L6199)).

Two transformations happen on the way out:

1. **The sensor prefix is stripped** from every key: `SDS_P1` → `P1`, `BME280_temperature` →
   `temperature`. The receiving pin gives the values their meaning.
2. **An `X-PIN` header** names that pin:

| Sensor | `X-PIN` |
| --- | --- |
| SDS011 (and PMS, HPM, SPS30, NextPM, IPS-7100) | `1` |
| BME280 | `11` |
| BMP280 / BMP180 | `3` |
| DHT22, HTU21D, SHT3x | `7` |
| GPS | `9` |
| DS18B20 | `13` |
| DNMS | `15` |
| SCD30 | `17` |
| PPD42NS | `5` |

So one cycle with our hardware produces two requests:

```http
POST /v1/push-sensor-data/ HTTP/1.1
Host: api.sensor.community
Content-Type: application/json
X-Sensor: esp8266-12345678
X-MAC-ID: esp8266-a1b2c3d4e5f6
X-PIN: 1
User-Agent: NRZ-2024-135/12345678/a1b2c3d4e5f6

{"software_version": "NRZ-2024-135", "sensordatavalues":[{"value_type":"P1","value":"8.30"},{"value_type":"P2","value":"5.10"}]}
```

```http
POST /v1/push-sensor-data/ HTTP/1.1
Host: api.sensor.community
X-PIN: 11
…
{"software_version": "NRZ-2024-135", "sensordatavalues":[{"value_type":"temperature","value":"18.42"},{"value_type":"pressure","value":"99123.00"},{"value_type":"humidity","value":"54.30"}]}
```

`X-Sensor` is the device identity the network registers: the literal string `esp8266-` plus the ESP
chip ID in decimal. Port **80 by default**, switching to 443 with a TLS session if the `ssl_dusti`
option is on ([:1312-1320](../src-original/airrohr-firmware/airrohr-firmware.ino#L1312-L1320)). The
per-cycle `samples`/`signal` fields are **not** part of these requests. Requests have a 20 s timeout
and failures only increment a counter — there is no retry or queue, the cycle's data is simply lost.

### madavi.de: one combined POST

Madavi is the network's graphing backend. It receives the **entire payload in a single request** —
all sensors plus the housekeeping fields — with no `X-PIN` header
([:5788-5792](../src-original/airrohr-firmware/airrohr-firmware.ino#L5788-L5792)), keys unstripped
(`SDS_P1`, `BME280_temperature`, …). Host `api-rrd.madavi.de/data.php`, port 80 or 443.

*Porting note: the port reproduces this request shape exactly — per-sensor POSTs, stripped prefixes,
`X-PIN`/`X-Sensor` headers, values as two-decimal strings — because it is what the network accepts.
What changes is that publishing to sensor.community and to madavi are each an independent optional
toggle; neither is assumed on.*

### Other destinations

All off by default, configured on the same page ([ext_def.h:57-102](../src-original/airrohr-firmware/ext_def.h#L57-L102)):

| Target | Endpoint | Format |
| --- | --- | --- |
| openSenseMap | `ingress.opensensemap.org/boxes/{id}/data?luftdaten=1`, 443 | combined JSON, box ID in URL |
| Feinstaub-App | `server.chillibits.com/data.php`, 80 | combined JSON |
| aircms | `doiot.ru/php/sensors.php`, 80 | form-ish body, HMAC-SHA1 signed URL |
| Custom HTTP | user host/port/URL, optional basic auth | combined JSON with an added `esp8266id` field |
| InfluxDB | user host/port/URL, optional basic auth | line protocol, measurement name configurable, tagged `node=esp8266-<chipid>` |
| CSV | USB serial | header row + one row per cycle, `;`-separated |

## 6. Local interfaces

The device runs a web server on port 80 with optional HTTP basic auth, and registers mDNS as
`airRohr-<chipid>.local` ([:2715-2736](../src-original/airrohr-firmware/airrohr-firmware.ino#L2715-L2736)):

| Route | Purpose |
| --- | --- |
| `/` | landing page with links |
| `/config` | the configuration form — sensors, APIs, WiFi, intervals |
| `/values` | current readings as an HTML table, including sea-level pressure and dew point |
| `/status` | device health: firmware version, free heap and fragmentation, uptime, reset reason, NTP state, SDS011 firmware version, and an error block — WiFi error count and last disconnect reason, per-API error counts, last send return code, SDS read errors, number of sends, average send time |
| `/data.json` | the last payload verbatim, plus an `age` field in seconds |
| `/metrics` | the same values in OpenMetrics/Prometheus text format, tagged `node="esp8266-<chipid>"` |
| `/debug`, `/serial` | change debug verbosity / view serial output |
| `/removeConfig`, `/reset` | wipe `config.json` / reboot |

Serial debugging runs at **9600 8N1** with five verbosity levels (error → max info,
[defines.h:40-44](../src-original/airrohr-firmware/defines.h#L40-L44)). Setting debug to *none* frees
the port for the CSV output mode.

*Porting note: `/config`, `/debug`, `/removeConfig` and `/reset` have no counterpart in the port —
ESPHome covers configuration, logging, restart and factory reset natively, and registers mDNS itself.
Only the read-only surfaces (`/values`, `/status`, `/data.json`, `/metrics`) are worth considering.*

## 7. Configuration and persistence

About 70 settings live in a `cfg::` namespace
([:133-247](../src-original/airrohr-firmware/airrohr-firmware.ino#L133-L247)). Each has a compile-time
default in [ext_def.h](../src-original/airrohr-firmware/ext_def.h), can be changed at runtime through
`/config`, and is persisted as JSON.

- **Storage**: `/config.json` on SPIFFS. Writes rename the previous file to `/config.json.old` first,
  and a failed parse falls back to that copy
  ([:1096-1221](../src-original/airrohr-firmware/airrohr-firmware.ino#L1096-L1221),
  [:1250-1299](../src-original/airrohr-firmware/airrohr-firmware.ino#L1250-L1299)).
- **Key table**: [airrohr-cfg.h](../src-original/airrohr-firmware/airrohr-cfg.h) is generated, not
  hand-edited. It maps each JSON key (`sds_read`, `bmx280_read`, `sending_intervall_ms`, …) to a typed
  pointer, which is what drives both the config form and the load/save loops.
- **Migration on load**: if the stored `SOFTWARE_VERSION` differs, the file is rewritten after load.
  Legacy keys are folded forward (`bme280_read`/`bmp280_read` → `bmx280_read`), placeholder values are
  cleared (the example senseBox ID, the old `api.luftdaten.info` Influx host), and the interval is
  floored at 5 s.

### AP configuration mode

If the configured WiFi does not connect within ~20 s (40 retries × 500 ms), the device switches to AP
mode ([:2768-2904](../src-original/airrohr-firmware/airrohr-firmware.ino#L2768-L2904)):

1. scan for networks, then pick whichever of channels 1, 6 and 11 has the weakest neighbour;
2. start an AP named `airRohr-<chipid>` at **192.168.4.1**, with a wildcard DNS responder on port 53
   so any HTTP request lands on the config page (captive portal);
3. serve the config UI for `time_for_wifi_config` — **10 minutes** by default — then shut the AP down
   and retry the station connection.

Note a documentation discrepancy worth knowing: `Readme.md` states the AP is unencrypted, but
`FS_PWD` defaults to `airrohrcfg` ([ext_def.h:15](../src-original/airrohr-firmware/ext_def.h#L15)),
so the AP built from this source is password-protected.

*Porting note: settings move into ESPHome YAML rather than a runtime-mutable flash namespace, so
`/config.json`, the `.old` fallback and the migration rules all disappear; AP config mode is replaced
by ESPHome's built-in `wifi` AP mode.*

## 8. Firmware updates

Once a day, and once shortly after the first NTP sync, the firmware checks for a new build
([:4749-4861](../src-original/airrohr-firmware/airrohr-firmware.ino#L4749-L4861)):

1. fetch `firmware.sensor.community/airrohr/update/latest_<lang>.bin.md5` over TLS;
2. compare it with the running sketch's MD5 — equal means nothing to do;
3. download the firmware, its MD5 and a small **second-stage loader** into SPIFFS, validating size and
   checksum;
4. flash the loader and reboot; the loader replaces the main firmware from SPIFFS on the next boot.

The two-stage dance exists because a 1M/3M flash split has no room for two full images side by side.
A `use_beta` flag switches to the beta channel.

*Porting note: none of this is ported. ESPHome's OTA/update mechanism is used instead and updates are
flashed deliberately.*

---

## Appendix A — all supported sensors

Only one PM sensor can be used at a time: SDS011, PMS, HPM, NextPM and IPS-7100 all share the same
serial pins, and the firmware selects one of them at boot
([:5890-5927](../src-original/airrohr-firmware/airrohr-firmware.ino#L5890-L5927)). DHT22 and DS18B20
both occupy D7, so those two are also mutually exclusive. GPS and PPD42NS share D5/D6.

| Sensor | Measures | Bus / pins | Config key | `X-PIN` | `value_type`s |
| --- | --- | --- | --- | --- | --- |
| SDS011 | PM10, PM2.5 | serial D1/D2 @9600 | `sds_read` | 1 | `SDS_P1`, `SDS_P2` |
| PMS1003–7003 | PM1, PM2.5, PM10 | serial D1/D2 | `pms_read` | 1 | `PMS_P0`, `PMS_P1`, `PMS_P2` |
| Honeywell HPM | PM2.5, PM10 | serial D1/D2 | `hpm_read` | 1 | `HPM_P1`, `HPM_P2` |
| Tera NextPM | PM1/2.5/10 mass + count | serial D1/D2 @115200 8E1 | `npm_read` | 1 | `NPM_P0/P1/P2`, `NPM_N1/N10/N25` |
| Piera IPS-7100 | PM0.1–PM10 mass + count | serial D1/D2 @115200 | `ips_read` | 1 | `IPS_P0/P1/P2/P01/P03/P05/P5`, `IPS_N*` |
| Sensirion SPS30 | PM1/2.5/4/10 + counts + typical size | I²C `0x69` | `sps30_read` | 1 | `SPS30_P0/P1/P2/P4`, `SPS30_N05/N1/N25/N4/N10`, `SPS30_TS` |
| PPD42NS | PM10, PM2.5 (pulse counting) | GPIO D5/D6 | `ppd_read` | 5 | `durP1`, `ratioP1`, `P1`, `durP2`, `ratioP2`, `P2` |
| BME280 | temp, humidity, pressure | I²C `0x77`/`0x76` | `bmx280_read` | 11 | `BME280_temperature/_humidity/_pressure` |
| BMP280 | temp, pressure | I²C `0x77`/`0x76` | `bmx280_read` | 3 | `BMP280_temperature/_pressure` |
| BMP180 | temp, pressure | I²C `0x77` | `bmp_read` | 3 | `BMP_temperature`, `BMP_pressure` |
| DHT22 | temp, humidity | 1-wire-ish D7 | `dht_read` | 7 | `temperature`, `humidity` |
| HTU21D | temp, humidity | I²C `0x40` | `htu21d_read` | 7 | `HTU21D_temperature/_humidity` |
| SHT3x | temp, humidity | I²C `0x44` | `sht3x_read` | 7 | `SHT3X_temperature/_humidity` |
| SCD30 | temp, humidity, CO₂ | I²C `0x61` | `scd30_read` | 17 | `SCD30_temperature/_humidity/_co2_ppm` |
| DS18B20 | temp | OneWire D7 | `ds18b20_read` | 13 | `DS18B20_temperature` |
| DNMS | noise LAeq/min/max | I²C `0x55` | `dnms_read` | 15 | `DNMS_noise_LAeq/_LA_min/_LA_max` |
| GPS (NEO-6M) | lat, lon, altitude, time | serial D5/D6 @9600 | `gps_read` | 9 | `GPS_lat`, `GPS_lon`, `GPS_height`, `GPS_timestamp` |

I²C addresses per [Contributing.md:101-117](../src-original/airrohr-firmware/Contributing.md#L101-L117).
The Readme warns that GPS combined with a PM sensor can crash the firmware.

**Displays** (all I²C, not used in our configuration): SSD1306 and SH1106 OLEDs, LCD1602 and LCD2004
at `0x3F` or `0x27`. Screens rotate every 5 s
([defines.h:61](../src-original/airrohr-firmware/defines.h#L61)), with optional WiFi-info and
device-info screens appended.

## Appendix B — porting notes

Collected from the inline notes above. These are decisions already taken for the ESPHome port; the
port design itself is a separate document.

### Replaced by ESPHome, not reimplemented

- WiFi connection and AP/captive-portal setup → ESPHome `wifi` (with `captive_portal` if wanted)
- mDNS registration → handled by ESPHome
- Firmware updates → ESPHome OTA, flashed deliberately by the user
- Debug output and verbosity levels → ESPHome `logger`
- Restart and factory reset → ESPHome buttons/services
- Runtime configuration in `/config.json` → ESPHome YAML, compile-time
- SDS011 and BME280 drivers → native `sds011` and `bme280_i2c` components
- Temperature correction offset → an `offset:` filter on the temperature sensor. Filters run before
  publishing, so every consumer sees the corrected value, matching airRohr. Anything deriving dew
  point or sea-level pressure must read the **published** temperature, not the raw component value,
  or it will diverge from the original.

### Dropped

- The daily OTA check and the two-stage SPIFFS loader
- The `/config`, `/debug`, `/removeConfig` and `/reset` web routes
- The 28-day forced reboot and the SPIFFS config migration logic

### Reproduced deliberately — custom work in the port

These behaviours are kept because the receiving network expects them, not because they are convenient.

- **The sensor.community request shape stays exactly as airRohr does it**: one POST per sensor,
  prefix-stripped keys (`SDS_P1` → `P1`), `X-PIN` and `X-Sensor` headers, `application/json`, and
  values as JSON **strings with two decimal places**. See §5.
- **Independent on/off toggle per upload target** — sensor.community, madavi and any other
  destination are each switchable, none implied by another.
- **SDS011 keeps its duty cycling and min/max-trimmed averaging** (§4): stopped between cycles, 15 s
  warm-up, 5 s of accumulation, lowest and highest sample discarded before averaging. This is airRohr
  behaviour that sensor.community's data expects, and it is what preserves the laser's service life —
  so the SDS011 is *not* free-running on an ESPHome `update_interval`.
- **Temperature correction is applied to every temperature sensor**, including the four airRohr
  misses (HTU21D, SHT3x, SCD30, BMP180). Correcting the original's inconsistency is intentional.
- **Sea-level pressure and dew point are kept where available**, as **local values only** — exposed
  through the ESPHome web server / Home Assistant, never added to any upload payload, exactly as
  airRohr treats them (§4). Neither has a built-in ESPHome platform; both are template sensors
  computing the formulas from [:1346-1367](../src-original/airrohr-firmware/airrohr-firmware.ino#L1346-L1367),
  and both must consume the **published** (offset-corrected) temperature.

### Simplified deliberately

- **The BME280 becomes a normal polled ESPHome sensor** rather than a once-per-send forced read.
  Nothing in airRohr's handling of it is special beyond the temperature offset, which the `offset:`
  filter already covers. Two details still carry over: the API pin differs by chip (11 for BME280,
  3 for BMP280, §5), and a failed read must be **omitted** from the payload — airRohr sends no key
  at all for a failed sensor, so the port must skip `NaN` rather than serialise it.

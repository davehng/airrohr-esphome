# Plan: ESPHome port of airrohr-firmware → `src-esphome/airrohr.yaml`

> **Status: implemented and compiling.** The port is in
> [../src-esphome/airrohr.yaml](../src-esphome/airrohr.yaml), with
> [../src-esphome/secrets.yaml.example](../src-esphome/secrets.yaml.example) alongside it.
> **Status: complete and live.** Every step of [Verification](#verification) passes. The port runs on
> hardware and publishes to the real sensor.community and madavi endpoints, which accept the data
> (HTTP 201 and 200 respectively). See [Implementation notes](#implementation-notes) for what changed
> along the way and what is worth watching in the long run.

## Context

[airrohr.md](airrohr.md) documents what the original firmware does and records the porting
decisions already taken. This plan implements those decisions as a **single ESPHome YAML file** for
the target hardware: **NodeMCU v2/v3 + SDS011 + BME280, no display**.

The point of the port is to keep the device's *observable behaviour on the network* identical —
sensor.community must not be able to tell the difference — while handing WiFi, OTA, logging,
configuration and local UI to ESPHome. In particular the device keeps the **same sensor identity**
(`esp8266-<chipid>`), so an already-registered airRohr sensor keeps reporting under its existing
registration after reflashing.

Decisions confirmed: file goes in `src-esphome/` (not the upstream checkout in `src-original/`);
Home Assistant API **and** web server; upload targets are **runtime switches**.

## Research findings that shape the design

Checked against the ESPHome sources and docs, not assumed:

- **`sensor.sds011` parses frames but does not duty-cycle the way airRohr does.** Its
  `update_interval` sets the *sensor's own* working period (its firmware wakes ~30 s per period);
  `rx_only` and `update_interval` are mutually exclusive in the schema. So the component is used
  purely as a **frame parser** in continuous reporting mode, and the port drives start/stop itself.
- **airRohr's start/stop frames are reproducible verbatim from YAML.** `SDS_rawcmd()`
  ([utils.cpp:405-422](../src-original/airrohr-firmware/utils.cpp#L405-L422)) builds a 19-byte frame
  `AA B4 <h1> <h2> <h3> 00×10 FF FF <ck> AB` with `ck = h1+h2+h3-2`. `uart.write` sends these
  directly — a documented YAML action, no reliance on ESPHome internals:

  | Command | h1 h2 h3 | checksum |
  | --- | --- | --- |
  | Start (wake) | `06 01 01` | `06` |
  | Stop (sleep) | `06 01 00` | `05` |
  | Working period = continuous | `08 01 00` | `07` |
  | Reporting mode = active | `02 01 00` | `01` |

  (`SDS011Component::set_working_state(bool)` is public and sends the same `0x06` command, but it is
  not part of the YAML API and could change; raw frames are the stable route.)
- **`http_request` fits.** `request_headers` values and `body` are both templatable, so `X-PIN` and
  `X-Sensor` can be lambdas. `esp8266_disable_ssl_support: true` drops the TLS stack — both endpoints
  are plain HTTP port 80 in airRohr's defaults, so nothing is lost and flash is saved.
- **Pins force a software UART.** GPIO5/GPIO4 are not ESP8266 hardware-UART pins, so ESPHome falls
  back to its software implementation at 9600 — the same arrangement airRohr uses, and it leaves
  UART0 free for `logger`.
- **`bme280_i2c`** already does forced-mode one-shot reads; `oversampling: 1x` + `iir_filter: OFF`
  matches airRohr's `setSampling()` ([:5408-5412](../src-original/airrohr-firmware/airrohr-firmware.ino#L5408-L5412)).

## Deliverable

`src-esphome/airrohr.yaml` — no external components, no `includes:` — plus a `secrets.yaml.example`
template.

### Structure

**`substitutions`** — everything a user would have edited on airRohr's `/config` page: device name,
`temp_offset`, `height_above_sealevel`, `sending_interval_s` (145), `warmup_time_s` (15),
`reading_time_s` (5), endpoint URLs, `software_version`.

**Core blocks** — `esp8266` (board `nodemcuv2`), `wifi` (with `ap:` fallback, replacing airRohr's
captive portal), `captive_portal`, `logger`, `api`, `ota`, `web_server`, `http_request`
(`timeout: 10s`, `esp8266_disable_ssl_support: true`), `i2c` (SDA GPIO0/D3, SCL GPIO2/D4),
`uart` (RX GPIO5/D1, TX GPIO4/D2, 9600).

**Sensors**

- `sds011` with `id`, both PM sensors `internal: true` — they are taps, not published entities. Each
  has `on_value` that appends to a global vector **only while the collection window is open**.
- `bme280_i2c`, `update_interval: 60s`, `address: ${bme280_address}`, temperature carrying
  `filters: [offset: ${temp_offset}]`.
- Two `template` sensors `SDS_P1` / `SDS_P2`, `update_interval: never`, published by the cycle script.
- Two `template` sensors for the local-only derived values, both reading the **published**
  (offset-corrected) temperature:
  - dew point — `k3*((k2*T/(k3+T))+log(RH/100)) / ((k2*k3/(k3+T))-log(RH/100))`, `k2=17.62`,
    `k3=243.12` ([:1346-1355](../src-original/airrohr-firmware/airrohr-firmware.ino#L1346-L1355))
  - pressure at sea level — `p_hPa * pow((T+273.15)/(T+273.15+0.0065*h), -5.255)`
    ([:1360-1367](../src-original/airrohr-firmware/airrohr-firmware.ino#L1360-L1367))
- Diagnostics standing in for `/status`: `wifi_signal`, `uptime`, and a `version` text sensor.

**Switches** — `publish_sensor_community` and `publish_madavi`, template switches with
`restore_mode: RESTORE_DEFAULT_ON`, optimistic. Each upload script checks its own switch.

**Globals** — `std::vector<float>` for PM2.5 and PM10 samples, a `bool collecting` gate.

### The measurement cycle

An `interval: ${sending_interval_s}s` runs one `script` (`mode: single`), which makes airRohr's
`millis()` arithmetic explicit and readable:

```text
clear sample vectors, collecting = false
uart.write  Start frame                 → fan on
delay ${warmup_time_s}s                 → frames arrive and are ignored
collecting = true
delay ${reading_time_s}s                → frames accumulate
collecting = false
uart.write  Stop frame                  → fan off
lambda: trimmed mean → publish SDS_P1 / SDS_P2, build payloads
script.execute: upload_all               → per-sensor: empty payloads skipped
```

Trimmed mean, mirroring [:3527-3536](../src-original/airrohr-firmware/airrohr-firmware.ino#L3527-L3536):
with more than 2 samples drop one min and one max, then average; with **zero** samples publish
nothing, so the payload omits the keys entirely, as airRohr does. Sample count is logged each cycle
so the 15 s/5 s windows can be checked against reality.

At `on_boot`: send the working-period and reporting-mode frames, then Stop — airRohr's
`powerOnTestSensors()` sequence ([:5491-5498](../src-original/airrohr-firmware/airrohr-firmware.ino#L5491-L5498)).

### Uploads

A parameterised script (`parameters: {pin: string, payload: string}`) does one POST, so the
sensor.community request shape lives in exactly one place:

```yaml
- http_request.post:
    url: ${sc_url}
    request_headers:
      Content-Type: application/json
      X-Sensor: !lambda 'static const std::string s = esphome::str_sprintf("esp8266-%u", ESP.getChipId()); return s.c_str();'
      X-MAC-ID: !lambda 'static const std::string s = "esp8266-" + esphome::get_mac_address(); return s.c_str();'
      X-PIN: !lambda 'static std::string s; s = pin; return s.c_str();'
    body: !lambda 'return payload;'
```

Header lambdas must return `const char *` — `request_headers` is a
`TemplatableValue<const char *>`, unlike `body`, which takes a `std::string`. The statics own the
storage; the POST is issued synchronously within the action, so the pointers stay valid.

`ESP.getChipId()` is the low 24 bits of the MAC — byte-identical to what the original firmware sends,
which is what preserves the existing sensor.community registration.

The upload script then, guarded by the switches:

1. **sensor.community**, two POSTs — `X-PIN: 1` with keys `P1`/`P2`, and `X-PIN: 11` with
   `temperature`/`humidity`/`pressure` (Pa). Prefixes stripped, as in §5 of the doc.
2. **madavi**, one combined POST, keys unstripped (`SDS_P1`, `BME280_temperature`, …).

A shared value-serialising lambda enforces the wire format from §5: every value a **JSON string**
formatted `%.2f`, and **`NaN` skipped** rather than serialised — the failed-read behaviour the doc
flags for the now-polled BME280.

Combined payload also carries `samples` (SDS sample count this cycle), `interval` and `signal` (RSSI).
`min_micro`/`max_micro` are dropped: they measure airRohr's `loop()` timing, which has no equivalent.

## Risks and how the file handles them

- **Flash budget — measured, not a problem.** `api` + `web_server` + `http_request` all fit
  comfortably: **530 049 of 1 044 464 bytes (50.7%)**, against the original firmware's 67.1%
  ([airrohr-firmware.ino:52-53](../src-original/airrohr-firmware/airrohr-firmware.ino#L52-L53)).
  The planned fallback (`web_server` → `version: 1`, then dropping `captive_portal`) is not needed.
- **RAM.** 37 344 of 81 920 bytes static (45.6%), against the original's 41.8%. That leaves ~44 KB of
  heap for `web_server`, `api` and `http_request` at runtime — adequate, and helped by the TLS stack
  being compiled out, but free heap is worth watching in the logs during the first long run.
- **Blocking POSTs.** `http_request` is synchronous; three POSTs at up to 10 s each sit inside the
  script. Same behaviour as airRohr, and it happens while the fan is already off.
- **Software UART glitches** at 9600 — inherent to the pin choice and identical to airRohr, which
  also drops malformed frames on checksum.
- **GPIO0/GPIO2 are boot-strapping pins.** I²C idle-high with pull-ups keeps them safe, but a shorted
  bus will prevent boot. Noted in a comment in the file.

## Implementation notes

Deviations from the plan as written, decided while building:

- **`update_interval` is omitted from the `sds011` block** rather than set to `0min`. The component's
  default is already 0 min (continuous), and omitting it avoids relying on `0min` passing schema
  validation. The port additionally sends the working-period frame itself at boot, so the sensor ends
  up in continuous mode either way.
- **`esphome::str_sprintf` replaces `std::to_string`** for the chip ID. `std::to_string` is not
  reliably available in the ESP8266 toolchain.
- **ESPHome 2024.8+ is required** — `ota:` as a list of platforms, and `request_headers:` on
  http_request actions (renamed from `headers:` in that release).
- **Endpoint substitutions are full URLs** (`sc_url`, `madavi_url`) rather than split host/path.
- **BME280 pressure is converted from hPa to Pa** in the payload builder. ESPHome's `bme280_i2c`
  publishes hPa; airRohr sends Pa. The sea-level pressure template uses the hPa value directly, as
  the original formula expects.
- **User-Agent differs.** airRohr sends `version/chipid/macid`; the port sends just the version
  string. Identification is via `X-Sensor`, which matches exactly.

Observed on hardware, not defects:

- **`measure_cycle took a long time for an operation (134 ms), max is 50 ms`** is logged every cycle.
  `http_request` is synchronous, so the three POSTs block ESPHome's main loop. Measured at 134 ms
  against a LAN listener and **2447 ms against the live endpoints** — three sequential internet round
  trips. Worst case is the 10 s timeout per request, so roughly 30 s if all three hang. This is the same blocking behaviour the original firmware has, and it happens after the
  fan is already off, so nothing time-critical is affected — but the warning is permanent.
- **ESPHome fires the first `interval:` within 5 s of boot, not after a full interval.**
  `Scheduler::set_interval` offsets the first execution by `min(interval/2, 5s)` chosen at random, to
  avoid a thundering herd. So the first measurement cycle begins almost immediately after power-on —
  exactly when the SDS011 is coldest — rather than 145 s in. This is why cold starts reliably produce
  empty cycles, and why the skip-on-no-samples behaviour below is structural rather than cosmetic.
- **The first cycles after a cold start reported `samples: 0`**, then self-corrected. The SDS011 needs
  several minutes of running before it returns usable readings, which no per-cycle warm-up of a
  sensible length can cover. Rather than lengthen the warm-up, sensors are kept **independent**: a
  cycle with no PM samples omits the `X-PIN: 1` request and the `SDS_*` keys, while the BME280
  request and the madavi payload go out as usual. This matches the original firmware, which drops a
  failed sensor's keys but keeps sending the rest, and it means a dead PM sensor never silences the
  weather data. A `no SDS011 samples this cycle` warning marks those cycles in the log.

Found on first contact with hardware and fixed:

- **The BME280 I2C address was pinned to `0x77`; the actual device is at `0x76`**, so the component
  logged `Communication failed` and was marked FAILED, making every read NaN. The original firmware
  probes `0x77` then falls back to `0x76`; the port cannot probe, so the address is now a
  `bme280_address` substitution, defaulting to the commoner `0x76`. The `i2c: scan: true` output in
  the boot log reports the right value for a given board.

Found on the first `esphome compile` and fixed:

- **`request_headers` lambdas must return `const char *`**, not `std::string` — five lambdas failed
  to convert. Each now holds its value in a function-local `static std::string` and returns
  `.c_str()`. `body:` is unaffected; it is templatable as `std::string`.

Three further defects were found and fixed during a read-back of the generated file:

1. The `on_response` lambda referenced a script parameter. Triggers get their own parameter pack, so
   `pin` is not in scope there — this would have failed to compile.
2. `(int) id(wifi_rssi).state` casts `NaN` before the first WiFi signal update; now guarded.
3. With no valid samples the builder still produced `"sensordatavalues":[]` and posted it. airRohr
   checks `data.length()` and skips; the port now skips per sensor.

## Verification

Steps 1 and 2 **pass**. Steps 3 onward are outstanding — nothing has run on hardware yet.

1. ~~`esphome config src-esphome/airrohr.yaml`~~ — done, schema and substitutions resolve.
2. ~~`esphome compile src-esphome/airrohr.yaml`~~ — done, builds clean at 50.7% flash / 45.6% RAM.
   See Risks above.
3. ~~**Dry run before touching the live API.**~~ Done, against a local Python HTTP listener. Three
   requests per cycle, exactly as designed:
   - `X-PIN: 1` → `{"P1":"4.10"},{"P2":"0.80"}` — prefixes stripped, two-decimal strings
   - `X-PIN: 11` → `temperature`, `humidity`, `pressure` — the last as **101875.55 Pa**, i.e. the
     hPa→Pa conversion is correct
   - madavi → the combined payload with prefixes intact, plus `samples`/`interval`/`signal`
   - `X-Sensor` was `esp8266-<n>`, where `<n>` is the decimal value of the low 24 bits of the
     device's own MAC — verified against the test device. (Worked example: a device with MAC
     `aa:bb:cc:dd:ee:ff` reports `esp8266-14544639`, since `0xDDEEFF` = 14544639.) This is what
     the original firmware sends, so an existing registration is preserved
   - each returned HTTP 200 and was logged as such
4. ~~Confirm sample counts and the trimmed mean.~~ Done, and the outlier rejection is confirmed to
   be doing real work. A cycle with frames PM10 `4.8, 4.6, 4.8, 4.9, 6.7` dropped the 4.6 and the
   6.7 spike and reported **4.83** — the plain mean would have been 5.16. PM2.5 agreed as well
   (1.07 trimmed vs 1.08 plain). Every cycle observed collected exactly 5 samples, as predicted from
   ~1 frame/s across the 5 s window.
5. ~~Derived values.~~ Dew point verified: 24.07 °C / 50.66 %RH produced **13.2 °C**, matching the
   original's formula computed independently. Sea-level pressure **cannot be verified** while
   `height_above_sealevel` is `0.0` — the formula reduces to the identity and returns the measured
   pressure unchanged (observed: 1018.72 hPa against a measured 1018.7). Set a real height to
   exercise it; at 100 m that reading becomes 1030.52 hPa, about +11.8 hPa, matching the usual rule
   of thumb.
6. ~~Live publishing.~~ Done. With `sc_url`/`madavi_url` pointed at the real endpoints and both
   switches on, a cycle produced `sensor.community -> HTTP 201` for both the `X-PIN: 1` and
   `X-PIN: 11` requests — 201 Created is the API accepting the reading — and `madavi -> HTTP 200`.
   The trimmed average was confirmed once more on live data: frames `3.6, 4.1, 4.2, 4.0, 4.5`
   reported **4.10**, against a plain mean of 4.08.

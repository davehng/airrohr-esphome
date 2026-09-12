# airrohr-esphome

An [ESPHome](https://esphome.io) port of the
[airRohr firmware](https://github.com/opendata-stuttgart/sensors-software) — the Arduino firmware
behind the [Sensor.Community](https://sensor.community) citizen air-quality network.

My motivation for doing the port was wanting an easier way to update the firmware (ESPHome is great
for this) and I was having problems with the AirRohr sensor not playing nicely with the 802.11kv
features on my home wifi network. Claude is also really good at rewriting and porting code, so
Claude Code greatly reduced the friction of getting this work done.

The goal for the port is that **the sensor data logger APIs cannot tell the difference**.
The device reports the same values, in the same payload format, under the same sensor identity
as the original firmware - while WiFi provisioning, OTA updates, logging, configuration and
the local UI are handed to ESPHome instead of being reimplemented.

## Target hardware

| | |
| --- | --- |
| Board | NodeMCU v2/v3 (ESP8266) |
| Particulate matter | Nova Fitness SDS011 (serial, D1/D2) |
| Temperature, humidity, pressure | Bosch BME280 (I²C, D3/D4) |
| Display | none |

This is the common Sensor.Community build. The original firmware supports sixteen sensors and seven
upload targets; this port deliberately covers one configuration well rather than all of them.

## What is preserved from the original

- **Sensor identity.** `X-Sensor: esp8266-<chipid>` is derived from the MAC exactly as the original
  computes it, so a sensor already registered with Sensor.Community keeps its registration after
  reflashing.
- **The request shape.** One POST per sensor with the sensor prefix stripped and an `X-PIN` header
  (1 for the SDS011, 11 for the BME280), values as JSON strings with two decimals.
- **SDS011 duty cycling.** The fan is stopped between cycles and started 20 s before each send —
  15 s warm-up, then a 5 s collection window — using the original's own command frames. This is what
  keeps the laser's service life, and it is what the network's data expects.
- **Independent sensors.** A sensor with no reading this cycle is omitted from the payload rather
  than sent as a placeholder, and it does not hold up the others — so while the SDS011 warms up after
  a boot, the BME280 keeps reporting.
- **Trimmed averaging.** The lowest and highest sample of each cycle are discarded before averaging.
- **Temperature correction** as a configurable offset, applied to temperature only.

## What is different

- Updates go through ESPHome's OTA rather than the original's daily self-update and two-stage loader.
- Configuration lives in YAML rather than a web form backed by a JSON file in flash.
- Publishing to Sensor.Community and to madavi.de are **independent runtime switches**, so the device
  can run without publishing anywhere.
- The temperature correction is applied to every temperature sensor; the original applies it to only
  three of its seven.
- Dew point and sea-level pressure are exposed locally (Home Assistant / web server) and, as in the
  original, are never uploaded.

## Layout

| Path | Contents |
| --- | --- |
| [`src-esphome/`](src-esphome/) | The port: `airrohr.yaml` and a `secrets.yaml.example` template |
| [`src-original/`](src-original/) | The upstream `sensors-software` repository, as a git submodule — reference only, never modified |
| [`docs/airrohr.md`](docs/airrohr.md) | What the original firmware does — measurement cycle, payload formats, configuration, OTA |
| [`docs/portingplan.md`](docs/portingplan.md) | How the port was designed, what changed during implementation, and how to verify it |
| [`docs/lessonslearned.md`](docs/lessonslearned.md) | What this port taught us — framework gotchas, what counts as evidence, where the documentation drifted |

## Getting started

```sh
git clone --recurse-submodules https://github.com/davehng/airrohr-esphome.git
cd airrohr-esphome/src-esphome
cp secrets.yaml.example secrets.yaml   # then fill in your WiFi and keys
esphome run airrohr.yaml
```

Requires ESPHome 2024.8 or newer.

`secrets.yaml` is gitignored — only the template is tracked.

The original firmware in `src-original/` is a submodule, and is needed only if you want to read the
sources the port is based on. If you cloned without `--recurse-submodules`, fetch it with:

```sh
git submodule update --init
```

**Before publishing to the live API**, leave both `Publish to …` switches off and check the logs for
a few cycles: each should report roughly five SDS011 samples. `docs/portingplan.md` describes a dry
run against a local HTTP listener to inspect the exact payload first.

## Status

Working. Runs on a NodeMCU v2 (50.7% flash, 45.6% RAM) and publishes to the live Sensor.Community
and madavi endpoints, which accept the data. The duty cycle, trimmed averaging, unit conversions,
payload shapes and device identity have all been verified against real readings — see
[docs/portingplan.md](docs/portingplan.md) for the evidence.

Long-run behaviour is still unproven: it has not yet been left running for days, so heap
fragmentation and reliability across reboots are unmeasured.

## License

**GPL-3.0**, the same licence as the firmware it is derived from — see [LICENSE](LICENSE).

The port reproduces protocol details, command frames and algorithms read directly from the original
GPL-3.0 sources, so it is a derivative work and carries the same terms.

## Attribution

The original airRohr firmware is the work of Code for Stuttgart, Sensor.Community contributors and
Dirk Mueller; the copy in [`src-original/`](src-original/) is theirs, unmodified.

The ESPHome port and the documentation in this repository were written by
**[Claude](https://claude.com/claude-code)** (Anthropic), working from the original firmware sources
under the direction of the repository owner.

# CLAUDE.md

Guidance for Claude Code working in this repository.

## What this is

An ESPHome port of the airRohr firmware for Sensor.Community air-quality sensors. The entire
deliverable is one file: `src-esphome/airrohr.yaml`. Target hardware is a NodeMCU v2/v3 (ESP8266)
with an SDS011 particulate sensor and a BME280, no display.

The port is complete, running on hardware, and publishing to the live endpoints.

## Layout

| Path | Notes |
| --- | --- |
| `src-esphome/airrohr.yaml` | The port. Nearly all work happens here. |
| `src-esphome/secrets.yaml` | Gitignored. **Never commit, never print its contents.** |
| `src-original/` | Git submodule of the upstream GPL-3.0 project. **Read-only — never modify.** |
| `docs/airrohr.md` | What the original firmware does. The reference for wire formats. |
| `docs/portingplan.md` | Design decisions, deviations, and the verification record. |
| `docs/lessonslearned.md` | Framework gotchas and what went wrong. Read before non-trivial changes. |

## The governing constraint

Sensor.Community must not be able to tell this apart from the original firmware. These are
invariants, not preferences — changing any of them changes what a public dataset receives:

- **Device identity.** `X-Sensor: esp8266-<n>` where `<n>` is `ESP.getChipId()`, the low 24 bits of
  the MAC. This is what the network knows the device by; an existing registration depends on it.
- **One POST per sensor**, with the sensor prefix stripped (`SDS_P1` → `P1`) and an `X-PIN` header:
  1 for the SDS011, 11 for a BME280, 3 for a BMP280.
- **Values are JSON strings with two decimals**, never numbers. `NaN` is omitted entirely rather than
  serialised — a sensor with no reading contributes no keys.
- **A payload with no values is not sent at all**, rather than sent with an empty array.
- **Pressure is uploaded in Pa.** ESPHome's BME280 publishes hPa; the payload builder multiplies by
  100. Sea-level pressure and dew point are local-only and must never be uploaded.
- **SDS011 duty cycling and trimmed averaging** are reproduced deliberately: fan stopped between
  cycles, 15 s warm-up, 5 s collection, then discard the single lowest and highest sample before
  averaging. This protects the laser and is what the network's data expects.
- **Sensors are independent.** A cycle with no PM samples still uploads the BME280 reading, and vice
  versa. Do not couple them.

`docs/airrohr.md` §5 has the exact wire format with worked examples. Consult it before touching the
payload builder.

## Before changing airrohr.yaml

- Read `docs/lessonslearned.md` first. It records ESPHome-specific traps already paid for — notably
  that `request_headers` lambdas must return `const char *` while `body:` takes `std::string`, and
  that `interval:` fires within 5 s of boot rather than after a full interval.
- Keep anything a user might tune in `substitutions:` at the top, not buried in a component.
- Comments should explain *why*, and cite the original by file and line where behaviour is being
  reproduced (e.g. `airrohr-firmware.ino:3527-3536`).
- Preserve the GPL-3.0 header. This repo is GPL-3.0 because it derives from GPL-3.0 sources.

## Validating changes

ESPHome is **not installed** in this development environment, and neither is PyYAML — there is no way
to validate YAML locally. The user runs `esphome config` and `esphome compile` and reports back. Do
not claim a change is validated; say plainly that it is unverified.

When a change affects uploads, the order is:

1. `esphome compile` — the only thing that checks the lambdas.
2. **Dry run.** Point `sc_url` / `madavi_url` at a local HTTP listener and inspect the actual bytes.
3. Flash with the publish switches off and check the logs for a few cycles.
4. Only then point back at the live endpoints.

Never point a test build at `api.sensor.community` with the real device identity. Published readings
land in a public dataset attributed to a registered sensor, and there is no undo.

## Evidence, not verdicts

When recording a verification in `docs/portingplan.md`, write down the data, not the conclusion.
"Frames 4.8, 4.6, 4.8, 4.9, 6.7 → reported 4.83, plain mean would be 5.16" is checkable; "trimming
verified" is not. Ask whether the test could have failed — an early trimmed-average check passed on
data where the trimmed and untrimmed means were identical, and so proved nothing.

Keep `docs/portingplan.md` current when behaviour changes. It is edited by small patches and drifts;
a full read-through periodically is worth it.

# Lessons learned porting airRohr to ESPHome

Written during and after the port documented in [portingplan.md](portingplan.md). Everything here
comes from something that actually went wrong. These lessons learned were initially written
by Claude (Opus 5).

## Porting to a framework

**A framework's config schema is not its type system.** The single compile failure in this project
came from trusting the wrong source. ESPHome validates `http_request` headers as
`cv.templatable(cv.string)` in Python, which reads as "a string, possibly templated" — but the C++
side is `TemplatableValue<const char *>`, while the neighbouring `body:` really is `std::string`.
Five lambdas returned `std::string` and none of them compiled. The Python schema describes *what YAML
is accepted*, not *what the lambda must return*. When a lambda's return type matters, read the C++
header or just compile.

**When the original probes for a value, that value varies in the field.** The firmware tries I²C
`0x77`, then falls back to `0x76`. Porting that as a hardcoded `0x77` produced a sensor marked FAILED
and every reading NaN, on the first real device. A probe in the source is a signal that the author
met both cases; collapsing it to a constant discards that knowledge. It is now a substitution with
the commoner value as the default.

**Check the framework's timing semantics rather than assuming them.** `interval: 145s` does not mean
"first run at 145 s". ESPHome's scheduler offsets the first execution by a random
`min(interval/2, 5s)` to avoid everything starting at the same time, so the first measurement cycle
begins within five seconds of boot — exactly when the SDS011 is coldest and returns nothing. An
entire class of cold-start behaviour followed from a scheduling detail that was never going to
appear in the component documentation.

**Unit conventions differ at every boundary.** ESPHome's BME280 publishes hPa; the Sensor.Community
API expects Pa. Nothing catches this — both are plausible numbers, and a 100× pressure error would
have been accepted by the API and quietly wrong in the public dataset forever. Found by reading both
sides, not by testing.

**Identity is part of the contract.** The single most important line in this port computes
`ESP.getChipId()`, the low 24 bits of the MAC, because that is what the original sends as `X-Sensor`
and therefore what the network knows the device by. Reproducing it meant an existing registration
survived the reflash. In any port that talks to a service, find the values that constitute identity
before designing anything else.

**Reusing a field name silently changes its meaning.** The original's `samples` counts `loop()`
iterations — tens of thousands per interval. The port has no loop to count, so it reports the number
of sensor readings: 5. Same key, same type, entirely different quantity, and the receiving service
graphs it either way. This was kept deliberately, for a diagnostic endpoint only, and written down.
The failure mode to avoid is doing it *without* noticing.

## Validation and evidence

**A test that could not have failed is not evidence.** The first trimmed-average check looked like a
clean pass: five samples, computed 4.10, device reported 4.10. But the plain mean of those samples
was *also* 4.10 — the data could not distinguish a working implementation from one that ignored the
trimming entirely. Confirmation only arrived with a later cycle containing a 6.7 µg/m³ spike, where
trimmed (4.83) and untrimmed (5.16) diverge. Before recording a check as passed, ask what result
would have revealed the bug.

**Some checks are structurally incapable of discriminating.** Sea-level pressure was tested at a
height of 0.1 m, where the barometric formula shifts the result by 0.012 hPa. The observed value
matched the computed one exactly — and would have matched almost as well with a wrong exponent. The
deployment site is genuinely at sea level, so this code path is an identity in normal operation.
Recorded as weakly tested rather than as verified.

**Dry-run against something you control before touching a shared service.** Pointing the endpoints
(sc_url and madavi_url) at a local Python listener cost about two minutes and made the payloads
inspectable byte by byte. The data being published goes into a public dataset under a real
sensor's identity; there is no meaningful undo. The dry run found no defects, which is not an
argument against having done it.

**Buffered logs lie about time.** A cycle appeared to complete 11.6 s after boot, when the cycle
physically takes 20 s. Log lines emitted before the log transport connects are flushed on connect and
stamped with the *receive* time. Relative order survives; absolute timing does not. Do not reconstruct
timing from the boot-phase section of a remote log.

**Distinguish expected noise from defects, in writing.** `measure_cycle took a long time for an
operation (2447 ms)` appears every single cycle. It is inherent to synchronous HTTP in an event loop,
it matches how the original firmware behaves, and it is harmless here because it happens after the
fan is off. Written down as expected, it is a known characteristic; left undocumented, it is a bug
someone will chase in six months.

## Working from original source

**Prefer the source to its documentation.** The upstream README states the configuration access point
is unencrypted. The code sets a default password. Both were written in good faith; only one is what
the device does.

**Reproduce protocol bytes rather than reimplementing behaviour.** The SDS011 start/stop commands are
sent as the original's exact 19-byte frames, checksum and all, via a documented `uart.write` action.
The alternative — calling a public-but-undocumented C++ method on the ESPHome component — would have
worked today and broken on some future release. Copying the bytes made the port depend on the
sensor's protocol, which is fixed, instead of on a framework's internals, which are not.

**A behavioural quirk is not automatically worth porting.** The original applies its temperature
correction to three of its seven temperature sensors. That is an oversight, not a design, and the
port applies it to all of them. Conversely the duty cycling and trimmed averaging *were* reproduced
exactly, because the receiving network's data depends on them. Each quirk needs the question asked
out loud: is this intentional?

## Process

**State the consequence of an instruction before implementing it.** "If we get 0 samples then don't
send anything" is unambiguous as written, and implementing it literally coupled the BME280 to the
SDS011 — a dead particulate sensor would have silenced temperature, humidity and pressure too. The
consequence was flagged in the same message as the implementation, and the decision was reversed a
minute later. The cost of surfacing it was one sentence; the cost of not surfacing it would have been
a silent data loss discovered months later.

**Documents edited by patch drift.** Over roughly twenty small edits, [portingplan.md](portingplan.md)
accumulated a duplicated status banner, a stale "cannot be verified" claim that had since been
verified, two lists that omitted later additions, and a cross-reference to behaviour that had been
replaced. None were visible while editing a single section. A full read-through found eight in one
pass — worth doing before anyone else relies on the document.

**Record the evidence, not just the verdict.** "Trimmed mean verified" is worth very little six months
on. "Frames 4.8, 4.6, 4.8, 4.9, 6.7 → reported 4.83, plain mean would be 5.16" can be re-checked by
anyone, and reveals its own limitations.

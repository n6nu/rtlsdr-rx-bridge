# RTL-SDR RX Bridge — Release Notes

## v0.99.3 — spectrum waterfall toggle (2026-05-02)

The built-in spectrum / waterfall display can now be turned off from
the **View menu** (or the **Ctrl+W** shortcut). Useful when you don't
need the visual debugging and would rather not pay the CPU cost.

- View → "Show spectrum waterfall" — checkable, default on.
- New CLI flag `--no-waterfall` launches with the display off and
  persists the choice to INI key `gui/waterfall_enabled`.
- When off, three layers of work are skipped: per-IQ-buffer
  `FftEngine::pushIq()` (the per-sample int8→float + Hann window
  multiply at the full 2 Msps rate), the widget paint events, and
  the 20 Hz row-poll timer. Roughly 2–5 % of one CPU core saved.
- Default ON for first installs and for upgrades — no surprise
  change for existing testers.

Drop-in upgrade from v0.99.2; no INI migration.

## v0.99.2 — beta (2026-04-30)

Feature-parity release with the SDRplay sibling, plus shared GUI code
via Phase 1b refactor.

- **Transverter offset** for IF-transverter / upconverter setups. New
  Settings → "Transverter offset" field (signed MHz). The SDR is tuned
  to *(WSJT-X dial + offset)* while the GUI, WSJT-X, QMAP, and the
  LinradServer header all keep showing the operating dial.
  CLI: `--transverter-offset <MHz>`. INI key:
  `radio/transverter_offset_hz`.
- **Manual SDR frequency override.** Settings checkbox + freq field;
  decouples the bridge from the WSJT-X dial for QMAP-priority
  observation. WSJT-X narrowband decode only works when WSJT-X dial =
  manual freq. CLI: `--manual-freq <MHz>`. INI keys:
  `radio/manual_freq_override`, `radio/manual_freq_hz`.
- **Periodic streaming-stats log line** every 5 seconds.
- **Frequency display sourced from the bridge's actual operating freq**
  — populates correctly at startup before WSJT-X broadcasts.
- **High-contrast IF readout** under the dial display when transverter
  offset is non-zero.
- **Phase 1b refactor**: `RxMainWindow` and `RxSettingsDialog` now
  live in `bridge-core/` and are shared with the HackRF and SDRplay
  sibling apps.

## v0.99.1 — beta (2026-04-29)

- **Auto direct-sampling switch** (tester request). Settings → new
  checkbox **"Auto: Q-channel below 25 MHz"** above the existing
  Direct-sampling combo. When enabled, the bridge automatically flips
  the dongle into Q-channel direct-sampling mode whenever the WSJT-X
  dial drops below 25 MHz, and back to standard quadrature mode at
  25 MHz and above. The manual combo greys out while auto is active.
  Off by default — existing v0.99.0 INIs come up unchanged after
  upgrade. CLI flag: `--direct-sampling-auto`.
- Persisted INI key: `rtlsdr/direct_sampling_auto` (bool).

## v0.99.0 — first beta (2026-04-28)

### Known issues in this build

- **Residual I/Q image on the Linrad / QMAP wideband path.** A real signal
  at offset +X kHz from the dial appears as a faint mirror at −X kHz with
  only ~10 dB rejection. The HackRF RX Bridge gets >40 dB on the same
  code path; the RTL-SDR's image isn't being suppressed by the adaptive
  I/Q balance correction. The image tracks the dial frequency, so it's a
  baseband artifact (not a fixed birdie or external interference). QMAP
  may decode the same Q65 message at both +X and −X bins. Toggling the
  Settings → "I/Q balance correction" checkbox does not change the
  rejection, so the issue is likely upstream of the balancer (the RTL-SDR
  output isn't a true I/Q-imbalanced quadrature signal in the way the
  blind α / sin(φ) estimator assumes — under investigation). This does
  **not** affect dial-following, WSJT-X RX audio, or single-bin Q65
  decodes; please report any other issues you find regardless.

First release of the **RTL-SDR RX Bridge** — a Windows-only companion app
that lets a $25 RTL-SDR dongle add wideband Q65 (QMAP) reception alongside
an existing real radio (IC-905, IC-705, FT-991A, etc.) without disrupting
the real radio's TX setup. Sibling to the **HackRF RX Bridge** — same
shape, same WSJT-X / QMAP integration, different (cheaper, lower
performance) front end.

### Use case

Your IC-905 (or any rig with a normal CAT/audio interface) handles TX
and narrowband RX as it always has, controlled by WSJT-X via Hamlib. An
**RTL-SDR** — fed from a splitter on the same antenna, or a separate
RX-only antenna — runs alongside the real rig as a **wideband observer**.
This bridge:

- Listens to **WSJT-X UDP messages** (port 2237 by default) for the
  current dial frequency, mode, and transmit state
- Tunes the RTL-SDR to match
- Demodulates SSB to **VB-Audio Virtual Cable Line 1** so WSJT-X's
  "Sound input (RX)" sees the RTL-SDR audio
- Streams **96 kHz IQ to QMAP** (UDP 50004) for wideband Q65 decode
- Stops RTL-SDR RX during WSJT-X TX so the local-TX bleed-through can't
  slam the front end, and (optionally) drives the bias tee on/off to
  match WSJT-X's TX state for transverter sequencer / LNA power

WSJT-X CAT continues to control the real radio. WSJT-X's "Sound
output (TX)" still goes to the rig's USB audio interface as before.
Only the **RX audio path** is replaced with bridge-fed audio from the
RTL-SDR.

### What an RTL-SDR can and can't do for weak-signal work

The RTL-SDR is an 8-bit, 2.4 MHz-max, R820T/T2-front-end SDR designed
for casual scanning. Useful properties for this bridge:

- **Frequency range**: 24 MHz – 1.7 GHz native, plus HF via the
  direct-sampling switch (RTL-SDR.com V3+) — covers 6 m, 2 m, 70 cm
  meaningfully; 23 cm only marginally (sensitivity drops above ~1.5 GHz)
- **2.048 Msps default** — easily wide enough for the 96 kHz QMAP
  wideband stream
- **Bias tee** (RTL-SDR.com V3+) — drives an external LNA or transverter
  sequencer at 4.5 V on the SMA centre conductor

Limitations relative to the HackRF:

- **8-bit ADC** vs HackRF's 8-bit (similar) but the RTL-SDR's noise
  figure and IMD3 corner are noticeably worse in absolute terms
- **2.4 Msps max** vs HackRF's 20 Msps — fine for 96 kHz QMAP, not
  enough for any future wideband-search modes
- **No TX path** — this is fundamental, the RTL-SDR is RX-only silicon

For weak-signal work on 6 m / 2 m / 70 cm with a decent external LNA,
an RTL-SDR.com V3+ on a clean band is genuinely usable. On 23 cm or
in a strong-RF environment, a HackRF RX bridge will outperform it.

### Installation note (read first)

The installer is **not code-signed** and is **64-bit only**
(Windows 10 / 11 x64). On first launch on a fresh Windows machine you
will see Microsoft Defender SmartScreen warn:

> Windows protected your PC.
> Microsoft Defender SmartScreen prevented an unrecognized app from
> starting.

Click **More info → Run anyway**. You should only see this once per
binary. The same warning may appear once on the installed
`rtlsdr-rx-bridge.exe`; handle it the same way.

### What you'll need to install separately

- **RTL-SDR USB driver (WinUSB)** — the installer offers to launch
  Zadig at the end to set this up automatically. Skip if `rtl_test`
  on your machine already prints tuner / sample-rate without an error.
- **VB-Audio Virtual Cable** — <https://vb-audio.com/Cable/>. Provides
  the `Line 1` virtual sound device the bridge feeds.
- **WSJT-X 2.7+** — for FT8 / FT4 / Q65 narrowband decoding.
- **QMAP 0.6+** — for wideband Q65. Set Network input = enabled, UDP
  port = 50004.

### WSJT-X configuration

| Setting | Value |
|---|---|
| Radio | your real rig, via Hamlib (Settings → Radio → choose your rig and CAT method — *not* a `Hamlib NET rigctl` pointing at this bridge) |
| PTT method | CAT |
| Sound output (TX) | the real rig's USB audio interface |
| Sound input (RX) | `Line 1 (Virtual Audio Cable)` |
| Settings → Reporting → "Accept UDP requests" | **enabled**, port `2237` |

That last item is required — without it WSJT-X doesn't broadcast its
status messages and the bridge has no way to know the dial frequency.

Launch order: **real rig → WSJT-X → RTL-SDR RX Bridge → QMAP**.

### Features

- **WSJT-X UDP listener** (port 2237) for dial freq, mode, and TX
  state. Same protocol GridTracker / JTAlert / Log4OM use; works
  cross-machine.
- **RTL-SDR RX path** at 2.048 Msps with the standard `uint8 → int8`
  centre-of-128 conversion baked into the driver layer, so the
  downstream DSP shares the HackRF bridge's signed-int8 contract.
- **SSB demod** via Hilbert phasing to VB-Cable Line 1 for WSJT-X.
- **96 kHz IQ wideband stream** to QMAP via UDP 50004 (Linrad
  protocol), with the same `(-1)^n·conj()` transform and adaptive
  I/Q balance correction as the HackRF bridges — image rejection
  past −40 dBc on broadband signals.
- **RX-blanking on WSJT-X TX**: RTL-SDR RX is stopped while WSJT-X
  is keying the real rig, so the local-TX bleed-through doesn't
  hammer the front end.
- **Bias-tee follows WSJT-X TX**: if you have the bias tee enabled
  (manual toggle), the 4.5 V on the SMA centre conductor follows
  WSJT-X's transmit state for transverter sequencer / external-LNA
  power.
- **Settings dialog**: tuner gain (snapped to the librtlsdr-supported
  steps for your tuner — R820T exposes 29 steps from −1 dB to
  +49.6 dB), tuner-AGC, RTL2832 IF AGC, bias tee, direct sampling
  (HF), PPM correction, RX audio device, software RX audio gain
  (Windows WASAPI compensation), Linrad output gain, I/Q balance
  toggle with live α / φ / native-rejection diagnostics.
- **GUI status panel**: dial freq from WSJT-X, mode, WSJT-X UDP-link
  state, RTL-SDR status, RX peak meter, waterfall.

### Command-line options

- `--tuner-gain <dB>` — RTL-SDR tuner gain, snapped to nearest supported
  step (R820T: −1 to +49.6 dB)
- `--auto-gain` — tuner-internal auto-gain
- `--agc` — RTL2832 IF AGC (separate from tuner gain mode)
- `--bias-tee` — RTL-SDR.com V3+ bias tee (4.5 V on SMA centre conductor)
- `--direct-sampling <0|1|2>` — HF mode: off / I-channel / Q-channel
- `--ppm <ppm>` — frequency correction (RTL2832 reference oscillator)
- `--sample-rate <sps>` — sample rate (default 2048000; valid 225001–
  300000 or 900001–3200000)
- `--rx-device <name>` — audio output device (default Line 1)
- `--linrad-gain <dB>` — Linrad/QMAP digital output gain (default 20)
- `--wsjtx-port <port>` — WSJT-X UDP listener port (default 2237)
- `--no-gui` — headless mode
- `--console` — open a debug console with full stderr log

`--help` shows the full set.

### Reporting

Send observations / decodes / bug reports to
**<n6nu@arrl.net>**. Useful info to include:

- RTL-SDR vendor / tuner type (`rtl_test` output)
- Windows version
- WSJT-X version + which real rig you're using
- The bridge log: relaunch with `rtlsdr-rx-bridge.exe --console`,
  reproduce the issue, copy/paste the console output
- For QMAP issues, also `qmap.ini` and a wideband-waterfall
  screenshot

### License

Copyright (C) 2026 **Andreas Junge, N6NU** &lt;<n6nu@arrl.net>&gt;.
Licensed under the **GNU General Public License v3 or later** —
see [`LICENSE`](LICENSE). Bundled third-party components are
documented in [`THIRD_PARTY_LICENSES.md`](THIRD_PARTY_LICENSES.md).
**No warranty.** Install and run at your own risk.

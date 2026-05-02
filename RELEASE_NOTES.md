# RTL-SDR RX Bridge — Release Notes

## v1.0.0 — first stable (2026-05-02)

Promoted out of beta. RX-only RTL-SDR observer for QMAP wideband
Q65 + WSJT-X RX audio has been verified end-to-end on 2 m and
70 cm with multiple dongles. The 0.99.x line ends here; future
development opens a 1.x series.

Cumulative since v0.99.8:

- **Waterfall span now matches the actual IQ rate** (bridge-core
  fix). Display labels follow the real sample rate instead of the
  hardcoded 2 MHz default.

No INI / migration changes; v1.0.0 is a drop-in upgrade from v0.99.8.

## v0.99.8 — multi-instance support (multi-band ops) (2026-05-02)

Run two (or more) bridges side-by-side — different dongles, different
WSJT-X instances, different QMAP instances — without their settings
clobbering each other. Concrete use case: two RTL-SDRs feeding two
WSJT-X / QMAP pairs on 2304 / 2320 MHz subbands.

- New `--instance <name>` CLI flag. When set, the INI file, window
  title, and taskbar entry are namespaced. Two desktop shortcuts:

  ```
  "C:\Program Files\RTL-SDR RX Bridge\rtlsdr-rx-bridge.exe" --instance 2304
  "C:\Program Files\RTL-SDR RX Bridge\rtlsdr-rx-bridge.exe" --instance 2320
  ```

  produce two independent INIs (`RTL-SDR RX Bridge - 2304.ini` and
  `RTL-SDR RX Bridge - 2320.ini`) under `%APPDATA%\Roaming\n6nu\`.
  Default-instance launches (no flag) keep the existing
  `RTL-SDR RX Bridge.ini` unchanged.
- New **Settings → "Linrad TCP port"** and **"Linrad UDP port"**
  spinboxes. Defaults still 49812 / 50004. For multi-instance
  setups, increment per bridge (49813/50005, 49814/50006, …) so
  two QMAPs can talk to two bridges on the same machine. CLI:
  `--linrad-tcp-port`, `--linrad-udp-port`. INI keys:
  `linrad/tcp_port`, `linrad/udp_port`. Linrad ports take effect on
  the next launch.
- New **`--device-index <n>`** CLI flag plus `rtlsdr/device_index`
  INI key for picking which RTL-SDR dongle this instance opens when
  multiple are plugged in (0 = first found, 1 = second, …).
- Bridge-core change — same multi-instance / Linrad-port rows ship
  in all RX-only sibling apps (HackRF / SDRplay / AirSpy /
  Malachite) at v0.99.4 / v1.0.3 / v0.99.2 / v0.99.1.

**Multi-instance workflow (2× RTL-SDR, 2× WSJT-X, 2× QMAP):**

1. Plug in dongle A (index 0) and dongle B (index 1).
2. Launch WSJT-X #1 on 2 m, UDP Server port = 2237. Launch
   WSJT-X #2 on 70 cm, UDP Server port = 2238.
3. Launch QMAP #1 listening on UDP 50004; QMAP #2 on UDP 50005.
4. First bridge shortcut: `--instance 2m`. Open Settings → dongle
   index 0, audio = VB-Cable Line 1, WSJT-X port 2237, Linrad
   TCP 49812, Linrad UDP 50004.
5. Second bridge shortcut: `--instance 70cm`. Settings → dongle
   index 1, audio = VB-Cable Line 2, WSJT-X port 2238, Linrad
   TCP 49813, Linrad UDP 50005.

Both bridges run independently. Window titles include the instance
name so they're easy to tell apart in alt-tab and on the taskbar.

Drop-in upgrade from v0.99.7. Existing single-instance INIs continue
to work unchanged.

## v0.99.7 — installer bug-fix: Zadig actually launches now (2026-05-02)

The optional **"Install RTL-SDR USB driver (WinUSB via Zadig)"** task in
the v0.99.6 (and earlier) installer never actually ran Zadig. The
`zadig.exe` payload was bundled inside `setup.exe` but never extracted
to disk during install, so the `[Run]` entry that was supposed to
launch it was silently a no-op (the entry's `skipifdoesntexist` flag
hid the missing-file error). Symptom: fresh installs without a
pre-existing WinUSB binding produced "RTL not found" at bridge launch.

The bug was an Inno Setup `dontcopy` flag on the `zadig.exe` source
line — that flag suppresses automatic extraction. v0.99.7 removes
the flag so Zadig is extracted to `{tmp}` for the duration of the
install (and cleaned up after).

**No code changes** — only the installer is different. If you
already manually ran Zadig on v0.99.6 and have a working WinUSB
binding, you can skip this update.

Drop-in upgrade from v0.99.6.

## v0.99.6 — configurable WSJT-X UDP port (multi-instance ops) (2026-05-02)

Tester request: support multi-band setups where multiple WSJT-X
instances run on different UDP ports (2237, 2238, 2239 …) and each
bridge instance follows the matching one. The `--wsjtx-port <port>`
CLI flag already supported this on launch; v0.99.6 adds a GUI
control + persistence + live re-bind without restart.

- New **Settings → "WSJT-X UDP port"** spin box (1024–65535, default
  2237). Persisted to INI key `wsjtx/udp_port`.
- The bridge re-binds the UDP socket immediately on Apply when the
  value changes — no app restart needed. Log line confirms:
  `[Settings] WSJT-X UDP port: 2237 → 2238 (re-binding)`.
- The CLI flag `--wsjtx-port` now defaults to the INI-stored value
  (still 2237 for fresh installs), so a port set via Settings is
  honored on subsequent CLI launches too.
- Bridge-core change — same Settings row appears in all RX-only
  sibling apps (HackRF / SDRplay / AirSpy / Malachite) at their
  next version bump.

**Multi-band workflow:**

1. WSJT-X #1 → Reporting → UDP Server port = 2237 (default), 2 m.
2. WSJT-X #2 → Reporting → UDP Server port = 2238, 70 cm.
3. First bridge instance (default) follows WSJT-X #1.
4. Second bridge instance: Settings → WSJT-X UDP port = 2238 →
   Apply. Each bridge feeds its own QMAP instance.

Drop-in upgrade from v0.99.5.

## v0.99.5 — fix "fuzzy" int16 stereo audio (Win7 / Qt5) (2026-05-02)

Tester report against the Win7 v0.99.4 build: WSJT-X RX audio sounded
"fuzzy" (distorted, with a hash overlay on the wanted signal); the
QMAP UDP wideband path was clean. Same setup on Win11 was clean.

**Root cause** in `bridge-core/QtAudioBridge.cpp`'s int16 output
branch: the loop only wrote one `int16_t` per audio frame even when
the device negotiated **int16 stereo** (2-channel). The audio buffer
was `Qt::Uninitialized`-allocated and only half-filled — every other
sample slot was uninitialised memory streaming directly into the
audio device.

**Why the bug only surfaced on Win7 / Qt5**: VB-Audio Virtual Cable
on Win7 reports its preferred audio format as `int16 stereo`, so the
bridge's format-negotiation accepted that and the broken int16 branch
ran. On Win11 / Qt6, the same VB-Cable reports `float stereo` as
preferred — the bridge accepted that instead, and the (correct)
float branch wrote both channels, dodging the bug entirely.

**Fix**: the int16 branch now fills every channel with the same mono
signal, exactly like the float branch does. This is a `bridge-core`
fix — every sibling app (HackRF / RTL-SDR / SDRplay / AirSpy /
Malachite) benefits any time it lands on int16 stereo output. On Qt6
builds where the device prefers float, the behaviour is unchanged.

Drop-in upgrade from v0.99.4. Win11 audio is unchanged (was correct
already).

## v0.99.4 — Settings → Reset frequency defaults + apply() leak fix (2026-05-02)

Tester report against v0.99.3: "the PLL of the RTL-SDR doesn't lock
properly; the bridge displays at startup the freq that's in 'Manual
SDR frequency' even though the checkbox isn't ticked."

Two fixes shipped together:

- **`Settings → "Reset frequency settings to defaults…"` button.**
  Clears `radio/manual_freq_override`, `radio/manual_freq_hz`, and
  `radio/transverter_offset_hz`, retunes the SDR to the current
  WSJT-X dial (or the persisted dial if WSJT-X hasn't been heard
  yet). User-recourse for testers who end up with stale freq state
  in their INI from earlier experiments — typically a manual
  override left enabled, or a non-zero transverter offset they've
  forgotten about. Radio-specific settings (gain, AGC, bias-T,
  notches, antenna, direct sampling, PPM) are NOT touched.
- **`Settings → Apply` no longer accumulates stale `manual_freq_hz`
  values.** Previously, every Apply wrote whatever was in the
  manual-freq spin box to `radio/manual_freq_hz`, regardless of
  whether the override checkbox was ticked. The spin box prefilled
  with the current operating freq when no manual was saved, so
  even a no-op Apply landed a value in INI — the runtime
  conditional masked it (override was off so the value was
  ignored), but on the next Settings open the spin box would
  display the stale value, looking like a bug. v0.99.4 only writes
  `manual_freq_hz` to INI when override is actually on, and
  removes the key when override is off.

If you've been running v0.99.2 or v0.99.3 and the bridge feels
"stuck" at a wrong frequency, click the new Reset button once.
You don't need to delete the INI by hand.

Drop-in upgrade from v0.99.3.

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

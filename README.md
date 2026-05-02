# RTL-SDR RX Bridge — Beta Releases

Windows installer downloads for the **RTL-SDR RX Bridge** — a Qt6
C++ application that adds wideband Q65 (QMAP) reception via a
$25 RTL-SDR dongle to a station already running a real radio
(IC-905, IC-705, FT-991A, etc.) for TX. Sibling to the HackRF
RX Bridge, with a different (cheaper, lower-performance) front
end.

The bridge listens to **WSJT-X UDP** for the dial frequency, tunes
the RTL-SDR to match, demodulates SSB to **VB-Cable Line 1** for
WSJT-X RX audio, and streams **96 kHz IQ to QMAP** for wideband Q65
decode. Your real rig keeps doing TX (and narrowband RX, if you
prefer). No interference with your existing CAT / audio setup.

Author: **Andreas Junge, N6NU** &lt;<n6nu@arrl.net>&gt;.

---

## Latest release — v1.0.0 (stable)

| Variant | Download |
|---|---|
| **Windows 10 / 11** (installer) | **[rtlsdr-rx-bridge-1.0.0-setup.exe](rtlsdr-rx-bridge-1.0.0-setup.exe)** |

Promoted out of beta. Verified end-to-end on 2 m and 70 cm with
multiple RTL-SDR dongles. Cumulative since v0.99.8 adds the
bridge-core waterfall span fix (display labels now match the
real IQ rate).

A Win7 portable zip will follow when a tester asks; the v0.99.7
Win7 build remains in this repo's git history for now.

---

### v0.99.8 — Multi-instance support (multi-band ops)

- Run **two (or more) bridges side-by-side** for multi-band setups —
  e.g. two RTL-SDRs, one feeding a 2 m WSJT-X+QMAP pair and one
  feeding a 70 cm pair. Each bridge instance gets its own INI file,
  its own dongle, its own VB-Cable line, its own WSJT-X UDP port,
  and its own Linrad TCP / UDP ports — no shared state.
- New `--instance <name>` CLI flag namespaces the INI under
  `%APPDATA%\Roaming\n6nu\RTL-SDR RX Bridge - <name>.ini`.
- New **Settings → "Linrad TCP port"** / **"Linrad UDP port"** rows
  (defaults 49812 / 50004). Increment per bridge / QMAP pair.
- New `--device-index <n>` flag + `rtlsdr/device_index` INI key for
  picking which dongle this instance opens.
- Window title now shows the instance name so two side-by-side
  bridges are easy to tell apart in alt-tab and on the taskbar.
- See RELEASE_NOTES.md for the full step-by-step multi-instance
  workflow.

The Windows 7 portable zip will follow in a separate update.

What's new in v0.99.7 — installer bug-fix:

- The optional **"Install RTL-SDR USB driver (WinUSB via Zadig)"**
  task in the v0.99.6 (and earlier) installer never actually
  launched Zadig — the bundled `zadig.exe` was in the installer
  but a stray `dontcopy` flag prevented it from being extracted
  to `{tmp}` during install, and the `[Run]` entry that was
  supposed to launch it failed silently. Symptom: fresh installs
  without a pre-existing WinUSB binding came up with **"RTL not
  found"** at bridge launch. v0.99.7 removes the `dontcopy` flag
  so the install-time Zadig step actually runs. **No code
  changes** — if you already manually ran Zadig on v0.99.6 and
  have a working binding, you can skip this update.

What's new in v0.99.6 — multi-instance / multi-band feature:

- **Configurable WSJT-X UDP listener port** for multi-band ops. New
  Settings → "WSJT-X UDP port" spin box (1024–65535, default 2237).
  Run a second WSJT-X instance on port 2238 (3rd on 2239, …) and
  point a second bridge at it — each bridge feeds its own QMAP
  instance. Persisted to INI (`wsjtx/udp_port`); the bridge
  re-binds the socket immediately on Apply, no app restart needed.
  CLI flag `--wsjtx-port` honors the INI default for fresh launches.

What's new in v0.99.5 — bug-fix release:

- **Fix "fuzzy" WSJT-X RX audio on Windows 7 / Qt5 builds.** The
  audio path was filling only one channel of the stereo VB-Cable
  buffer when the device negotiated int16 stereo (which is what
  VB-Cable on Win7 / Qt5 prefers). The other channel was
  uninitialised memory — that's the hash overlay testers heard.
  Win11 / Qt6 was unaffected (device prefers float stereo, hits
  a different code path that already handled stereo correctly).
  QMAP UDP wideband path was clean on both.

What's new in v0.99.4 — bug-fix release:

- New **Settings → "Reset frequency settings to defaults…"** button.
  Use it once if v0.99.3 felt "stuck" at a wrong frequency. It clears
  the manual SDR-freq override, the saved manual freq, and the
  transverter offset. Radio-specific settings (gain, AGC, bias-T,
  antenna, direct sampling, PPM) are NOT touched.
- **`Settings → Apply` no longer accumulates a stale "manual SDR
  frequency" value when the override checkbox is off.** Earlier
  versions wrote whatever was in the spin box on every Apply, which
  is why the field kept showing a value even when override was
  clearly unchecked.

What's new in v0.99.2 — feature parity with the SDRplay sibling:

- **Transverter offset** for IF-transverter / Ham-It-Up upconverter
  setups. Settings → "Transverter offset" field (signed MHz). The
  RTL-SDR tunes to *(WSJT-X dial + offset)* while the GUI, WSJT-X,
  and QMAP all keep showing the operating dial.
  CLI: `--transverter-offset <MHz>`.
- **Manual SDR frequency override.** Settings checkbox + freq field;
  decouples the bridge from the WSJT-X dial. Useful for QMAP-priority
  observation when activity spans more than 90 kHz around the dial.
  CLI: `--manual-freq <MHz>`.
- **Periodic streaming-stats log line** every 5 seconds.
- **Frequency display sourced from the bridge's actual operating
  freq** — populates correctly at startup before WSJT-X broadcasts.
- **High-contrast IF readout** under the dial display when transverter
  offset is non-zero.
- **Phase 1b refactor**: GUI classes now shared with the HackRF and
  SDRplay sibling apps via `bridge-core/`.

Full per-version notes are in [`RELEASE_NOTES.md`](RELEASE_NOTES.md).

### Known issue in this build

This is a first-cut beta and has **one known issue worth flagging
before you start**:

- **Residual I/Q image on the QMAP wideband path.** A real signal at
  offset +X kHz from the dial appears as a faint mirror image at −X
  kHz, suppressed by only ~10 dB. (For comparison, the HackRF RX
  Bridge sibling gets >40 dB rejection on the same code path.) QMAP
  may decode the same Q65 message twice — once at the real bin and
  once at the mirror bin. The image tracks the dial frequency, so
  it's a baseband artifact in the RTL-SDR path; toggling the
  Settings → "I/Q balance correction" checkbox does not help.
  Under investigation. Does **not** affect dial-following, WSJT-X
  RX audio, or single-bin Q65 decoding — please report any other
  issues you hit regardless.

### First-launch SmartScreen warning

The installer is **not code-signed** and is **64-bit only**
(Windows 10 / 11 x64). On first launch you will see:

> Windows protected your PC.
> Microsoft Defender SmartScreen prevented an unrecognized app from
> starting.

Click **More info → Run anyway**. You should only see this once
per binary. The same warning may appear once on the installed
`rtlsdr-rx-bridge.exe`; handle it the same way.

### What you'll need

- **Real radio + WSJT-X** — the rig is whatever you already have
  (IC-905, IC-705, FT-991A, etc.). WSJT-X drives it via Hamlib as
  always; this bridge does NOT replace that.
- **RTL-SDR USB driver (WinUSB)** — the installer offers to launch
  Zadig on completion to set this up. Pick **Bulk-In, Interface
  (Interface 0)** for VID `0BDA` PID `2832`/`2838`/etc., target
  driver = **WinUSB**, click **Replace Driver**. Skip if `rtl_test`
  on your machine already prints tuner / sample-rate without an
  error.
- **VB-Audio Virtual Cable** — <https://vb-audio.com/Cable/>.
  Provides the `Line 1` virtual sound device the bridge feeds.
- **WSJT-X 2.7+** with the UDP server enabled —
  Settings → Reporting → "Accept UDP requests" → port `2237`.
  Without this WSJT-X doesn't broadcast its dial freq and the
  bridge has nothing to track.
- **QMAP 0.6+** — Network input enabled, UDP port `50004`.

### WSJT-X audio routing for this bridge

| Setting | Value |
|---|---|
| Radio | your real rig, via Hamlib |
| PTT method | CAT |
| Sound output (TX) | the real rig's USB audio interface |
| **Sound input (RX)** | **`Line 1 (Virtual Audio Cable)`** ← fed by this bridge |
| Settings → Reporting → Accept UDP requests | **enabled**, port 2237 |

Launch order: **real rig → WSJT-X → RTL-SDR RX Bridge → QMAP**.

### First-time configuration in the bridge

After install, launch the bridge, click **Settings…**, and:

1. Set **RX audio output** to `Line 1 (Virtual Audio Cable)` —
   the default is the system audio device, not Line 1, so you
   need to pick it explicitly the first time.
2. Set **Tuner gain** for your operating conditions. The spinbox
   snaps to the librtlsdr-supported steps for your tuner (R820T:
   29 steps from −1 dB to +49.6 dB). A reasonable starting point
   on 2 m is 28–32 dB. If you have a clean band and want to dig
   for weak signals, try the higher end; if you're near broadcast
   FM or strong pagers, lower the gain or enable tuner auto-gain.
3. Click **Apply**. Settings persist to
   `%APPDATA%\Roaming\n6nu\RTL-SDR RX Bridge.ini` so subsequent
   launches come up the same way.

---

## What an RTL-SDR can and can't do here

The RTL-SDR is an 8-bit, 2.4 Msps-max, R820T/T2-front-end SDR
designed for casual scanning. Useful properties for this bridge:

- **Frequency range**: 24 MHz – 1.7 GHz native, plus HF via the
  direct-sampling switch (RTL-SDR.com V3+) — covers 6 m, 2 m, 70 cm
  meaningfully; 23 cm only marginally (sensitivity drops above
  ~1.5 GHz).
- **2.048 Msps default** — easily wide enough for the 96 kHz QMAP
  wideband stream.
- **Bias tee** (RTL-SDR.com V3+) — drives an external LNA or
  transverter sequencer at 4.5 V on the SMA centre conductor.

Limitations relative to the HackRF One:

- **Worse noise figure and IMD3** than HackRF in absolute terms.
- **2.4 Msps max** vs HackRF's 20 Msps — fine for 96 kHz QMAP, not
  enough for any future wideband-search modes.
- **No TX path** — fundamental, the RTL-SDR is RX-only silicon.

For weak-signal work on 6 m / 2 m / 70 cm with a decent external
LNA, an RTL-SDR.com V3+ on a clean band is genuinely usable. On
23 cm or in a strong-RF environment, a HackRF RX bridge will
outperform it.

---

## Reporting

Send observations / decodes / bug reports directly to
**<n6nu@arrl.net>**. Useful information to include:

- RTL-SDR vendor / tuner type (`rtl_test` output)
- Windows version
- WSJT-X version + which real rig you're using
- The bridge log: relaunch with `rtlsdr-rx-bridge.exe --console`,
  reproduce, copy/paste the console output
- For QMAP issues, also `qmap.ini` and a wideband-waterfall
  screenshot (especially if you can show the I/Q image issue
  improving or worsening with different gain / source signals —
  any data point helps the investigation)

---

## License

Copyright (C) 2026 Andreas Junge, N6NU &lt;<n6nu@arrl.net>&gt;.
Licensed under the **GNU General Public License version 3 or later**;
see [`LICENSE`](LICENSE).

This program is distributed in the hope that it will be useful, but
**WITHOUT ANY WARRANTY**; without even the implied warranty of
**MERCHANTABILITY** or **FITNESS FOR A PARTICULAR PURPOSE**. Use it
at your own risk.

Bundled third-party components — including librtlsdr (GPLv2),
FFTW3 (GPLv2+), Qt 6 (LGPLv3), SoXR (LGPLv2.1), libusb (LGPLv2.1+),
FFmpeg shared libraries (LGPLv2.1+), and Zadig (GPLv3, by Pete
Batard / libwdi) — are documented in
[`THIRD_PARTY_LICENSES.md`](THIRD_PARTY_LICENSES.md). Source code
for the bridge itself is available on request from N6NU under the
GPLv3 "written offer" provision (§6) at the email address above; a
public source-code repository will be linked here once the project
leaves beta.

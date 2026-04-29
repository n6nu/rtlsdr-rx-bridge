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

## Latest beta — v0.99.0

Download: **[rtlsdr-rx-bridge-0.99.0-setup.exe](rtlsdr-rx-bridge-0.99.0-setup.exe)**

Full per-version notes, system requirements and known limitations
are in [`RELEASE_NOTES.md`](RELEASE_NOTES.md).

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

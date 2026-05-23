# IPU6 webcam on Arch Linux — investigation notes (2026-05-22)

Notes from debugging a Dell laptop (Alder Lake IPU6EP, OV01A10 sensor) whose
webcam stopped working after `pacman -Syu`. Captures the full state of
the IPU6 stack on Arch in May 2026, what works, what doesn't, and why.

## Hardware

```
0000:00:05.0 Multimedia controller [0480]: Intel Corporation Alder Lake
  Imaging Signal Processor [8086:465d] (rev 02)
  Subsystem: Dell Device [1028:0af3]
Sensor: ov01a10 17-0036  (1 MP OmniVision, MIPI CSI-2)
```

## Initial state

- Running kernel: `linux-lts 6.6.31-1-lts` (May 2024) — predates this repo's
  documented minimum (`6.6.55+`), yet it worked.
- Mainline kernel installed but not booted: `linux 7.0.9.arch1-1`.
- This repo's custom stack was installed:
  `intel-ipu6-dkms-git-fix`, `intel-ipu6ep-camera-bin-fix`,
  `intel-ipu6ep-camera-hal-git-fix`, `icamerasrc-git-fix`,
  `v4l2-relayd r42.6fd6b6a-1`, `v4l2loopback-dkms`.

## What broke (proximate cause)

`/var/log/pacman.log` showed two changes from the 2026-05-18/21 `-Syu`:

1. **2026-05-18** — `linux: 6.19.10 → 7.0.8 → 7.0.9`. DKMS rebuild of
   `ipu6-drivers/r165.cfb7af1e5` against `7.0.9-arch1-1` **failed**
   (`==> WARNING: dkms install ... exited 10` — see
   `/var/lib/dkms/ipu6-drivers/r165.cfb7af1e5/build/make.log`,
   compile error on `drivers/media/pci/intel/ipu6/../ipu-trace.o`).
   Did not affect the running LTS kernel, but flagged that the
   out-of-tree driver is end-of-life vs current kernel API.
2. **2026-05-21** — `v4l2-relayd r42.6fd6b6a-1 → 0.2.0-1`. The
   `pacman -Syu` batch ran with a long `--ignore` list, but
   `v4l2-relayd` wasn't on it, so the AUR/upstream version replaced
   the repo's custom build. The 0.2.0 version doesn't carry this repo's
   wayland-fix patch (commit `d1c471c`), and the relay started
   exiting immediately — apps saw an empty `/dev/video0` loopback.

## Immediate fix (back to working on LTS 6.6.31)

Restored repo's custom packages by re-running `./install.sh`:

```fish
yay -R icamerasrc-git              # remove conflicting AUR version
cd ~/src/archlinux-ipu6-webcam
./install.sh                       # rebuilds + downgrades to repo's r42
```

To prevent the silent swap from recurring, the custom packages should be
added to `IgnorePkg` in `/etc/pacman.conf`:

```
IgnorePkg = v4l2-relayd icamerasrc-git-fix intel-ipu6-dkms-git-fix \
            intel-ipu6ep-camera-bin-fix intel-ipu6ep-camera-hal-git-fix
```

## Bigger picture: this repo is end-of-life for kernel ≥ 6.10

- `README.md` of this repo says supported kernels are LTS `6.6.55+`.
- IPU6 ISYS driver upstreamed in kernel `6.10` ([Phoronix](https://www.phoronix.com/news/Intel-IPU6-Media-In-Linux-6.10)).
- `intel/ipu6-drivers` out-of-tree repo doesn't build on kernel `6.17+`:
  - [intel/ipu6-drivers#423](https://github.com/intel/ipu6-drivers/issues/423) — Ubuntu 24.04, kernel 6.17, broken.
  - [Launchpad #2131913](https://bugs.launchpad.net/bugs/2131913) — `intel-ipu6-dkms fails to build on 6.17.0-6-generic`.
  - This repo's own [#95](https://github.com/stefanpartheym/archlinux-ipu6-webcam/issues/95) (6.12 LTS regression), [#89](https://github.com/stefanpartheym/archlinux-ipu6-webcam/issues/89) (6.10).
- Linux 7.0 ([Phoronix](https://www.phoronix.com/news/Linux-7.0-Released)) was released 2026-04-12. Linus bumped 6.19 → 7.0 purely
  because he rolls over at x.19; no major API break vs 6.18/6.19. So
  anything broken on 6.17/18/19 stays broken on 7.0. Linux 7.0 is **not**
  LTS — current LTS is **6.18** (supported until Dec 2028).

Conclusion: **this repo's stack only continues to work on `linux-lts 6.6.x`**.
Newer kernels need a different approach.

## Stack alternatives on kernel 7.0+ (Alder Lake IPU6EP + OV01A10)

### A. Upstream `libcamera` Simple pipeline + SoftISP

Tried this. State of play on 2026-05-22:

| Layer        | Status                                            |
|--------------|---------------------------------------------------|
| Kernel       | `intel-ipu6`, `intel-ipu6-isys`, `ivsc-csi`, `ov01a10`, LJCA bridge modules — all upstream and loaded. ✓ |
| Firmware     | `linux-firmware-intel` ships `ipu6epadln_fw.bin`. ✓ |
| Userspace    | `libcamera 0.7.1` + `libcamera-ipa` + `pipewire-libcamera` + `gst-plugin-libcamera`. ✓ |
| Sensor tuning| **`ov01a10.yaml` not shipped in libcamera 0.7.1** — falls back to `uncalibrated.yaml`. ✗ |
| AGC          | **Bang-bang controller** with fixed 10% step → **brightness flicker**. ✗ |
| Hardware ISP | **PSYS not used at all** — all processing on CPU. Worse quality. ✗ |

Workarounds applied:
- Disabled `Agc:` in `/usr/share/libcamera/ipa/simple/uncalibrated.yaml`
  to stop flicker. Backed up the original; added `NoUpgrade = usr/share/libcamera/ipa/simple/uncalibrated.yaml` to `/etc/pacman.conf`.
- Manual `analogue_gain` / `exposure` via udev rule:
  ```
  /etc/udev/rules.d/99-ov01a10-defaults.rules
  ACTION=="add", SUBSYSTEM=="video4linux", ATTR{name}=="ov01a10*", \
    RUN+="/usr/bin/v4l2-ctl -d $devnode --set-ctrl=analogue_gain=2048 \
                                         --set-ctrl=exposure=1500"
  ```
- Removed the no-longer-useful shim packages: `v4l2-relayd`,
  `v4l2-relayd-debug`, `v4l2loopback-dkms`.

Result: works in Firefox (`media.webrtc.camera.allow-pipewire=true`) and
Chromium (`chrome://flags` → `WebRtcPipeWireCamera` enabled), but image is
visibly worse than the old stack — washed-out colors, manually pinned
exposure, no auto-adaptation to lighting.

Tracking:
- [libcamera-devel: PATCH v4 proportional AGC](https://lists.libcamera.org/pipermail/libcamera-devel/2026-April/058416.html) — will fix flicker; not yet in a release.
- [intel/ipu6-camera-hal#161](https://github.com/intel/ipu6-camera-hal/issues/161) — OV01A10 needs proper tuning yaml + sensor helper.
- [Javier Tia's migration guide](https://jetm.github.io/blog/posts/ipu6-webcam-libcamera-on-linux/) — most comprehensive reference for this stack.

### B. `libcamera-ipu6` AUR — wraps `libcamhal` via libcamera pipeline handler

`https://aur.archlinux.org/packages/libcamera-ipu6` — fork of libcamera
from `kervel/libcamera@ipu6-pipeline-handler` packaged for Arch by
"Stick" (2026-04-30). Cloned to `~/src/libcamera-ipu6/`.

What it does:

```
sensor → ISYS+PSYS → libcamhal (Intel proprietary) →
   → libcamera ipu6 pipeline handler (kervel fork) → PipeWire → app
```

Restores the hardware ISP path **and** keeps native libcamera integration
(no `v4l2-relayd`/`v4l2loopback`). Ships an `ov01a10.yaml` and a sensor
helper patch for OV01A10 — explicit support for our exact sensor.
Tested by the maintainer on a Dell Latitude 5470 with the same chip
family.

Runtime dependencies (from PKGBUILD):
- `intel-ipu6-camera-hal-git` (AUR) — proprietary `libcamhal`.
- `intel-ipu6-camera-bin` (AUR) — Intel `.aiqb` tuning blobs.
- **`intel-ipu6-dkms-git` (AUR) — out-of-tree PSYS kernel module.**

**The catch on kernel 7.0.9**: the same DKMS module that already failed
to build for `linux 7.0.9` is required at runtime. Without
`/dev/ipu-psys0`, `libcamhal` can't initialize and the IPU6 pipeline
handler degrades or fails. So `libcamera-ipu6` works only on a kernel
where `intel-ipu6-dkms-git` builds — i.e., `linux-lts 6.6.x`.

### C. Stay on this repo's stack with `linux-lts 6.6.31`

Status quo. Works today, fragile against future `pacman -Syu`
unless `IgnorePkg` is set. Uses `icamerasrc + v4l2-relayd + v4l2loopback`
shim — older architecture but battle-tested for this hardware.

## Decision matrix

| Setup                                   | Kernel       | Image quality | Maintenance | Notes |
|-----------------------------------------|--------------|---------------|-------------|-------|
| This repo (status quo on LTS)           | `6.6.31-lts` | Excellent (HW ISP + Intel tuning) | High (`IgnorePkg`, stuck on old LTS) | Currently active |
| Upstream libcamera Simple on mainline   | `7.0.9`      | Poor (SoftISP, no tuning, AGC off → dark) | Low (everything in repos) | Currently active on `linux` kernel |
| `libcamera-ipu6` AUR on LTS             | `6.6.31-lts` | Excellent (HW ISP) + modern libcamera/PipeWire integration | Medium | **Recommended** if image quality matters |
| `libcamera-ipu6` AUR on mainline 7.0.9  | `7.0.9`      | N/A — DKMS doesn't build | N/A | Blocked until intel/ipu6-drivers gets 7.0 patches |

## Current configuration (as of end of session, 2026-05-22)

Boots: dual kernel (LTS 6.6.31 + mainline 7.0.9), each able to use the
camera with their respective userspace stacks:

- **On LTS 6.6.31**: this repo's custom packages all installed and working.
  `IgnorePkg` not yet set in `pacman.conf` (TODO).
- **On mainline 7.0.9**: upstream `libcamera 0.7.1` + `libcamera-ipa` +
  `gst-plugin-libcamera` + `pipewire-libcamera`. AGC disabled in
  `uncalibrated.yaml` (with `NoUpgrade` in `pacman.conf`). Manual
  exposure via udev rule (`/etc/udev/rules.d/99-ov01a10-defaults.rules`).
  Shim packages removed.

## Useful commands (reference)

```fish
# Identify hardware
lspci -nnk | grep -A3 -i "imag\|camera"
ls /sys/class/video4linux/v4l-subdev*/name | xargs -I{} sh -c 'echo {}; cat {}'

# Kernel module / firmware presence
lsmod | grep -iE "ipu6|ivsc|ljca|ov01a10|hi556|ov2740"
ls /usr/lib/firmware/intel/ipu/                 # ipu6*_fw.bin*
find /usr/lib/modules/$(uname -r)/kernel/drivers/media/pci/intel -type f

# libcamera-side discovery
cam -l
cam -c 1 -C5                                     # capture 5 frames sanity
qcam                                             # live preview window

# v4l2 controls (sensor subdev)
v4l2-ctl --list-devices
v4l2-ctl -d /dev/v4l-subdev5 --list-ctrls
v4l2-ctl -d /dev/v4l-subdev5 --get-ctrl=analogue_gain,exposure
v4l2-ctl -d /dev/v4l-subdev5 --set-ctrl=analogue_gain=2048 --set-ctrl=exposure=1500

# PipeWire-side check
wpctl status | grep -iE "camera|video"
gst-device-monitor-1.0 Video/Source

# GStreamer test capture (encoded JPEG, unlike cam --file which is raw)
gst-launch-1.0 libcamerasrc num-buffers=1 \
  ! video/x-raw,width=1280,height=800 ! videoconvert ! jpegenc \
  ! filesink location=/tmp/cam.jpg

# DKMS state
sudo dkms status
cat /var/lib/dkms/ipu6-drivers/*/build/make.log | tail -40

# udev reload after editing rules
sudo udevadm control --reload
sudo udevadm trigger --subsystem-match=video4linux

# PipeWire reload after libcamera config change
systemctl --user restart wireplumber pipewire pipewire-pulse
```

## Browser flags

- **Firefox**: `about:config` → `media.webrtc.camera.allow-pipewire = true`.
- **Chromium/Chrome/Brave**: `chrome://flags` → `WebRtcPipeWireCamera` → Enabled.
- **Electron apps (Signal, Slack, Discord, etc.)**: launch with
  `--enable-features=WebRtcPipeWireCamera`.

## Patching the DKMS module for kernel 7.0 (this branch's work)

Update 2026-05-23: rather than waiting for upstream `intel/ipu6-drivers`
to ship a 7.0 patchset (still open, see PRs #424 and #425 — neither
addresses 7.0 specifically), the DKMS module was patched locally in this
repo. The patch lives at
[intel-ipu6-dkms-git/0001-kernel-7-build-fixes.patch](intel-ipu6-dkms-git/0001-kernel-7-build-fixes.patch).
PKGBUILD bumped from pinned commit `cfb7af1e` → `ca28a0278` (tip of
`iotg_ipu6` as of 2026-05-22, includes PR #430 "IPU6 release for iot
kernel v6.18").

### Build errors encountered, in order, and the fix for each

| # | Error                                                                              | Root cause                                                                                                                  | Fix                                                                                                 |
|---|------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| 1 | `MODULE_IMPORT_NS(INTEL_IPU_BRIDGE)` / `EXPORT_SYMBOL_NS_GPL(..., INTEL_IPU6)` fail to compile (28 sites) | Kernel 6.13 [`cdd30ebb1b9f`](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2579808.html) — namespace must now be a string literal | New `include/ipu-namespace-compat.h` shim that `__stringify()`s the namespace on ≥6.13; force-included via `subdir-ccflags-y` |
| 2 | `asc->match.src_pad` — `struct v4l2_async_match_desc` has no `src_pad`            | `src_pad` is an out-of-tree-only field added by `patch/v6.18.3/0008-media-lt6911-2-pads-linked-to-ipu-2-ports-for-split-mode.patch` | Pass `-1` (matches `patch/v6.18.3/0023-...align-params-for-non-MIPI-split...`); function discovers source pad itself |
| 3 | `v4l2_get_link_freq(ext_sd->ctrl_handler, …)` — incompatible pointer              | Kernel 6.15 dropped the ctrl_handler overload (`_Generic`), 7.0 removed it entirely; only `struct media_pad *` accepted     | Use `src_pad` (already in scope) on ≥6.15                                                           |
| 4 | `struct v4l2_subdev_stream_config` undefined                                      | Kernel 6.18 made the struct opaque; access via `state->routing.routes[]` now                                                | Walk `state->routing.routes[i]` and match `route->sink_pad == r_pad->index`                         |
| 5 | `struct vb2_ops` has no `wait_prepare`/`wait_finish`                              | Kernel 7.0 removed these callbacks; core handles queue locking via `q->lock`                                                | `#if LINUX_VERSION_CODE < KERNEL_VERSION(7, 0, 0)` around the two assignments                       |
| 6 | `field 'clkdev_data' is of incomplete type`                                       | `<linux/clkdev.h>` was included only under `#if IS_ENABLED(CONFIG_INTEL_IPU_ACPI)`, but the struct using `clk_lookup` isn't guarded | Include `<linux/clkdev.h>` unconditionally                                                          |
| 7 | `ipu6_isys_init`: trailing comma syntax error when `CONFIG_INTEL_IPU_ACPI` off    | Function signature/call site put a comma OUTSIDE the conditional `spdata` arg                                               | Move the comma INSIDE the `#if IS_ENABLED(CONFIG_INTEL_IPU_ACPI)` block on both signature and call  |
| 8 | `ipu_get_acpi_devices` undefined at modpost                                       | We disabled the `ipu-acpi` modules but the call site in `ipu6.c` is gated by `CONFIG_INTEL_IPU_ACPI` (and the macro was still defined) | Stop exporting `CONFIG_INTEL_IPU_ACPI` / `CONFIG_INTEL_IPU6_ACPI` in the Makefile                   |
| 9 | `isx031.c:1059: invalid array subscript` on `platform_data->suffix[0]`            | Pre-existing code bug: struct has `char suffix;` but driver formats it with `%s` and subscripts with `[0]`                  | Disable the industrial/automotive sensor drivers (ISX031, MAX9X, AR0234, LT6911UX{C,E}) for DKMS — they don't apply to consumer laptops, the `.c.non_upstream` files require manual rename anyway |

### What gets built now

Three core modules only — exactly what `libcamera-ipu6` / `libcamhal` needs:
- `intel-ipu6.ko` (replaces the kernel's in-tree one)
- `intel-ipu6-isys.ko` (replaces the kernel's in-tree one)
- `intel-ipu6-psys.ko` ⭐ — **the module with no upstream equivalent**, required by libcamhal to open `/dev/ipu-psys0`.

### Confirmed working on 2026-05-23

```
$ sudo dkms status
ipu6-drivers/r193.ca28a0278, 7.0.9-arch1-1, x86_64: installed (Original modules exist)

$ ls /lib/modules/7.0.9-arch1-1/updates/dkms/ | grep ipu6
intel-ipu6-isys.ko.zst
intel-ipu6.ko.zst
intel-ipu6-psys.ko.zst
```

The `(Original modules exist)` note means DKMS detected the kernel's own
in-tree `intel-ipu6{,-isys}` and our `/updates/dkms/` versions take
precedence at modprobe time (standard DKMS behavior).

### What this unblocks

The `libcamera-ipu6` AUR package (cloned at `~/src/libcamera-ipu6`) was
previously blocked on kernel 7.0 because it requires `/dev/ipu-psys0`
(provided only by this DKMS module). With the patched DKMS now building,
the proper migration path on `linux 7.0.9` is:

```fish
# remove the upstream Simple-pipeline libcamera packages (they conflict
# with libcamera-ipu6, which provides=libcamera)
yay -R gst-plugin-libcamera libcamera-ipa libcamera

# install Intel's userspace stack + the libcamera fork that wraps it
yay -S intel-ipu6-camera-hal-git intel-ipu6-camera-bin
cd ~/src/libcamera-ipu6 && makepkg -si

# undo the dark-image workarounds — libcamhal does proper AGC via PSYS
sudo rm -f /etc/udev/rules.d/99-ov01a10-defaults.rules
sudo cp /usr/share/libcamera/ipa/simple/uncalibrated.yaml.bak \
        /usr/share/libcamera/ipa/simple/uncalibrated.yaml 2>/dev/null

# reboot, then verify pipeline handler is now "ipu6" (not "simple")
cam -l
```

This should restore the hardware-ISP image quality (the thing the
upstream Simple pipeline can't match) while keeping you on the new
kernel. The four "fixes you won't find documented elsewhere" from
[libcamera-ipu6's README](file:///home/user/src/libcamera-ipu6/README.md)
handle the GDM/wireplumber/seccomp/libcamhal race conditions.

### gcc 16 also breaks `intel-ipu6-camera-hal-git`

On the same migration day, building Intel's userspace HAL (libcamhal)
failed against gcc 16 because two legacy warnings were promoted to
errors:

```
src/iutils/CameraDump.cpp:551
  int bytes_read = 0;      // set but never read
modules/ia_css/ipu6/include/ia_css_psys_terminal_impl.h:1862
  unsigned mem_offset;     // set but never read
```

Both are pre-existing dead-variable warnings that older gcc tolerated.
Fixed in this repo's [intel-ipu6ep-camera-hal-git/PKGBUILD](intel-ipu6ep-camera-hal-git/PKGBUILD)
by exporting `-Wno-error=unused-but-set-variable` (+ a few related
flags) in `build()`. The PKGBUILD now also `provides=intel-ipu6-camera-hal-git`
so it is a drop-in for `libcamera-ipu6`'s dependency.

Same naming-provides change applied to
[intel-ipu6ep-camera-bin/PKGBUILD](intel-ipu6ep-camera-bin/PKGBUILD) —
this Alder-Lake-only variant now also `provides=intel-ipu6-camera-bin`,
so the same code path covers users on this repo's `-fix` variants and
users coming from `libcamera-ipu6 → intel-ipu6-camera-{hal,bin}` AUR
deps.

### Caveats

- DKMS build still **fails on `linux-lts 6.6.31`** (the patches assume
  newer-kernel APIs in places). If you want LTS to work too, reinstall
  the previous `intel-ipu6-dkms-git-fix r165.cfb7af1e5` while on LTS, or
  bound the patches with version guards.
- The Makefile no longer builds the industrial/automotive sensors. If
  you have ISX031/MAX9X/LT6911 hardware, revert the commented-out
  `obj-y` lines in `intel-ipu6-dkms-git/0001-kernel-7-build-fixes.patch`
  and fix `isx031.c`'s `suffix[0]`/`%s` mismatch yourself.

## Recommended next steps

1. **Short term**: install `libcamera-ipu6` from AUR now that the PSYS
   module is available on 7.0. See the migration block above.
2. **Medium term**: open a PR upstream to `intel/ipu6-drivers` based on
   the local patch, so other distros benefit (the bulk of it is generic
   kernel-API churn, not laptop-specific).
3. **Long term (passive)**: track libcamera 0.8 (proportional AGC, per-
   sensor tuning files for OV01A10). When those land, the upstream
   Simple-pipeline path may be good enough to drop libcamhal entirely.

## Sources

- [Linux 7.0 release — Phoronix](https://www.phoronix.com/news/Linux-7.0-Released)
- [Linux 7.0 — The Register](https://www.theregister.com/2026/04/13/linux_kernel_7_releaseed/)
- [Intel IPU6 driver — kernel.org](https://docs.kernel.org/driver-api/media/drivers/ipu6.html)
- [Intel IPU6 Webcam on Linux: From Proprietary Stack to Mainline — Javier Tia](https://jetm.github.io/blog/posts/ipu6-webcam-libcamera-on-linux/)
- [How to use the IPU6 webcam with kernel 6.10+? — Arch Linux Forums](https://bbs.archlinux.org/viewtopic.php?id=297262)
- [libcamera-ipu6 AUR package](https://aur.archlinux.org/packages/libcamera-ipu6)
- [kervel/libcamera — IPU6 pipeline handler fork](https://github.com/kervel/libcamera)
- [intel/ipu6-drivers issues](https://github.com/intel/ipu6-drivers/issues)
- [intel/ipu6-camera-hal#161 — OV01A10 missing tuning](https://github.com/intel/ipu6-camera-hal/issues/161)
- [libcamera Sensor Tuning Guide](https://libcamera.stefanklug.com/docs/tuning-guide/tuning.html)
- [libcamera-devel: PATCH v4 proportional AGC](https://lists.libcamera.org/pipermail/libcamera-devel/2026-April/058416.html)
- [stefanpartheym/archlinux-ipu6-webcam README](./README.md)

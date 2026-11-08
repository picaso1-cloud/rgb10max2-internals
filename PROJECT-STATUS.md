# PROJECT STATUS — Handhelds (compaction checkpoint)

**Date:** 2026-11-08 · **Purpose:** complete cross-project state so context can
be compacted and work resumed from this file alone. Mirrored in both
`rgb10max2-internals` and `rg353v-internals` repos.

---

## Devices & access

| Device | SoC | OS | Access |
|---|---|---|---|
| **Powkiddy RGB10 Max 2** | RK3326, 1 GB | andr36oid (Android 11) on SD | `adb connect 192.168.0.99:5555` (IP varies; also seen 192.168.31.190); root daemon via `/data/local/tmp/cmd.in`; **currently: new SD card pending** |
| **Anbernic RG353V** | RK3566, 2 GB | **GammaOS_Lite_v1.5** (Android 12, eMMC) + **ArkOS** (SD/TF1) | Android side: wireless adb `192.168.1.148:5555` (persisted via Magisk). ArkOS side: `ssh ark@192.168.1.148`, password `ark` (sudo same). Both currently offline/other network |

Host: `/home/souhail/Andr36oid` workspace. Devices are on `192.168.x` networks
that change between sessions — re-scan if connect fails.

---

## Project 1 — RGB10 Max 2: andr36oid port  ✅ feature-complete (rebuild pending)

**Production state:** v31 DTB + v32 service, validated on hardware. The test SD
card later **died** (unstable reads; BOOT FAT corrupted) — hence the rebuild
kit. Console currently has no working card.

### Done (all hardware-verified)
- **Wi-Fi from battery every cold boot**: rk817 PMIC-side latch via raw i2c
  `0xB4=0x47` (`rk817tool`). Latch survives warm reboots, resets on cold off.
- **Charging + Wi-Fi simultaneously** (stock parity): stock topology — OTG phy
  port **disabled**, dwc2 phyless with `phys` kept (defer timing), `dc_det`
  intact. Verified `+869 mA` charging with `wlan0` up.
- **Boot black screen fixed** (v27): auto sleep/wake cycle after bootanim
  (SF/HWC never enables the CRTC; DPMS cycle fixes it). Screen comes up alone.
- **L3/R3 fix** (v29): stick pushes emit BTN_THUMBL/THUMBR (317/318).
- **Right-stick remap** (v30/v31): `amux-channel-mapping = <1 0 2 3>` +
  `invert-absrx` (no invert-absry) — mouse mode cursor and emulator stick
  correct (the board muxes right-X on ch0/ch1 differently than the R36S; the
  stock `abs_*_channel` props are inert in this kernel).
- **Mouse mode**: toggle = **Select+Start** (hold ~1 s), pointer = right stick;
  props `pointer_stick=right`, `invert_ry=false`, `pointer_invert_x=false`.
- **Performance (v28)**: swappiness 100, SD readahead 512, iostats off, GPU max
  600 MHz (unlocked from 520), UI animations 0.5, background janitor (am
  kill-all every 40 s + force-stop thorium/vlc), `gamemode` script
  (CPU floor 1.008 GHz). RAM available 369→442 MB.
- **Remote play (v32)**: Tailscale (CLI daemon, tailnet `the console`
  `100.x.x.x`, autostart + keep-alive in service) + Moonlight v12.1
  (preinstalled) + Sunshine on PC. Service v32 runnable on any card.
- **Docs**: `PORT-REPORT.md`, `device-work/kernel/VALIDATION.md` (v9→v32),
  `FIRMWARE-GUIDE.md`, `OS-RESEARCH.md`, repo **rgb10max2-internals**.

### Next actions (ordered)
1. **New SD card**: flash `dist/andr36oid-rgb10max2-master.img`
   (md5 `618f948d7055af948c362ff593433924`) → extend userdata to fill the card
   → restore `dist/userdata-full.tar` (apps/data/wifi) → restore
   `dist/essentials.tgz` (tailscale auth into `/data/local/tmp/ts/state/`) →
   boot → verify: Wi-Fi + charging + mouse mode + tailscale (`100.x.x.x`)
   + Moonlight hosts.
2. Gameplay test of GPU 600 MHz + `gamemode` → keep or revert.
3. Optional: kernel patch for honest battery icon while latched; deeper
   performance measurement.

### Known limits (by design)
- Cable ADB/USB device modes: phy port disabled for charging (documented trade-off).
- Battery icon may read “charging” while latched (self-sensing; % is honest).

### Artifacts (`/home/souhail/Andr36oid/dist/`)
- `andr36oid-rgb10max2-master.img` (4.7 GB, md5 618f948d…) — CLEAN rebuild from
  official base, verified (mtools).
- `andr36oid-rgb10max2-v26.img` — older working image (v26-era).
- `andr36oid-rgb10max2-clone.img` — clone of the DYING card (do not use).
- `userdata-full.tar` (1.5 GB) — full /data snapshot (apps+data+wifi).
- `essentials.tgz` (1.6 KB) — tailscale auth (`ts/state/tailscaled.state`).
- `andr36oid-20261024-1750-rk3326.img.xz` — official base release.
- Builders: `image-patcher/build-rgb10max2-image.sh` (mtools BOOT rebuild),
  `image-patcher/migrate-console-state.sh`.

---

## Project 2 — RG353V: GammaOS study + optimization  ✅ deployed

### Done
- Full study: hardware, GammaOS structure (Android 12 sargo GSI on RK3566 BSP,
  dynamic partitions, Magisk), drivers, **kernel+DTB decompiled** (`survey/boot.dts`),
  comparisons vs the RGB10 Max 2 pain list.
- **Wireless adb** root cause (adbd USB-only) + fix + **Magisk persistence**
  (`/data/adb/service.d/janitor.sh` sets `service.adb.tcp.port 5555`).
- **Optimization deployed**: `/data/local/tmp/gamemode` (CPU floor 816 MHz +
  DMC floor 780 MHz; GPU min not settable — devfreq ignores while suspended),
  `gamemode-off`, janitor (`am kill-all` every 60 s).
- PS2 on Android already works for the user (AetherSX2/ARMSX2 APK).
- Upgrade path studied: **GammaOSNext does NOT support the RG353 series**
  (deprecated GammaOS-RK3566 v1.5.1 is current); SDDiskTool recovery model +
  safest procedures documented.

### Next actions
1. (Optional) deploy Moonlight + Tailscale on GammaOS (staged in plan).
2. Continue when the ROCKNIX experiment (Project 4) gives data.

---

## Project 3 — RG353V: ArkOS side  ⏸ investigated

### Done
- Survey (Ubuntu 19.10, kernel 4.19.172, ark/ark SSH, governors, storage).
- **PS2-on-Linux investigation (complete record, `12-ps2-linux-investigation.md`)**:
  ARMSX2 Linux-arm64 AppImage fully starts after bundling glibc 2.39,
  libstdc++ 14, FreeType 2.13, libwayland 1.22 — but **ArkOS cannot display a
  Qt6 window** (weston 5 lacks `xdg_wm_base`; xwayland/weston die on VT
  signals; Xorg fbdev fails `mmap fbmem: Invalid argument`; sway can't get a
  VT from SSH). ES restored (`systemctl start emulationstation`).
- PS2 assets confirmed on the SD: `/roms2/ps2/*.iso` (GTA VCS, GTA SA, NFS UG,
  Crash TTR, SNK NeoGeo Collection, Twisted Metal) + BIOS
  `/roms2/ps2/bios/SCPH30004R.bin` (+ scph39001).

### Next actions
- Only via Project 4 (ROCKNIX). Keep ArkOS unchanged meanwhile.

---

## Project 4 — Arch-R style Linux for RK3566/RG353V  🚧 planned (next big task)

**Feasibility confirmed**: mainline has `rk3566-anbernic-rg353v.dts`; ROCKNIX
already builds the RK3566 boot chain; Arch-R's board layout is
`projects/ArchR/devices/<SoC>/`.

### Stages
0. **Validate**: flash `dist/ROCKNIX-RK3566-Specific.img.gz` (1.36 GB, 20261101)
   to a spare SD → boot → verify panel/touch/audio/Wi-Fi/GL (`xdg_wm_base`,
   Mesa) → run ARMSX2 AppImage natively → test PS2 ISOs. **Needs a spare A2 SD.**
1. Capture ROCKNIX boot chain (u-boot/DDR blobs/kernel/DTB/partitions).
2. Arch Linux ARM rootfs on the ROCKNIX boot chain (hybrid "Arch-R for RK3566").
3. Fork Arch-R, add `projects/ArchR/devices/RK3566/`, validate, propose upstream.

Full plan: `13-archr-rk3566-plan.md`.

---

## Repositories & upstream

| Repo | State |
|---|---|
| `picaso1-cloud/rgb10max2-internals` | @ `d3abf32` — RGB10 Max 2 knowledge base (10 docs + artifacts) |
| `picaso1-cloud/rg353v-internals` | @ `e095856` — RG353V knowledge base (14 docs + survey) |
| `andr36oid/andr36oid_sdflasher` PR **#2** (`picaso1-cloud:rgb10max2-port`) | open, mergeable, head **`649dee6`** — docs + DTS + service + builder |
| Local workspace | `/home/souhail/Andr36oid` — PORT-REPORT, VALIDATION, device-work, image-patcher, dist |

GitHub token: fine-grained, `picaso1-cloud` (push via explicit URL form;
never stored in files). PRs can't be created cross-repo with this token (user
opened #2 manually).

---

## Open decisions for the user

1. **RGB10 Max 2**: new card size/class → flash master + migrate state (Project 1 step 1).
2. **RG353V**: spare SD for the ROCKNIX Stage-0 test?
3. If the ArkOS SD is to be reused for ROCKNIX: back up `/roms2` (ROMs/ISOs, several GB) first?
4. Priority order: RGB10 rebuild · RG353V ROCKNIX/Arch-R · GammaOS remote-play.

---

## Quick reference

```sh
# RGB10 Max 2
adb connect <ip>:5555
adb shell 'cat > /data/local/tmp/cmd.in << "EOF"\n{ ... } > /data/local/tmp/cmd.out 2>&1\nchmod 666 /data/local/tmp/cmd.out\nEOF'
# tailscale (on device)
TS=/data/local/tmp/ts/tailscale; SK=--socket=/data/local/tmp/ts/tailscaled.sock
$TS $SK status ; $TS $SK ip -4

# RG353V GammaOS
adb -s <serial|ip:5555> shell ; su -c '/data/local/tmp/gamemode'
# RG353V ArkOS
sshpass -p ark ssh ark@<ip>   # sudo with same password
systemctl {stop,start} emulationstation
```

## Session history highlights (for context)

- RGB10 Max 2: v9→v32 iterations; the four big discoveries: rk817 latch,
  stock-topology charging (phy port), CRTC handoff black screen, right-stick
  mux remap. Test card died → clean rebuild kit + mtools BOOT pipeline.
- RG353V: GammaOS = GSI+vendor; wireless adb persistence; gamemode/janitor;
  ARMSX2-on-ArkOS blocked by display stack (full record); ROCKNIX identified as
  the modern path; Arch-R port plan written.

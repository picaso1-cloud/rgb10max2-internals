# Powkiddy RGB10 Max 2 — Internals

A knowledge base for the **Powkiddy RGB10 Max 2** handheld (Rockchip **rk3326**,
1 GB RAM, Android-capable), compiled from hands-on porting work: bringing
[andr36oid](https://github.com/andr36oid) (Android 11 / LineageOS 18.1 fork) to
full hardware parity — Wi-Fi, charging (both at once), display, input — and
documenting every non-obvious behavior of the board so nothing is lost.

Everything here was **verified on real hardware** (stock RetroOZ logs, live adb
sessions, register-level PMIC reads/writes). Where a claim is theory, it says so.

> **Cross-device context:** see `MASTER-PLAN.md` (three-device strategy),
> `SHARED-KNOWLEDGE.md` (reusable findings), `docs/DECISIONS.md`,
> `docs/RE-METHOD.md`. This repo is the RGB10 Max 2 slice of the platform.

## The one-paragraph summary

The board has **two USB-C ports** (power: charge-only, OTG: data) that share one
5 V net. The Wi-Fi dongle (RTL8723BU on an internal hub) hangs off the OTG bus
and its power comes from the rk817 PMIC's OTG rail, which behaves as a
**PMIC-side latch** that only a raw i2c write can flip. Enabling the OTG *phy*
port tells the charger driver about OTG, which makes it disable its own
plug-detect interrupts and kills charging — so the phy port must stay **disabled**
(like stock), giving charging + Wi-Fi simultaneously. The display has a
surfaceflinger/HWC quirk: the CRTC is never enabled for Android's framebuffer,
fixed by an automatic sleep/wake cycle in a boot service. Boot completes in
~48 s (~51 s to usable UI) on Android 11 with 1 GB RAM.

## Contents

| File | What's in it |
|---|---|
| [01-hardware.md](01-hardware.md) | Board identity, SoC, ports, nets, Wi-Fi/BT module, input, battery |
| [02-power-and-charging.md](02-power-and-charging.md) | The rk817 OTG latch, charger driver trap, the shared-net model, final working design |
| [03-display.md](02-display.md) | Panel, orientation, the black-screen bug, the DPMS-cycle fix |
| [04-boot-chain.md](04-boot-chain.md) | u-boot, boot.ini, DTB selection, card layout, boot timing analysis |
| [05-wifi.md](05-wifi.md) | RTL8723BU/rtl8xxxu quirks, WPA2 forcing, regdomain, power-save, real-world limits |
| [06-android-internals.md](06-android-internals.md) | init/zygote timings, the vendor service pattern, root-command daemon, boot speed levers |
| [07-methods-and-tools.md](07-methods-and-tools.md) | Card-surgery rules, adb tooling, pstore/dmesg reading, DTB compile flow |
| [08-validation-log.md](08-validation-log.md) | Condensed version history v9 → v32 with what each proved |
| [09-remote-play.md](09-remote-play.md) | Moonlight + Sunshine + Tailscale: full remote Steam setup, installed and working |
| [10-rebuild-guide.md](10-rebuild-guide.md) | SD card rebuild: card shopping, clone vs clean-rebuild paths, userdata growth, state migration |
| [11-rocknix-rk3326-port.md](11-rocknix-rk3326-port.md) | ROCKNIX shared-core port plan: Max 2 = OGS-class board, what works out of the box, DTS/quirk work, Wi-Fi/power risks |
| [service/](service/) | The production boot service (`diag.sh` v27 + init rc) |
| [tools/](tools/) | `rk817tool.c` (raw PMIC i2c) and `wifitool.c` (nl80211, no libnl) |
| [dts/](dts/) | The production Android device tree for the board |

## Status of the port (Nov 2026)

- ✅ Wi-Fi from battery on every cold boot
- ✅ **Charging and Wi-Fi simultaneously** (stock parity)
- ✅ Bluetooth, display (270°), all buttons/sticks, battery gauge
- ✅ Screen comes up by itself after boot (no button press)
- ✅ Off-state charging (PMIC autonomous)
- ⏳ Cable ADB / USB device modes (requires phy enabled — conflicts with charging, see 02)
- ⏳ Performance tuning phase (RAM/CPU/services for gaming)
- ✅ Upstream contribution: [andr36oid/andr36oid_sdflasher PR #2](https://github.com/andr36oid/andr36oid_sdflasher/pull/2)

## Related repos

- Upstream image/flasher: `andr36oid/andr36oid_sdflasher` (our docs live in PR #2)
- Kernel source used (dtsi base): `andr36oid/android_kernel_gameconsole_r36s`

# 08 — Validation log (condensed, v9 → v27)

Full history lives in the porting workspace (`device-work/kernel/VALIDATION.md`);
this is the condensed, self-contained version. Each version = a DTS + service
pair validated on hardware.

| Ver | What it proved / changed |
|---|---|
| v9–v12 | early bring-up: display pipeline, orientation, buttons/sticks, battery gauge verified against the stock RetroOZ boot log (ground truth: hub `05e3:0608` + dongle `0bda:b720` enumerate on dwc2) |
| v13 | captured the charger-driver behavior: with the phy enabled, the phy fires `EXTCON_USB_VBUS_EN`, charger sets `otg_in=1` and disables its plug IRQs — the seed of the final explanation |
| v15 | dwc2 phyless probe timing: keep `phys=<&u2phy_otg>` with the port disabled → lookup defers until UTMI clock, then `-ENODEV` → phyless probe with clock running. Removing `phys` outright = `dwc2_core_reset: HANG!`, probe -16 |
| v18/v19 | latch discovery: kernel writes never reach the OTG rail (register cache lies); raw i2c `0xB4=0x47` works; latch survives warm reboot, resets on cold power-off; `rk817tool` born |
| v20 | Wi-Fi from battery on every cold boot achieved; radio tuning (`wifitool ps-off`, `reg MA`) + 30 s watchdog |
| v21 | deleted `dc_det_gpio` to fix the false "charging" icon → **broke charging entirely** (reverted; lesson: dc_det enables the charge path) |
| v23 | first production image; BT verified; WPA2-force method for mixed networks |
| v24 | "charger-aware" service — **wrong model** (assumed separate nets); charging-while-off verified by user; false icon explained as self-sensing loop |
| v25 | charger-driver unbind for honest icon; wifi watchdog for hub-loss recovery; still wrong model |
| **v26** | **THE corrected design**: stock topology (`u2phy_otg` disabled, dwc2 phyless-with-phys, dc_det intact) + unconditional latch + watchdog. **Charging + Wi-Fi simultaneously verified live** (`Charging 3.899 V +869 mA`, `usbdevs=8`, `wlan0` up). Boot-time detail: charger present at boot feeds the hub before the latch is needed |
| v27 | boot black screen fixed: auto sleep/wake cycle after `boot_completed` (panel comes up by itself — user confirmed zero button presses) + `nobootanimation=1` → ready at 51.1 s (was 62.9 s), boot_completed ~48 s. Screen alive ~66 s (v28 will move it right after boot_completed ~52 s) |

## Open items (as of v27)

- battery icon may show "charging" while latched on battery (self-sensing; the
  % gauge is honest) — kernel-driver patch deferred to the kernel-build phase
- cable ADB / USB device modes need the phy port enabled → conflicts with
  charging (documented trade-off)
- v28: reorder service so the panel cycle runs immediately after
  `boot_completed` (~52 s instead of 66 s)
- performance phase: RAM/CPU/services baseline → trim for gaming
- CPE-C391 AP test pending (`1234567890`)

## Production artifacts

- image: `andr36oid-rgb10max2-v26.img` (4.73 GiB, md5 `206d39785f3eeb15b3d50ef571177630`)
  — v27 service installed live on the card after imaging (vendor remount)
- upstream: `andr36oid/andr36oid_sdflasher` PR #2, branch `rgb10max2-port`
  (head `08f77bc` at time of writing)

# evergo initramfs (MT6833 / Redmi Note 11 5G)

Bring-up ramdisk for the mainline Linux port on the MT6833 evergo.

## Layout

```
init.c              the init program (source, version controlled)
Makefile            builds root/init from init.c
root/               the ramdisk tree -- this is what gets packed
  init              generated, not tracked
  lib/firmware/     blobs request_firmware() serves before switch_root
  dev/ proc/ sys/   mount-point placeholders (kept via .gitkeep)
```

## Build

```
make            # aarch64-linux-gnu-gcc -static -Os -> root/init
make clean
```

The output is a statically linked aarch64 binary of roughly 650 KiB.  It is
an ordinary glibc program -- stdio, `mount(2)`, the watchdog ioctls, `mmap`
-- so the `-nostdlib` / `-ffreestanding` flags that the xaga (MT6895)
reference initramfs uses do not apply here.

`mkinitramfs.sh` in the mt6833-port tree rebuilds the tree from scratch on
every build, in this order:

1. `rm -rf root/{proc,sys,dev,lib}`, then recreate the `.gitkeep` dirs
2. `cp build/firmware/* root/lib/firmware/`
3. `make -C <this repo>`

so the binary always comes from the source here instead of from a copy
sitting in the build tree.

**`root/lib/firmware/` is a second copy.**  It is tracked in this repo, but
step 1 deletes it and step 2 refills it from `mt6833-port/build/firmware/`.
Editing a blob only here has no effect on the next build: change the one
under `build/firmware/`, otherwise the two versions silently diverge.

## What init.c does

It runs as PID 1 inside the initramfs and stays there until it finds a real
Linux rootfs.  The stages, in the order `main()` runs them:

| Stage | Behaviour |
|---|---|
| watchdog | Takes over the TOPRGU timer that `setup.c` arms for 31 s.  The fd is inherited across `switch_root`, because reopening `/dev/watchdog` afterwards fails -- that once caused a reboot loop every 33 s. |
| black box | Writes a boot marker and a periodic heartbeat to pmsg, and replays the previous boot's pstore console and pmsg ring.  A silent death becomes "last seen at t=X" after the reset, read back on the next boot. |
| console | Opens `/dev/ttyGS0` (the built-in `g_serial`) and streams the kernel log through it.  Single-key commands are read from the same port; run `console_pump()` to see the current list. |
| connectivity | `COMBO_IOCTL_CONNSYS_SOC_HW_INIT` plus `COMBO_IOCTL_DO_MODULE_INIT` on `/dev/wmtdetect`, forks `conn_launcher()` to answer WMT's ROM-patch requests, then flips the gen4m power switch through `/dev/wmtWifi`.  This is what makes `wlan0` exist at all. |
| firmware | Copies eight of the blobs below into `/rootfs/lib/firmware`.  `MS_MOVE` hides the initramfs tree from the kernel, so a later firmware reload (a WiFi toggle, a retry after an assertion) would otherwise fail with `-ENOENT`. |
| nvdata | Mounts the nvdata partition read-only and copies `APCFG/APRDEB/BT_Addr` to `/rootfs/lib/firmware/mediatek/mt6833/`.  The Bluetooth address is device-unique and must not be baked into an image that other units also flash. |
| rootfs | Probes the block devices and only hands over when `is_linux_rootfs()` matches, so a stock encrypted Android userdata stays untouched.  Then `MS_MOVE` + `chroot` + `exec /sbin/init`. |
| diagnostics | i2c bus scan and a per-bus death watch, IRQ delivery test, pinctrl and clk dumps.  All reachable from the console. |

The hand-over is deliberately the last thing that happens: the watchdog,
the console and the connectivity stack are all live before the rootfs ever
becomes `/`, which is what makes an early hang observable instead of silent.

## Firmware notes

### Touch

`lib/firmware/novatek_ts_ct_fw.bin` is the **CSOT** panel blob.  The vendor
ships two, selected by LCM name: `k16a_36_02_0a_vdo` (Tianma) needs
`novatek_ts_fw.bin`, `k16a_42_02_0b_vdo` (CSOT) needs
`novatek_ts_ct_fw.bin`.  This unit is CSOT.  Flashing the Tianma blob makes
every button of the right-hand PIN column land one row higher.

### Connectivity (MT6631)

Three groups, and a missing file keeps CONN from ever coming up:

| Files | Role |
|---|---|
| `soc2_2_ram_{mcu,bt,wifi}_1_1_hdr.bin` | the three ROM patches (CONN MCU / BT / WLAN) that `wmt_rom_patch_info()` loads |
| `WIFI_RAM_CODE_soc2_2_1_1.bin` | the WLAN runtime firmware |
| `WMT_SOC.cfg`, `BT_FW.cfg`, `wifi.cfg` | configuration the stack reads by name |

The `*_1a_1` copies are shipped but **no code path opens them**:
`wmt_rom_patch_info()` builds its filenames from a hard-coded `_1_1`.  They
are about 1.6 MB of dead weight in the ramdisk unless a variant SoC turns up
that needs the other ROM patch revision.

## Credit

Structure and the Bluetooth-address bring-up logic both refer to
https://github.com/MT6895-Mainline/initramfs.git

All the Code is Written By DeepSeek V4.1 Flash



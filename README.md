# macOS Tahoe on the Lenovo LOQ 15IAX9E (Intel iGPU acceleration)

This is my OpenCore EFI for the Lenovo LOQ 15IAX9E. It gives full Metal acceleration on the Intel iGPU under macOS Tahoe 26.7.1.
It uses Apple's Tiger Lake graphics driver through the IrisXe project and NootedGreen, with a few fixes for this laptop.
Wi-Fi works with AirportRTW. The steps are in my Wi-Fi repo:
[LOQ-15IAX9E-RTL8852BE-WiFi-macOS](https://github.com/LilyandSimba/LOQ-15IAX9E-RTL8852BE-WiFi-macOS).

Debugged and built with help from Claude Code.

> **Warning.** This is for the **Lenovo LOQ 15IAX9E** with the **i5 12450HX** and the Intel iGPU **`8086:468B`** only.
> It will not work as is on other LOQ models, other CPUs or other iGPUs. The GPU size (16 EU, 4 L3 banks), the USB map and the
> ACPI tables are made for this exact machine. Wrong GPU values hang the boot or break the picture. Keep a working USB installer
> and a backup before you try it. NootedGreen still panics now and then, see [Known issues](#known-issues).

## Hardware

| Part | What it is |
|---|---|
| Laptop | Lenovo LOQ 15IAX9E (83LK / 83ME) |
| CPU | Intel Core i5 12450HX (Alder Lake HX, 4P + 4E cores, 12 threads) |
| iGPU | Intel UHD Graphics `8086:468B`, 16 EU, shown to macOS as `9A49` |
| dGPU | NVIDIA, turned off in macOS. HDMI is wired to it, so there is no external screen |
| Screen | BOE NE156FHM-NX9, 1920x1080, 144 Hz |
| Audio | Realtek ALC257 (sold as ALC3287) on `8086:7AD0` |
| Wi-Fi | Realtek RTL8852BE `10ec:b852`, works with AirportRTW |
| Bluetooth | Realtek, on the same card. Doesn't work yet |
| macOS | Tahoe 26.7.1 (25G241) |

## What works

* Metal acceleration. Full desktop, Liquid Glass and animations.
* 144 Hz on the built in screen. 60 Hz is also in the menu.
* Wi-Fi (RTL8852BE) with AirportRTW, once you download the kext and turn it on. Passed 7 checks over 6 boots with 0% packet loss.
* Speakers and volume keys (layout 18).
* Brightness slider and brightness keys.
* YouTube in Safari.
* Keyboard and trackpad.
* Booting OpenCore from the internal EFI partition next to Windows.

## What doesn't work (yet)

* Bluetooth. The USB port it sits on isn't in the USB map, so macOS doesn't see it. `bluetoothd` restarts every ~20 seconds.
* External screens. HDMI is on the NVIDIA GPU.
* Hardware video decode. Video decodes on the CPU, which is fine for YouTube.
* OpenGL is software only. macOS has no Tiger Lake OpenGL driver.
* Full volume is quieter than on Windows.
* Not tested by me: sleep, headphone jack, microphone, battery percentage. Ethernet was plugged in next to Wi-Fi, but I didn't test it
  on its own or unplug it while Wi-Fi was on.

## Known issues

* **NootedGreen kernel panics (not fixed yet).** The Mac has logged 7 kernel panic reports from 2026-10-28 to 2026-10-31, and it
  restarts by itself each time. I looked closely at the 4 newest. They came 1 to 26 minutes after boot, and the backtraces point to
  NootedGreen, Apple's Tiger Lake framebuffer and the graphics accelerator (one in WindowServer, one in `powermetrics`, two in
  `kernel_task`). None of the 7 reports mentions Wi-Fi code. Save your work often.
* The GPU resets once about 40 seconds after boot. It recovers and nothing visible breaks.
* Some lag. System Settings pages can take 5 to 15 seconds to open, and wallpaper changes are slow.

## Security

This setup lowers macOS security. It needs:

* `amfi_get_out_of_my_way=1` (boot argument) turns off AMFI, so macOS stops enforcing code signatures and entitlements of apps and kexts.
* `csr-active-config` = `130A0000` partly turns off SIP. It lets unsigned and unapproved kexts load, lets protected system files be
  changed, and lets a modified system volume boot.
* `SecureBootModel` = `Disabled` turns off Apple Secure Boot in OpenCore, so nothing checks that macOS's boot files are signed by Apple.
* `DisableIoMapper` = `true` (kernel quirk) turns off macOS's use of VT-d, the IOMMU that protects memory from devices.

Anything you run has more power over the system. Only install software you trust.

## Before you start

* **Four kexts are not included and are switched off.** `AirportRTW`, `SimpleHDA`, `USBToolBox` and `RealtekRTL8111` are in
  `Kernel > Add` with `Enabled` set to `false`, so a fresh boot doesn't look for files that aren't there. Until you add and enable them
  there is no Wi-Fi, no sound, no Ethernet and no USB map. See Install step 1.
* **The USB map is made for my laptop.** `UTBMap.kext` lists the USB ports of this exact machine. Build your own map with the
  [USBToolBox tool](https://github.com/USBToolBox/tool) and replace `UTBMap.kext`. Ports that aren't in the map don't work, and a map
  from another machine can leave out ports on yours (camera, Bluetooth, USB sockets).
* **OpenCore must stay the DEBUG build.** It's included. The RELEASE build of OpenCore 1.0.9 panics on this laptop about 2 seconds
  into boot with "Non-monotonic time". The DEBUG build with `Target = 0` boots fast and quiet.
* **Security gets lowered.** Read the [Security](#security) section first. The graphics stack needs AMFI off and SIP partly off.
  AirportRTW needs `SecureBootModel = Disabled`.
* **Two EFI drivers differ from my own EFI.** Apple's `HfsPlus.efi` and `apfs_aligned.efi` can't be shared. This EFI uses
  OpenCore's `OpenHfsPlus.efi` and `EnableJumpstart = true` instead, so OpenCore loads the APFS driver from your macOS disk.
  I tested this combination on the LOQ and it boots fine. If the boot picker doesn't show macOS on your machine, look there first.

## Install

1. **Download these four kexts** and put them in `EFI/OC/Kexts/`. They aren't included because of their licences. The config
   already lists all four, but switched off. After you copy each kext, open `config.plist` (OpenCore Configurator, ProperTree or a
   text editor), go to `Kernel > Add`, find the entry and set `Enabled` to `true`.

   | Kext | Turn it on for | Download |
   |---|---|---|
   | SimpleHDA (v1.0.0 RELEASE) | Sound. Needs `keepsyms=1` | https://github.com/laobamac/SimpleHDA/releases |
   | USBToolBox | The USB map. Turn it on together with `UTBMap.kext` | https://github.com/USBToolBox/kext/releases |
   | RealtekRTL8111 | Ethernet | https://github.com/Mieze/RTL8111_driver_for_OS_X/releases |
   | AirportRTW (V2.2.1, `AirportRTW-Tahoe-Release.zip`) | Wi-Fi. The Wi-Fi card SSDT is already off in this config | https://github.com/souissiak1-maker/AirportRTW/releases |

   `UTBMap.kext` is already enabled, but it needs USBToolBox. Until USBToolBox is on, the map does nothing. Build your own map first
   (see "Before you start").
2. **Make your own SMBIOS.** In `config.plist`, `PlatformInfo > Generic` has `SystemSerialNumber`, `MLB`, `SystemUUID` and `ROM`
   empty. Run [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS), pick "Generate SMBIOS" for **MacBookPro16,2**, and fill them in.
   Use your Ethernet MAC address (6 bytes) for `ROM`. Never use someone else's serials.
3. **Keep `keepsyms=1` in the boot arguments.** SimpleHDA refuses to start without it, and then there is no sound.
4. Unzip `LOQ-15IAX9E-EFI.zip`, put the `EFI` folder on a FAT32 USB stick and boot from it.
5. **Run the IrisXe installer once in macOS.** Get it from
   [JulioNectar/Hackintosh-XPS13Plus-IrisXe-Tahoe](https://github.com/JulioNectar/Hackintosh-XPS13Plus-IrisXe-Tahoe), then run
   `./install.sh --check` and `./install.sh`. It installs Apple's Tiger Lake graphics kexts, the patched Metal driver and the shader
   compiler. Approve the system extensions when macOS asks, then reboot. Without this step there is no acceleration.
   Use the NootedGreen in this EFI, not the one in the project's EFI folder.
6. Optional: install OpenCore to the internal EFI partition. **Do this from Windows, not from macOS** (see the next section).
   * Back up the EFI partition first.
   * In an admin PowerShell: `mountvol Z: /s`, then copy only `EFI\OC`:
     `robocopy <USB>:\EFI\OC Z:\EFI\OC /E`. Don't touch `Z:\EFI\Microsoft` or `Z:\EFI\Boot`.
   * Check it read-only: `chkdsk Z:` (no `/f`). It must report no problems. Then `mountvol Z: /d`.
   * Add a firmware boot entry: `bcdedit /copy {bootmgr} /d "OpenCore"`, then
     `bcdedit /set {<new id>} path \EFI\OC\OpenCore.efi` and `bcdedit /set {fwbootmgr} displayorder {<new id>} /addlast`.
   * Don't use OpenCore's `LauncherOption` here. It would put OpenCore in front of Windows, and the macOS ACPI tables turn off the
     NVIDIA GPU and the Wi-Fi card.

## The EFI short-name bug (dead keyboard and trackpad)

After I copied the EFI to the internal EFI partition from macOS, the internal boot came up with no keyboard and no trackpad.
Booting the same EFI from USB was fine.

What happened:

* FAT32 stores a short 8.3 name next to every long file name. macOS gave two plugins in the same folder the same short name:
  `VoodooGPIO.kext` and `VoodooI2CServices.kext` both got `VOODOO~2.KEX`, and so did `VoodooPS2Trackpad.kext` and
  `VoodooPS2Keyboard.kext`. macOS's own `fsck_msdos` doesn't report this.
* Windows ran chkdsk on the EFI partition (Startup Repair does this too). It renamed one of each pair to `VOODOO~2.KE0` but left the
  long name's checksum pointing at the old short name. The firmware then only sees the short name, so OpenCore couldn't find
  `VoodooI2CServices.kext` and `VoodooPS2Keyboard.kext`. No keyboard, no trackpad.

How to check (Windows, admin PowerShell):

```
mountvol Z: /s
chkdsk Z:
mountvol Z: /d
```

Without `/f` chkdsk only reads. Look for "More than one ... entry in folder". No such line and no problems found = fine.

How to fix it:

* **If chkdsk shows "More than one" and nothing has been repaired yet:** rename one file of each pair to a temporary name and back,
  in Windows. Windows then gives it a new, unique short name. For example:
  ```
  ren Z:\EFI\OC\Kexts\VoodooI2C.kext\Contents\PlugIns\VoodooI2CServices.kext VoodooI2CServices.tmp
  ren Z:\EFI\OC\Kexts\VoodooI2C.kext\Contents\PlugIns\VoodooI2CServices.tmp VoodooI2CServices.kext
  ```
  Do the same for `VoodooPS2Keyboard.kext` in `VoodooPS2Controller.kext\Contents\PlugIns`. Run `chkdsk Z:` again: it must
  report no problems.
* **If it was already "repaired"** (you see names like `VOODOO~2.KE0`, or the internal boot has no keyboard): delete the broken kext
  folders on `Z:` and copy them again from the USB stick with `robocopy` in Windows. Then run `chkdsk Z:` again.

How to avoid it: write the internal EFI partition from Windows only (`robocopy`), and check it with a read-only `chkdsk` after.
Never let chkdsk `/f` run on it until the read-only check is clean.

## Boot arguments

```
-ngreentglwithgfx -allow3d -disablegfxfirmware -ngreenv189 -ngreenkeepefi -ngreenxefix -ngreenfixbootmode
amfi_get_out_of_my_way=1 ngreenV142=3 ngreen-eu=16 ngreen-l3=4 keepsyms=1
```

* `-allow3d` is required. Apple's accelerator won't start without it.
* `ngreenV142=3` uses Apple's real blit code.
* `-ngreenv189` fixes a WindowServer freeze.
* `ngreen-eu=16` and `ngreen-l3=4` match this chip. Other values hang the boot or leave pixels missing.
* `keepsyms=1` is needed by SimpleHDA.
* For troubleshooting add `-v debug=0x100`. For NootedGreen's debug logs add `-ngreenverbose`.

## What I changed in NootedGreen

The `patches` folder has my changes as git patches. They apply on top of the IrisXe project's main branch
(`git am patches/*.patch`). The kext in this EFI is built from them with `zsh build-nootedgreen.sh release`.

| Patch | What it does |
|---|---|
| 0001 | Accept `ngreen-eu=16` and `32`. This chip has 16 EU and the old list started at 48 |
| 0002 | Only one screen connector, the built in eDP. This chip has no Type C display port |
| 0003 | `XE-GMS`. Apple's screen driver works out the reserved GPU memory with the Tiger Lake formula. For this BIOS (GMS 0xFE, 60 MiB) it gets a negative size and never starts. The patch puts in the real size |
| 0004 | `XE-AUXA`. Keeps power on for the screen's AUX channel. Without it macOS can't read the screen's EDID |
| 0005 | `XE-FLIPLOG`. A debug log of screen flips |
| 0006 | `XE-NOSCANSYNC`. Skips a scanline wait during window swaps |
| 0007 | Shader compiler adapter. Debug dumps and a 250 ms wait only run when `XE_DUMP` is set. This made Liquid Glass changes fast |
| 0008 | Repeating debug logs only with `-ngreenverbose`, and a release build option |

The 144 Hz mode comes from a patched EDID in `config.plist` (`AAPL00,override-has-edid`). Brightness comes from `SSDT-PNLF.aml`.

## Credits

* **JulioNectar** for the IrisXe project. It does the hard work of running Apple's Tiger Lake driver on Xe graphics.
* **sgiammori (StezzaPilot)** for NootedGreen.
* **souissiak1-maker** for AirportRTW, and OpenIntelWireless for itlwm, which it builds on.
* acidanthera for OpenCore, Lilu, WhateverGreen, VirtualSMC and AppleALC.
* laobamac for SimpleHDA, dhinakg for USBToolBox, Mieze for RealtekRTL8111, and the VoodooI2C and VoodooPS2 teams.

## Licence

NootedGreen and my patches are under the [Thou Shalt Not Profit License 1.0](LICENSES/NootedGreen-IrisXe-TSNPL-1.0.txt). You can use,
change and share them, but not for profit, and changed versions keep the same licence. Every other part keeps its own licence, see
[LICENSES/INDEX.md](LICENSES/INDEX.md). There are no Apple binaries in this repository. This comes with no warranty. It can leave your
Mac unbootable.

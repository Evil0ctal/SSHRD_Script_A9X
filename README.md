# SSHRD_Script_A9X

**English** | [简体中文](README.zh-CN.md)

- An unofficial, enhanced fork of [Nathan (verygenericname)'s SSHRD_Script](https://github.com/verygenericname/SSHRD_Script), built on top of the [iPh0ne4s fork](https://github.com/iPh0ne4s/SSHRD_Script). Lineage: `verygenericname` → `iPh0ne4s` → `Evil0ctal (this fork)`
- This script has been tested working on Ubuntu 24.04, macOS Sonoma hackintosh, and ARM macOS 26 (Apple Silicon). There are still no warranties, please use at your own risk
- Linux or macOS required. Virtual machine and windows are not and will never be supported even if some features are available on them. It is recommended to use USB-A cable and Intel PC
- A7-A11 devices only. For 32-bit devices, use [Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit)

## What's new in this fork (SSHRD_Script_A9X)
This fork keeps every feature of the upstream iPh0ne4s version and adds the following refinements. None of the commands below changed — the improvements are in output organization, edge-case coverage, and robustness:
- **A9X device support (namesake of this fork)** — the upstream `verygenericname` and `iPh0ne4s` branches may fail to create the ramdisk or fail to boot on A9X-based devices (e.g. iPad Pro 1st gen / iPad mini 4-class A9X hardware). This fork has been tested on real A9X hardware and can successfully **create the ramdisk, boot it, and dump iOS 9 `.shsh2` blobs**.
- **Unified `userdata/` output directory** — all generated files are now grouped under `userdata/` instead of being scattered in the project root: dumped blobs go to `userdata/shsh2/`, activation backups to `userdata/activation_records/<serial>/`, and NAND/disk images to `userdata/disk/`.
- **Better phone-number detection** — activation backup now parses `CDMANetworkPhoneNumber`, `NetworkPhoneNumber`, and `SIMPhoneNumber` in addition to the default field, so CDMA and multi-SIM devices are handled.
- **Partial activation restore on 64-bit iOS 7.0–8.2** — instead of aborting, the script now restores `com.apple.commcenter.device_specific_nobackup.plist` and `IC-Info.sisv` (with correct permissions/ownership) via `mount_hfs`. `activation_record.plist` is still skipped because restoring it causes a bootloop.
- **Automatic hacktivate on iOS 10.0–10.1.1** — writes the required `CacheExtra` key into `MobileGestalt.plist` to work around these versions not recognizing `activation_record.plist`.
- **More robust error handling** — added non-fatal `|| true` guards, `iproxy`/`usbmuxd` cleanup on failure paths, and clearer, path-specific success/error messages.

See the [Comparison with upstream](#comparison-with-upstream) section below for a full breakdown.

## Basic Usage: create ramdisk, boot ramdisk, SSH into device
0. Clone this repository:   
`git clone https://github.com/Evil0ctal/SSHRD_Script_A9X --recursive`   
cd into the SSHRD_Script_A9X directory. Run `chmod +x sshrd.sh` if running the script for the first time
1. Enter DFU mode, run `./sshrd.sh <ramdisk version>` to create ramdisk
  - For iOS 7-9 devices, run `./sshrd.sh 10.0.1`
    - A7 iOS 7 devices will be stuck in a black screen recovery mode after loading a higher version ramdisk, boot 8.0 ramdisk to fix this. It is the only case that iOS 8 ramdisk should be used
  - For iOS 10+ devices, use device version as ramdisk version, e.g., run `./sshrd.sh 11.2.2` for iOS 11.2.2 iPhone 6s, or the closest one if target ipsw doesn't exist, e.g., `./sshrd.sh 11.1` for iOS 11.0.1 iPhone X
    - Use 14.6 ramdisk on iOS 15 devices if iOS 15 ramdisk crashes
    - A wrong ramdisk version might cause bootloop, and this always happens on 16.4+ devices, check device version first
  - It is common to see "an error occurred" or device rebooting, just repeat the process, re-enter DFU if necessary
2. Run `./sshrd.sh boot` to boot ramdisk, if unable to connect to device, unplug and replug the cable
3. Run `./sshrd.sh ssh` to SSH into device, if the terminal says "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!", run `rm -f ~/.ssh/known_hosts` and try again
  - GUI tools such as [FileZilla](https://filezilla-project.org/) can be used to access device
## Other Commands
### In this part, unless otherwise specified, all the commands should be executed after booting ramdisk, i.e., after creating ramdisk and running `./sshrd.sh boot`, before `./sshrd.sh ssh`
- Reboot device: `./sshrd.sh reboot`
- Erase device without updating on iOS 9+: `./sshrd.sh reset`
- Dump onboard blobs: `./sshrd.sh dump-blobs`
- Remove temporary files: `./sshrd.sh clean` (run this one when no device)
- Exit recovery mode: `./sshrd.sh --exit-recovery` (run this one in recovery mode)
- Backup and restore activation files (iOS 10+)
  - Run `./sshrd.sh --backup-activation` to backup activation files, `./sshrd.sh --restore-activation` to restore them
- Backup and restore activation files (iOS 7-9, requires open menu)
  - Commands are `./sshrd.sh --backup-activation-hfs` and `./sshrd.sh --restore-activation-hfs`
  - On 7.0-9.3.5, activation files cannot be downloaded using scp or sftp command, instead they can be moved to /private/var/mobile/Media (the directory that is accessible in normal mode without a jailbreak) to become downloadable, therefore passcode locked devices are not supported
  - On 8.3+, activation files can be restored in the same way, place them in /private/var/mobile/Media first. On 7.0-8.2, however, moving them back will cause bootloop
- Backup and restore the entire contents on NAND (dangerous, might cause bootloop)
  - Run `./sshrd.sh --dump-nand` to backup NAND to a .gz file, `./sshrd.sh --restore-nand` to restore the .gz file to /dev/disk0 on device. Do not mount any partition before running these commands
  - On 7.0-10.2.1, there are also a few more options: `./sshrd.sh --dump-disk0s1s1`, `./sshrd.sh --restore-disk0s1s1`, `./sshrd.sh --dump-disk0s1s2`, `./sshrd.sh --restore-disk0s1s2`
- Install TrollStore on 14.0-16.6.1, 16.7 RC, 17.0: `./sshrd.sh --install-trollstore`
- Un-disable and get unlimited passcode attempts on iOS 7-8: `./sshrd.sh --brute-force`
## Notes & Known Issues
- "kex_exchange_identification: read: Connection reset by peer" and "Connection reset by 127.0.0.1 port 2222" indicate an SSH connection issue, if this occurs, try the following solutions: unplug and replug device, change cable, re-enter ramdisk mode, reboot PC
- On Linux, A7 devices must be manually placed into pwnDFU using [ipwnder_lite](https://github.com/LukeZGD/ipwnder_lite). [Usage](https://github.com/LukeZGD/Legacy-iOS-Kit/wiki/Pwning-Using-Another-iOS-Device)
- If there are permission denied, terminated or operation not permitted errors with sshrd.sh, try running sshrd.sh with sudo, especially on macOS
- Even if mounting /mnt2 as read/write, some files like photos still won't be downloadable, that's due to userdata encryption and there's actually nothing wrong
- Devices downgraded with turdus merula might not be able to mount /mnt2
- iOS 15 ramdisk will crash when saving activation files for unknown reason, use 14.6 ramdisk instead
- iOS 7-9 ramdisk is unusable except early iOS 8 versions, and iOS 8 ramdisk is only for exiting black screen recovery mode caused by a higher version ramdisk on A7 iOS 7 devices
- iOS 16+ ramdisk is partially broken. On iOS 16 iPhone 8/Plus, mounting /mnt2 succeeds on freshly restored/reset devices but fails on most passcode locked devices, on iPhone X /mnt2 cannot be mounted at all, iPads untested. There is no ETA to fix the issue, it probably requires cracking paid ramdisk tools to figure out how to properly mount iOS 16+ filesystems

## Comparison with upstream
The table below shows where each layer of features comes from. Most of the heavy lifting (all the extra commands, tooling and legacy-iOS support) was done in the `iPh0ne4s` fork; this `A9X` fork focuses on polishing that work.

| Feature | verygenericname (original) | iPh0ne4s | Evil0ctal (this fork) |
| --- | :---: | :---: | :---: |
| Create / boot ramdisk, SSH, reboot, reset, clean, dump-blobs | ✅ | ✅ | ✅ |
| Activation backup/restore (`--backup-activation` / `--restore-activation`, incl. HFS variants) | ❌ | ✅ | ✅ |
| NAND & partition dump/restore (`--dump-nand`, `--dump-disk0s1sN`, ...) | ❌ | ✅ | ✅ |
| TrollStore install (`--install-trollstore`) | ❌ | ✅ | ✅ |
| Passcode brute-force / un-disable (`--brute-force`) | ❌ | ✅ | ✅ |
| Exit recovery (`--exit-recovery`) | ❌ | ✅ | ✅ |
| `gaster` / `kairos` / `ivkey` toolchain + iOS 7–9 (A7) support | ❌ | ✅ | ✅ |
| A9X device support (ramdisk create + boot + iOS 9 blob dump) | ⚠️ may fail | ⚠️ may fail | ✅ tested |
| Unified `userdata/` output layout | ❌ | ❌ | ✅ |
| Multi-field phone-number detection (CDMA / multi-SIM) | ❌ | ❌ | ✅ |
| Partial activation restore on 64-bit iOS 7.0–8.2 | ❌ | ❌ | ✅ |
| Automatic hacktivate on iOS 10.0–10.1.1 | ❌ | ❌ | ✅ |
| Extra robustness (non-fatal guards, proxy cleanup, clearer messages) | ❌ | partial | ✅ |

## Credits & Lineage
- [Nathan (verygenericname)](https://github.com/verygenericname) — original SSHRD_Script
- [iPh0ne4s](https://github.com/iPh0ne4s) — the fork that added activation/NAND/TrollStore/brute-force commands, the `gaster`/`kairos`/`ivkey` toolchain and legacy iOS 7–9 support
- [Evil0ctal](https://github.com/Evil0ctal) — this fork (`SSHRD_Script_A9X`): unified output directory, activation edge-case handling and robustness improvements
- [tihmstar](https://github.com/tihmstar) for pzb / original iBoot64Patcher / img4tool
- [xerub](https://github.com/xerub) for img4lib and restored_external in the ramdisk
- [Cryptic](https://github.com/Cryptiiiic) for the iBoot64Patcher fork
- [opa334](https://github.com/opa334) for TrollStore
- [Nebula](https://github.com/itsnebulalol) for QOL fixes to the original script
- [Ploosh](https://github.com/plooshi) for KPlooshFinder
# SSHRD_Script_A9X

[English](README.md) | **简体中文**

- 这是一个非官方增强版分支，基于 [Nathan (verygenericname) 的 SSHRD_Script](https://github.com/verygenericname/SSHRD_Script)，并在 [iPh0ne4s 分支](https://github.com/iPh0ne4s/SSHRD_Script) 之上进一步开发。血缘关系：`verygenericname` → `iPh0ne4s` → `Evil0ctal（本分支）`
- 本脚本已在 Ubuntu 24.04、macOS Sonoma 黑苹果，以及 ARM macOS 26（Apple Silicon）上测试通过。但仍不提供任何保证，请自担风险使用
- 需要 Linux 或 macOS。虚拟机和 Windows 不受支持，也永远不会支持，即使部分功能可能可用。推荐使用 USB-A 数据线和 Intel 平台的电脑
- 仅支持 A7-A11 设备。32 位设备请使用 [Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit)

## 本分支的更新内容（SSHRD_Script_A9X）
本分支保留了上游 iPh0ne4s 版本的全部功能，并新增了以下改进。下列改进不涉及任何命令的变动，而是集中在输出组织、边界情况覆盖和健壮性方面：
- **A9X 设备支持（本分支得名之处）** —— 上游 `verygenericname` 和 `iPh0ne4s` 分支在 A9X 处理器的设备（如初代 iPad Pro / A9X 硬件）上可能导致 ramdisk 创建失败或引导失败。本分支已在真实 A9X 硬件上实测，可成功**创建 ramdisk、引导 ramdisk，并成功备份出 iOS 9 版本的 `.shsh2` blobs 文件**。
- **统一的 `userdata/` 输出目录** —— 所有生成的文件现在集中归于 `userdata/` 目录下，而不再散落在项目根目录：导出的 blobs 存放到 `userdata/shsh2/`，激活文件备份存放到 `userdata/activation_records/<序列号>/`，NAND/磁盘镜像存放到 `userdata/disk/`。
- **更完善的电话号码识别** —— 激活备份现在除默认字段外，还会解析 `CDMANetworkPhoneNumber`、`NetworkPhoneNumber` 和 `SIMPhoneNumber`，从而支持 CDMA 和双卡/多卡设备。
- **64 位 iOS 7.0–8.2 的部分激活恢复** —— 脚本不再直接中止，而是通过 `mount_hfs` 恢复 `com.apple.commcenter.device_specific_nobackup.plist` 和 `IC-Info.sisv`（并修正权限/属主）。`activation_record.plist` 仍会跳过，因为恢复它会导致引导循环（bootloop）。
- **iOS 10.0–10.1.1 自动 hacktivate** —— 向 `MobileGestalt.plist` 写入所需的 `CacheExtra` 键，以绕过这些版本不识别 `activation_record.plist` 的问题。
- **更健壮的错误处理** —— 增加了非致命的 `|| true` 保护、失败路径下的 `iproxy`/`usbmuxd` 清理，以及更清晰、带具体路径的成功/错误提示。

完整对比请见下方的 [与上游的对比](#与上游的对比) 一节。

## 基本用法：制作 ramdisk、启动 ramdisk、SSH 连接设备
0. 克隆本仓库：   
`git clone https://github.com/Evil0ctal/SSHRD_Script_A9X --recursive`   
进入 SSHRD_Script_A9X 目录。首次运行脚本请先执行 `chmod +x sshrd.sh`
1. 进入 DFU 模式，运行 `./sshrd.sh <ramdisk 版本>` 制作 ramdisk
  - iOS 7-9 设备，运行 `./sshrd.sh 10.0.1`
    - A7 的 iOS 7 设备在加载了更高版本的 ramdisk 后会卡在黑屏恢复模式，启动 8.0 的 ramdisk 即可修复。这是唯一需要用到 iOS 8 ramdisk 的场景
  - iOS 10+ 设备，使用设备版本作为 ramdisk 版本，例如为 iOS 11.2.2 的 iPhone 6s 运行 `./sshrd.sh 11.2.2`；若目标 ipsw 不存在，则使用最接近的版本，例如为 iOS 11.0.1 的 iPhone X 运行 `./sshrd.sh 11.1`
    - 如果 iOS 15 的 ramdisk 崩溃，可在 iOS 15 设备上改用 14.6 的 ramdisk
    - 错误的 ramdisk 版本可能导致引导循环，这在 16.4+ 设备上一定会发生，请先确认设备版本
  - 出现 "an error occurred" 或设备重启是常见现象，重复流程即可，必要时重新进入 DFU
2. 运行 `./sshrd.sh boot` 启动 ramdisk，如果无法连接设备，请拔插数据线
3. 运行 `./sshrd.sh ssh` 通过 SSH 连接设备，如果终端提示 "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"，执行 `rm -f ~/.ssh/known_hosts` 后重试
  - 可使用 [FileZilla](https://filezilla-project.org/) 等图形化工具访问设备

## 其他命令
### 本部分中，除非另有说明，所有命令都应在启动 ramdisk 之后执行，即在制作 ramdisk 并运行 `./sshrd.sh boot` 之后、`./sshrd.sh ssh` 之前
- 重启设备：`./sshrd.sh reboot`
- 在 iOS 9+ 上抹除设备而不更新：`./sshrd.sh reset`
- 导出机上 blobs：`./sshrd.sh dump-blobs`
- 清理临时文件：`./sshrd.sh clean`（无设备连接时运行此命令）
- 退出恢复模式：`./sshrd.sh --exit-recovery`（在恢复模式下运行此命令）
- 备份和恢复激活文件（iOS 10+）
  - 运行 `./sshrd.sh --backup-activation` 备份激活文件，运行 `./sshrd.sh --restore-activation` 恢复它们
- 备份和恢复激活文件（iOS 7-9，需要 open menu）
  - 命令为 `./sshrd.sh --backup-activation-hfs` 和 `./sshrd.sh --restore-activation-hfs`
  - 在 7.0-9.3.5 上，激活文件无法通过 scp 或 sftp 命令下载，需要先将它们移动到 /private/var/mobile/Media（该目录在正常模式下无需越狱即可访问）才可下载，因此不支持带锁屏密码的设备
  - 在 8.3+ 上，可用同样方式恢复激活文件，先将它们放入 /private/var/mobile/Media。但在 7.0-8.2 上，将它们移回会导致引导循环
- 备份和恢复整个 NAND 内容（危险，可能导致引导循环）
  - 运行 `./sshrd.sh --dump-nand` 将 NAND 备份为 .gz 文件，运行 `./sshrd.sh --restore-nand` 将 .gz 文件恢复到设备的 /dev/disk0。运行这些命令前不要挂载任何分区
  - 在 7.0-10.2.1 上，还有几个额外选项：`./sshrd.sh --dump-disk0s1s1`、`./sshrd.sh --restore-disk0s1s1`、`./sshrd.sh --dump-disk0s1s2`、`./sshrd.sh --restore-disk0s1s2`
- 在 14.0-16.6.1、16.7 RC、17.0 上安装 TrollStore：`./sshrd.sh --install-trollstore`
- 在 iOS 7-8 上解除禁用并获得无限次密码尝试：`./sshrd.sh --brute-force`

## 注意事项与已知问题
- "kex_exchange_identification: read: Connection reset by peer" 和 "Connection reset by 127.0.0.1 port 2222" 表示 SSH 连接问题，若出现，请尝试以下方案：拔插设备、更换数据线、重新进入 ramdisk 模式、重启电脑
- 在 Linux 上，A7 设备必须使用 [ipwnder_lite](https://github.com/LukeZGD/ipwnder_lite) 手动进入 pwnDFU。[用法](https://github.com/LukeZGD/Legacy-iOS-Kit/wiki/Pwning-Using-Another-iOS-Device)
- 如果 sshrd.sh 出现 permission denied、terminated 或 operation not permitted 等错误，尝试用 sudo 运行 sshrd.sh，尤其是在 macOS 上
- 即使将 /mnt2 以读写方式挂载，某些文件（如照片）仍无法下载，这是由于用户数据加密所致，实际上并没有问题
- 使用 turdus merula 降级过的设备可能无法挂载 /mnt2
- iOS 15 的 ramdisk 在保存激活文件时会因未知原因崩溃，请改用 14.6 的 ramdisk
- iOS 7-9 的 ramdisk 除早期 iOS 8 版本外均不可用，而 iOS 8 的 ramdisk 仅用于修复 A7 iOS 7 设备上因加载更高版本 ramdisk 导致的黑屏恢复模式
- iOS 16+ 的 ramdisk 部分损坏。在 iOS 16 的 iPhone 8/Plus 上，对刚恢复/重置过的设备可成功挂载 /mnt2，但对大多数带锁屏密码的设备会失败；在 iPhone X 上完全无法挂载 /mnt2；iPad 未测试。该问题暂无修复时间表，可能需要破解付费 ramdisk 工具才能弄清如何正确挂载 iOS 16+ 的文件系统

## 与上游的对比
下表展示了每一层功能的来源。绝大部分繁重工作（所有额外命令、工具链和旧版 iOS 支持）都在 `iPh0ne4s` 分支中完成；本 `A9X` 分支专注于对这些工作进行打磨。

| 功能 | verygenericname（原始） | iPh0ne4s | Evil0ctal（本分支） |
| --- | :---: | :---: | :---: |
| 制作 / 启动 ramdisk、SSH、reboot、reset、clean、dump-blobs | ✅ | ✅ | ✅ |
| 激活文件备份/恢复（`--backup-activation` / `--restore-activation`，含 HFS 变体） | ❌ | ✅ | ✅ |
| NAND 及分区导出/恢复（`--dump-nand`、`--dump-disk0s1sN` 等） | ❌ | ✅ | ✅ |
| 安装 TrollStore（`--install-trollstore`） | ❌ | ✅ | ✅ |
| 密码暴力解锁 / 解除禁用（`--brute-force`） | ❌ | ✅ | ✅ |
| 退出恢复模式（`--exit-recovery`） | ❌ | ✅ | ✅ |
| `gaster` / `kairos` / `ivkey` 工具链 + iOS 7–9（A7）支持 | ❌ | ✅ | ✅ |
| A9X 设备支持（ramdisk 创建 + 引导 + iOS 9 blob 备份） | ⚠️ 可能失败 | ⚠️ 可能失败 | ✅ 已实测 |
| 统一的 `userdata/` 输出布局 | ❌ | ❌ | ✅ |
| 多字段电话号码识别（CDMA / 多卡） | ❌ | ❌ | ✅ |
| 64 位 iOS 7.0–8.2 的部分激活恢复 | ❌ | ❌ | ✅ |
| iOS 10.0–10.1.1 自动 hacktivate | ❌ | ❌ | ✅ |
| 额外的健壮性（非致命保护、代理清理、更清晰的提示） | ❌ | 部分 | ✅ |

## 致谢与血缘
- [Nathan (verygenericname)](https://github.com/verygenericname) —— 原始 SSHRD_Script
- [iPh0ne4s](https://github.com/iPh0ne4s) —— 添加了激活/NAND/TrollStore/暴力解锁命令、`gaster`/`kairos`/`ivkey` 工具链以及旧版 iOS 7–9 支持的分支
- [Evil0ctal](https://github.com/Evil0ctal) —— 本分支（`SSHRD_Script_A9X`）：统一输出目录、激活边界情况处理和健壮性改进
- [tihmstar](https://github.com/tihmstar) —— pzb / 原始 iBoot64Patcher / img4tool
- [xerub](https://github.com/xerub) —— img4lib 及 ramdisk 中的 restored_external
- [Cryptic](https://github.com/Cryptiiiic) —— iBoot64Patcher 分支
- [opa334](https://github.com/opa334) —— TrollStore
- [Nebula](https://github.com/itsnebulalol) —— 对原始脚本的诸多体验优化
- [Ploosh](https://github.com/plooshi) —— KPlooshFinder

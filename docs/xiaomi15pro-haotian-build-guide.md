# 小米 15 Pro（haotian / 2410DPN6CC）HyperOS 4 构建 SukiSU + SUSFS + Droidspaces 内核

> 设备与配置由本机 `adb` 实测核实（2026-11-10），报告见工作区 `reports/device-probe.md`、`reports/compat-evidence.md`、`reports/patch-kmi.md`。
> 本文只讲**怎么配置、怎么构建、怎么刷、怎么回滚**；构建和刷机请你自己执行。

---

## 0. 先看结论（务必读完）

### 0.1 配置结论

| 项目 | 结论 |
|---|---|
| 内核线 | `android15-6.6`，子版本 **6.6.118**（= ACK 分支 `common-android15-6.6-2026-01`） |
| 为什么是 6.6.118 | 设备原厂内核 `6.6.118-android15-8-...-4k`；仓库 `data/android15/6.6.json` 明确 `2026-01 ↔ 6.6.118`；2026-11-06 的最新 Release `v2.3.0-r10` 里仍有 `android15-6.6.118-2026-01-*` 产物，构建链可用 |
| KMI / 页大小 | `android15-8`，4K 页（`CONFIG_LOCALVERSION="-4k"`、`pagesize=4096`） |
| KSU 变体 | `SukiSU`（builtin 分支 = SUSFS inline hooks） |
| Droidspaces | 选 `678`（上游 Droidspaces 官方文档推荐先试 6/7/8 padding 槽；本树实测可干净应用；bootloop 再换 `345`。**不要选 123**：在 android15-6.6 的 `sched.h` 上打不上） |
| NTSync | 先关（补丁已验证能干净打上；确认内核能开机后再单独开它） |
| 已开启的增强 | `use_zram`（ZRAM 增强）、`use_bbg`（BBG 防格机）、`use_networking`（网络增强）、`cve_2026_43499_patch`（GhostLock 修复）默认全开 |
| 保持关闭 | `use_nomount` / `use_rekernel` / `use_kpm`（额外模块/补丁，非必需） |

### 0.2 但你先看这个：HyperOS 4 目前大概率刷了不能开机 ⚠️

这不是配置问题，是系统侧兼容问题，有同机型实测证据：

- 同型号 **小米 15 Pro，HyperOS 4.0.0.7（Android 17）**，刷本项目的 `android15 6.6.118` AnyKernel3 → **卡在 Mi Logo 之后的 HyperOS 画面死机**，只能线刷回原厂 boot 恢复；同一个内核刷 `OS3.0.309.0（Android 16）` 就正常。→ [issue #251](https://github.com/zzh20188/GKI_KernelSU_SUSFS/issues/251)
- 红米 K90（Android 17 / 6.6.118）同样失败，OS3 时期可以开机。→ [issue #252](https://github.com/zzh20188/GKI_KernelSU_SUSFS/issues/252)
- 项目作者 2026-09-20 回复：“其他内核作者也讲过，OS4 是不能用的”。截至 2026-11-10 没检索到任何 Android 17 / HyperOS 4 的成功案例。

**建议**：主力机先别刷。若一定要试，**先备份原厂 boot.img，并且只刷活动槽 `boot_a`**，保留 `boot_b` 作退路（见第 6、7 节）。

---

## 1. 设备实测信息（本次 adb 核实）

| 项目 | 实测值 |
|---|---|
| 型号 / 代号 | `2410DPN6CC` / `haotian`（Xiaomi 15 Pro，`ro.board.platform=sun`） |
| 系统 | HyperOS `OS4.0.0.19.XOBCNXM`，Android 17 / SDK 37，安全补丁 `2026-09-01` |
| 指纹 | `Xiaomi/haotian/haotian:17/CP2A.260605.016/OS4.0.0.19.XOBCNXM:user/release-keys` |
| 内核 | `6.6.118-android15-8-gb9cc6ec16bc8-abogki536571621-4k`，`#1 SMP PREEMPT Mon Jul 20 11:15:20 UTC 2026 aarch64` |
| 页大小 / KMI | 4K（`pagesize=4096`）/ `android15-8` |
| 分区 / 槽 | A/B，当前 `_a`；`boot_a=sde14`、`boot_b=sde49`；动态分区 + Virtual A/B；`vendor_dlkm` / `system_dlkm` 在 super 内（mapper `dm-7` / `dm-2`） |
| 当前 root | 无（user build、`adb root` 被拒、无 su/ksud）；但已装 **SukiSU Ultra 管理器 v4.3.0**（`com.sukisu.ultra`） |
| 当前内核配置 | `/proc/config.gz` 可读：`SYSVIPC/POSIX_MQUEUE/PID_NS/USER_NS/DEVTMPFS` **全部未开**，`IPC_NS` 未定义；`NET_NS/CGROUPS/MEMCG/PSI/TUN/VETH/BRIDGE` 已开；`MODVERSIONS=y`、`MODULE_SIG_FORCE` 未开；`ARM64_4K_PAGES=y` |
| 锁状态 | **自相矛盾，必须实测**：`ro.boot.verifiedbootstate=orange`（通常=已解锁），但 `ro.boot.flash.locked=1`、`ro.boot.vbmeta.device_state=locked`、`ro.secureboot.lockstate=locked` |

因为 `ro.boot.*` 来自 bootloader、`/proc/bootconfig` 又是 `0440 root:system` 不可读，**唯一可靠确认方式**是进 bootloader 只读查询（不刷写）：

```powershell
adb reboot bootloader
fastboot getvar unlocked          # 期望 yes
fastboot getvar flash.locked      # 期望 no/0
fastboot getvar current-slot      # 期望 a
fastboot oem device-info          # 小米/高通机型通常有 (bootloader) unlocked: true
fastboot reboot
```

如果 `unlocked: no`，本方案全部不成立，先走小米官方解锁（会清数据）。

---

## 2. 构建目标是怎么定出来的

1. 设备内核字符串里 `6.6.118` + `android15-8` + `-4k` → 对应 Android 15 GKI 6.6 线，子版本 118，KMI 8，4K 页。
2. 仓库 `data/android15/6.6.json`：`2026-01 → 6.6.118`、`2026-04 → 6.6.127`、`2026-07 → 6.6.139`、`2026-10 → 6.6.143`、`lts → 6.6.144`。`build.yml` 用 `repo init -b common-<android>-<kernel>-<os_patch_level>`，所以 `os_patch_level` 选 **`2026-01`** 才会检出 6.6.118。
3. 原厂内核同子版本是刚需：厂商 `vendor_dlkm` / `system_dlkm` 模块是按 6.6.118 这套 ABI 编译的。**不要**追新去选 6.6.127/143，除非你同时重编并刷入所有模块。
4. 注意日期：设备安全补丁 `2026-09-01` 是 **ROM 属性**，不要拿它当 `os_patch_level`；`os_patch_level` 是 ACK 月度分支编号。
5. 本次还做了独立核对：`android15-6.6-2026-01` 目前仍是**活跃分支**（镜像 packed-refs 里有，无 deprecated/ 前缀），实际 sparse 检出后 `Makefile` 为 `VERSION=6 PATCHLEVEL=6 SUBLEVEL=118`；`gki_defconfig` 里确实是 `CONFIG_LOCALVERSION="-4k"`。上游 Release `v2.3.0-r10`（2026-11-06）里也仍有 `android15-6.6.118-2026-01-*` 产物，说明这条构建链目前可用。

---

## 3. 已为你做好的仓库改动

新增一个**只构建 6.6.118 单个 job** 的工作流（不改上游任何逻辑，全部复用 `build.yml`）：

- 文件：`.github/workflows/kernel-haotian-6-6-118.yml`
- 名称：`内核构建 - 小米15Pro (6.6.118)`

它固定了：

| 输入 | 值 |
|---|---|
| `android_version` / `kernel_version` | `android15` / `6.6` |
| `sub_level` / `os_patch_level` | `118` / `2026-01` |
| `ksu_variant` / `ksu_mode` | `SukiSU` / `关闭`（关闭=同时集成 KSU+SUSFS） |
| `enable_susfs` | `true` |
| `droidspaces` | 默认 `678`（可选 `off/678/345`；`123` 在 6.6 上打不上，已从选项移除） |
| `droidspaces_ntsync` | 默认 `false` |
| `use_zram` | 默认 `true`（ZRAM 增强：builtin zram + LZ4KD/LZ4K） |
| `use_bbg` | 默认 `true`（BBG 防格机） |
| `use_networking` | 默认 `true`（BBRv3 / IPSet / CAKE / CIFS） |
| `cve_2026_43499_patch` | 默认 `true`（6.6.118 < 6.6.144，需要修；纯 rtmutex 内部改动，不影响 ABI） |
| `use_nomount` / `use_rekernel` / `use_kpm` / `supp_op` | 保持关闭 |
| `version` | 默认 `-android15-8-gb9cc6ec16bc8-abogki536571621-4k`，让产物 `uname -r` 与你的原厂内核**完全一致**（伪装/隐藏用途；不需要就清空） |
| `build_time` | 默认 `Mon Jul 20 11:15:20 UTC 2026`（与原厂 `#1 SMP PREEMPT ...` 一致；可留空用当前时间） |
| `artifact_upload_mode` | 默认 `上传全部`（要拿 `boot.img`） |

另外附带 `get-manager` job：直接产出与内核匹配的 **SukiSU 管理器 APK** 和 **SUSFS 模块 zip**。

> 直接用主入口 `main.yml` 也能构建，但它会跑 5 个 Android 版本 × 十几个子版本（80+ 个 job），纯浪费；这个专用工作流只跑 1 个。

---

## 4. 关键配置解释（为什么这么选）

### 4.1 Droidspaces 必须打 kABI 补丁，否则一定 bootloop

Droidspaces 需要 `CONFIG_SYSVIPC`、`CONFIG_POSIX_MQUEUE`、`CONFIG_IPC_NS`、`CONFIG_PID_NS`、`CONFIG_USER_NS`、`CONFIG_DEVTMPFS`。而原厂这些全是关的。

在 GKI 上，直接打开 `SYSVIPC / IPC_NS / POSIX_MQUEUE` 会给 `struct task_struct` 增加 `sysvsem/sysvshm` 字段、改变结构体布局 → 导出符号 CRC 变化 → 原厂 613 个模块拒载 → 卡开机/黑屏。

所以上游 Droidspaces 的做法是：**把这些字段挪进 `task_struct` 尾部本来就存在的 `ANDROID_KABI_RESERVE` padding 槽**，这样布局和 CRC 都不变。三个补丁文件就是把字段放进不同 padding 槽的版本：

```
Documentation/resources/kernel-patches/GKI/below-kernel-6.12/
├── 001.GKI-below-6.12-fix_sysvipc_kabi_6_7_8.patch   ← 先试这个（默认 678）
├── 001.GKI-below-6.12-fix_sysvipc_kabi_1_2_3.patch   ← 不行再试
├── 001.GKI-below-6.12-fix_sysvipc_kabi_3_4_5.patch   ← 再不行试
└── 002.5.10_or_lower_use_android_abi_padding_for_posix_mqueue.patch  ← 5.10 及以下才要，6.6 不需要
```

工作流 `droidspaces=678/345/off` 就是槽位三选一；**`123` 在 android15-6.6 上不可用**。

> 本树实测（在真实 6.6.118 `include/linux/sched.h` 上 `patch -p1 --dry-run`）：
> - `678`：干净应用（hunk1 offset -1，hunk2 offset 11 / fuzz 2），因为 task_struct KABI 区是 `ANDROID_KABI_USE(1, dmabuf_info)` + `RESERVE(2)..RESERVE(8)`，6/7/8 空闲；
> - `345`：也能打上（同样靠 fuzz），槽 3/4/5 空闲，可作回退；
> - `123`：**hunk #2 FAILED**（它期望上下文是 `RESERVE(1)`，实际第 1 槽已被 `dmabuf_info` 占用）。
>
> 危险点：`build.yml` 在 Droidspaces 补丁打失败时**只打 warning 然后继续编译**。所以如果选 123，会变成一个“开了 SYSVIPC 但没打 padding”的内核 —— task_struct 布局变化、符号 CRC 变化，刷进去大概率 bootloop。**千万别选 123**（本工作流已把它从选项里删掉）。

**不要**开 `CONFIG_CFS_BANDWIDTH` / `CONFIG_CGROUP_PIDS`：它们会动调度器/cgroup 结构，连 padding 补丁都救不了（工作流没开，保持即可）。

### 4.2 模块 vermagic：版本串不是关键，CRC 才是

网上常说“自编译内核版本串必须和原厂一致，否则模块加载失败”。这在你们的机器上**不是**准确说法：

- 设备 `CONFIG_MODVERSIONS=y`，内核 `kernel/module/version.c` 里的 `same_magic()` 在模块带 CRC 时**会跳过 vermagic 的版本前半段**，只比较 `SMP preempt mod_unload modversions aarch64` 这段。
- 所以 `uname -r` 不同不会导致原厂模块拒载；真正会导致拒载的是**符号 CRC 变化**（ABI 布局变了），这正是 4.1 里 kABI padding 补丁要解决的问题。
- `version` 输入的作用主要是**伪装/隐藏**（让 `uname -r` 看起来就是原厂），不是开机必需。默认帮你填成原厂串，不想要就清空。

### 4.3 四个已开启的开关（ZRAM / BBG / 网络增强 / CVE）

- **`use_zram`（ZRAM 增强）**：把 ZRAM 改成 builtin，替换 LZ4/LZ4HC、加 LZ4KD/LZ4K 算法和 zram 补丁，提升内存压缩/交换性能。注意原厂是 `zram=m`（system_dlkm 里的 `zram.ko`），改成 builtin 后系统再 modprobe 原厂 `zram.ko` 会报错（一般只是日志里一行失败，不影响使用，但这是四项里改动最大的一项）。若开机异常/卡顿/重启，**第一个关它**重编。
- **`use_bbg`（BBG 防格机）**：Baseband-guard LSM，阻止从 `su` 启动的进程对非白名单块分区做写入/改属性/破坏性 ioctl，保护 modem、基带、bootloader 这类分区不被恶意脚本抹掉（真正的“格机”场景）。ABI 安全（LSM blob 动态分配，不改结构体布局），上游 Release `v2.3.0-r10` 也是带 BBG 构建的。详见 4.4。
- **`use_networking`（网络增强）**：BBRv3、IPSet、FQ/FQ_CODEL/CAKE qdisc、connmark、TTL/HL、WireGuard、CIFS。提升网络吞吐/延迟，并让容器/Docker 的 netfilter/IPSet 能力更完整。上游 Release 就是开启状态，编译可行；若网络/耗电异常可关闭排查。
- **`cve_2026_43499_patch`（GhostLock 修复）**：CVE-2026-43499 + CVE-2026-53163 的 rtmutex 修复链；6.6.118 低于 6.6.144，需要打。只改 rtmutex 内部实现，不影响 ABI。若编译报补丁失败就关掉。
- `droidspaces_ntsync` 保持默认关：补丁已验证能在 6.6.118 上干净应用，等容器跑通后再单独开。

> 出问题时的排查顺序（一次只改一个）：先关 `use_zram` → 再关 `use_networking` → 再关 `use_bbg` → 最后关 `cve_2026_43499_patch`。

### 4.4 BBG 防格机到底防什么

BBG（[vc-teahouse/Baseband-guard](https://github.com/vc-teahouse/Baseband-guard)）是内核 LSM，在文件写路径上拦截：

- 从 `su` 启动的进程（untrusted）对**非白名单分区**的 `write`、`setattr`（chmod/chown 等）、破坏性 `ioctl`（`BLKDISCARD`/`BLKZEROOUT`/`BLKPG`/`BLKRRPART` 等）一律返回 `-EPERM`；
- 白名单默认包含 `boot`、`init_boot`、`vendor_boot`、`vendor_kernel_boot`、`dtbo`、`userdata`、`cache`、`metadata`、`misc`、`vbmeta*`、`recovery`，所以**正常刷内核/刷机不受影响**；
- 也就是说它保护的是 **modem / fsc / fsg / persist / xbl / abl** 等基带和 bootloader 分区不被抹掉，**不保护 `boot` 分区**（防刷坏 boot 还是要靠备份 + 回滚）。

可选加强（工作流没开）：`CONFIG_BBG_BLOCK_BOOT` / `CONFIG_BBG_BLOCK_RECOVERY` 会把 boot/recovery 也纳入保护，但这样在 Android 里就无法用 App 刷内核，只能 fastboot，一般不建议开。

验证是否生效：`adb shell dmesg | findstr baseband_guard` 能看到 `baseband_guard ... version` 之类的行；root 下尝试 `dd if=/dev/zero of=/dev/block/by-name/modem` 应被拒绝（**别真写**，只在确认要测试时用）。

---

## 5. 怎么构建

1. 仓库：你的 fork `Jgckm/GKI_KernelSU_SUSFS`（默认分支 `dev`，Actions 已确认处于 enabled）。先点 **Sync fork** 同步上游；本次新增的工作流已提交（见第 12 节）。
2. 打开 **Actions → 内核构建 - 小米15Pro (6.6.118) → Run workflow**，参数按默认即可（默认：`droidspaces=678`、`use_zram=true`、`use_bbg=true`、`use_networking=true`、`cve=true`）：
   - 如果只想先验证“这台机器到底能不能刷开机”，可以把 `droidspaces` 改成 `off`（纯 SukiSU+SUSFS），并暂时关掉 `use_zram`/`use_networking`，用最小内核测试；确认能开机后再开回来。
3. 等构建完成（单 job，约 30–60 分钟；失败会有 `*-Build-Logs` artifact，含编译日志）。
4. 下载 artifacts：
   - `SukiSU_kernel-android15-6.6-118`：内含 `android15-6.6.118-2026-01-AnyKernel3.zip`、`android15-6.6.118-2026-01-boot.img`、`boot-gz.img`、`boot-lz4.img`、`Image`。
   - `SukiSU-Manager(<版本>)`：管理器 APK。
   - `susfs-<版本>`：SUSFS 模块 zip（`ksu_module_susfs.zip`）。

> 想先白嫖验证的话：上游 Release [v2.3.0-r10](https://github.com/zzh20188/GKI_KernelSU_SUSFS/releases/tag/v2.3.0-r10) 里有 `android15-6.6.118-2026-01-boot.img` / `AnyKernel3.zip`（SukiSU+SUSFS，无 Droidspaces）。仅当你想先测试 OS4 到底能不能启动时用；Droidspaces 版仍需自己构建。

---

## 6. 刷机前必须做的两件事

### 6.1 确认 bootloader 真解锁

见第 1 节命令。没解锁就别继续。

### 6.2 备份原厂 boot.img（本机型的硬门槛）

当前系统 `OS4.0.0.19` 公开镜像站暂时没有对应的 fastboot 线刷包，只有更早的 recovery ROM；设备也没 root 不能 `dd`。所以：

- **方案 A**：拿 `OS4.0.0.19.XOBCNXM` 的完整 OTA/卡刷包（官方 OTA 直链格式：
  `https://bkt-sgp-miui-ota-update-alisgp.oss-ap-southeast-1.aliyuncs.com/<版本>/haotian-ota_full-<版本>-user-17.0-<hash>.zip`），用 `payload-dumper-go` 提取 `boot.img`（连同 `init_boot.img`、`vendor_boot.img` 一起留档）：
  ```powershell
  payload-dumper-go.exe -p boot -o .\payload_out haotian-ota_full-OS4.0.0.19.XOBCNXM-*.zip
  Get-FileHash .\payload_out\boot.img -Algorithm SHA256
  ```
- **方案 B**：先用不改 boot 的 LKM root 方案临时拿到 root，再 `dd` 出 `boot_a`：
  ```powershell
  adb shell su -c "dd if=/dev/block/by-name/boot_a of=/sdcard/boot_a_stock.img bs=1M"
  adb pull /sdcard/boot_a_stock.img .
  ```
- **方案 C**：如果以后出 fastboot 线刷包，直接用包里的 `boot.img`。

拿不到同版本 `boot.img` 就**不要刷**（刷坏了只能整包刷机，可能丢数据/触发防回滚）。

---

## 7. 刷入方案（风险从低到高）

> 只动 `boot` 分区，**不要**刷 `init_boot` / `vendor_boot` / `vendor_dlkm` / `system_dlkm`。
> 只刷活动槽 `boot_a`，`boot_b` 保持原厂作退路。

### 方案 A（推荐）：原厂 boot 里换内核，先 `fastboot boot` 试启动

```powershell
# 1) 解包原厂 boot（AOSP mkbootimg 工具，Windows 需要 Python）
git clone --depth 1 https://android.googlesource.com/platform/system/tools/mkbootimg
python .\mkbootimg\unpack_bootimg.py --boot_img .\boot_stock.img --out .\unpacked

# 2) 用新内核替换（文件名以 unpack 输出为准，通常是 kernel）
Copy-Item .\Image .\unpacked\kernel -Force

# 3) 按原 header 版本重打包（本机型 boot 无 ramdisk；若 unpack 有 ramdisk，用 --ramdisk 带上）
python .\mkbootimg\mkbootimg.py --header_version 4 --kernel .\unpacked\kernel --output .\new-boot.img

# 4) 先临时启动，不落盘（小米可能禁用该命令；失败则直接看第 5 步，但必须已备份）
fastboot boot .\new-boot.img

# 5) 确认能进系统后再刷活动槽
fastboot flash boot_a .\new-boot.img
fastboot reboot
```

> 用原厂 boot 重打包的好处：保留原厂 AVB footer/头字段，避免“测试私钥签名”带来的校验问题。
> `fastboot boot` 在 haotian 上是否可用未验证；不可用就只做第 5 步。

### 方案 B：直接刷工作流产出的 boot.img

```powershell
fastboot flash boot_a .\android15-6.6.118-2026-01-boot.img
```

工作流的 boot.img 是 `mkbootimg --header_version 4` + **每次构建临时生成的测试密钥** AVB 签名，与原厂签名不同；解锁状态下一般能刷，但 AVB 校验比方案 A 更容易出问题。**优先方案 A**。

### 方案 C：有 root 之后日常升级 —— Kernel Flasher 刷 AK3 zip

第一次从原厂状态做不了（需要已有 root）；以后升级可用：
```powershell
adb push .\android15-6.6.118-2026-01-AnyKernel3.zip /sdcard/Download/
# 手机上用 Kernel Flasher 选择该 zip 刷入（只写当前槽）
```

### AVB 兜底（只在确认是 AVB 拦了才做）

```powershell
# 用同版本固件包里的原厂 vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta .\vbmeta.img
# 恢复：fastboot flash vbmeta .\vbmeta.img
```
> 解锁状态下通常不需要；做了会降低安全等级，且以后 OTA/回锁可能出问题。

---

## 8. 刷后验证

```powershell
adb shell uname -r                    # 期望与 version 输入一致（默认等于原厂串）
adb shell getprop ro.boot.slot_suffix # 期望 _a
adb shell "cat /proc/version"
```

1. 安装 `SukiSU-Manager(...)` APK，打开应显示已安装/工作中（版本与内核匹配）。
2. 管理器里安装 `ksu_module_susfs.zip`（SUSFS 模块）→ 重启 → 管理器里确认 SUSFS 已启用。
3. 装 [Droidspaces APK](https://github.com/ravindu644/Droidspaces-OSS/releases/latest) → 授权 root → 装后端 → 重启 →
   打开 App **Settings → Requirements → Check Requirements**（或终端 `droidspaces check`），"must have"/"recommended" 应通过。
   - 若卡开机/bootloop：换 `droidspaces=345` 重新构建再试（**不要选 123**，原因见 4.1）。
4. 检查原厂模块是否正常（重点：显示/触摸/WiFi/相机/通话）：这些驱动都在 `vendor_dlkm`，如果它们加载失败通常会表现为开机卡界面或硬件不可用。

---

## 9. 回滚

```powershell
adb reboot bootloader
fastboot getvar current-slot
fastboot flash boot_a .\boot_stock.img     # 先只回滚活动槽
fastboot reboot
# 不行再考虑 boot_b；之前若关过 AVB，恢复 vbmeta：
fastboot flash vbmeta .\vbmeta_stock.img
```
- **不要**从 OS4 整包降级到 OS3（防回滚 ARB 风险）。
- 恢复后按仓库要求：先用原厂镜像救回，再收集 KernelSU 日志反馈（附机型/系统/内核版本/刷入文件名）。

---

## 10. 已知风险与建议

| 风险 | 可能性 | 说明 |
|---|---|---|
| **HyperOS 4 上自定义 GKI 直接不开机** | **高** | 同机型 issue #251 实测失败；Android 17 的红米 K90 同样失败；暂无成功案例 |
| 拿不到 OS4.0.0.19 原厂 boot.img | 中高 | 公开站无该版本线刷包；必须自己从 OTA 提取或临时 root dd |
| bootloader 实为锁定 | 中 | prop 自相矛盾，必须 fastboot 实测 |
| Droidspaces kABI 补丁槽不匹配 → 模块拒载 bootloop | 中 | 678 不行换 345（**不要 123**）；首刷可先用 `off` 验证 |
| ZRAM 改 builtin 后，系统再加载原厂 `zram.ko` 报错 | 中 | 一般只在日志报错、不影响使用；若开机异常/卡顿先关 `use_zram` |
| BBG 误伤正常写入（仅拦截 su 进程对非白名单分区） | 低 | `dmesg | findstr baseband_guard` 可看到 deny 详情；异常就关 `use_bbg` |
| 网络增强补丁导致网络/耗电异常 | 低 | 关 `use_networking` 重编排查 |
| AVB 校验导致不启动 | 中 | 优先方案 A（原厂 boot 重打包）；必要时关 AVB |
| OTA 覆盖内核丢 root | 高（长期） | A/B OTA 会写另一槽，升级后需重刷 |

如果最终确认 OS4 无法跑自定义 GKI：
- 关注本项目和上游 Droidspaces/其他内核作者何时适配 Android 17；
- 或评估**不改 boot 的 LKM root 方案**（KernelSU 官方 LKM / `ksud late-load`）。该路线在 haotian + OS4 上未被公开验证，只是方向；
- 在此之前不要把主力机当实验品。

---

## 11. 可选增强 / 后续可调项

- **伪装 `/proc/config.gz`**：把本机提取的原厂 config 放到 `config/stock_defconfig` 即可自动生效（构建时会把它作为 `/proc/config.gz` 的内容）：
  ```powershell
  Copy-Item .\device\stock_config.txt .\config\stock_defconfig
  ```
  不需要就删除该文件。
- **内核版本伪装**：工作流 `version` 输入默认已伪装成原厂串；不需要就清空。
- **ZRAM 增强 / BBG 防格机 / 网络增强 / CVE 修复**：工作流默认已全部开启；如果开机异常，按 4.3 的顺序逐个关闭排查（改动最大的是 ZRAM）。
- **NTSync**：Droidspaces 可选增强；补丁已在 6.6.118 上 dry-run 通过，可打开；若编译报 API 缺失就关掉（不影响 SYSVIPC 容器功能）。
- **IPv6 NAT / macvlan**：上游 GKI-safe 清单里还有 3 个可选项工作流没加 —— `CONFIG_IP6_NF_NAT`、`CONFIG_IP6_NF_TARGET_MASQUERADE`（NAT66）、`CONFIG_MACVLAN`（macvlan 网络模式）。需要的话可自行在 `build.yml` 的 Droidspaces 块补三行 `enable_config_if_defined`（与 kABI 无关）；不需要就保持现状。

---

## 12. 仓库改动清单

| 文件 | 说明 |
|---|---|
| `.github/workflows/kernel-haotian-6-6-118.yml` | **新增**：只构建 android15-6.6.118 单 job 的专用工作流（SukiSU + SUSFS + Droidspaces，参数可调） |
| `docs/xiaomi15pro-haotian-build-guide.md` | **新增**：本文档 |

不需要的工作流/文件可以随时删除或 `git revert` 本次提交；`build.yml` 等上游文件未改。同步上游（Sync fork）不会丢这个工作流，除非你强制 reset。

---

## 13. 参考链接

- 项目 README / 文档：https://github.com/zzh20188/GKI_KernelSU_SUSFS
- 进阶功能（含 Droidspaces 说明）：[docs/advanced-features.md](advanced-features.md)
- 项目 Wiki（救砖 / 选择版本 / 其他技巧）：https://github.com/zzh20188/GKI_KernelSU_SUSFS/wiki
- 本次同类机型失败案例：[issue #251](https://github.com/zzh20188/GKI_KernelSU_SUSFS/issues/251)、[#252](https://github.com/zzh20188/GKI_KernelSU_SUSFS/issues/252)
- Droidspaces 上游：https://github.com/ravindu644/Droidspaces-OSS
- Droidspaces 内核配置文档：https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md
- 设备信息/刷入/回滚证据：`reports/device-probe.md`、`reports/compat-evidence.md`、`reports/patch-kmi.md`

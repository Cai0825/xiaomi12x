# Droidspaces Kernel Builder - Xiaomi 12X (psyche) / PixelOS 17

这个目录用于为当前这台 Xiaomi 12X 构建带 Droidspaces 必需内核能力、同时保留现有 Magisk root 的 AnyKernel3 卡刷包。

## 已核实的设备信息

```text
设备:           Xiaomi 12X (psyche, 2112123AC)
系统:           PixelOS_psyche-17.0-20260904-1150
内核:           4.19.325-cip135-st19-perf-g59aa6cf45839-dirty
当前槽位:       _b
当前 root:      Magisk 31.0
已备份 boot_b:  /sdcard/boot_b-droidspaces-backup-20261023.img
备份 SHA-256:   4fb4419a3d67816fd6dea765f76de5a7e5c18a7f3ed5ffc67324998986bba3f8
```

当前内核配置明确关闭了以下能力：

```text
# CONFIG_PID_NS is not set
# CONFIG_SYSVIPC is not set
# CONFIG_POSIX_MQUEUE is not set
```

因此 `IPC_NS` 也无法启用，Droidspaces 的检查必然失败。

## 对应的内核源码

```text
Rocky7842/android_kernel_xiaomi_sm8250
commit 59aa6cf45839028d7a18f842e0941ec416ca1a00
```

该提交与设备 `/proc/version` 中的 `g59aa6cf45839` 一致。源码树是 Linux 4.19 非 GKI，不需要 GKI kABI 补丁。

## 构建内容

配置片段只打开 Droidspaces 必需项和 `devtmpfs`：

```text
CONFIG_NAMESPACES=y
CONFIG_UTS_NS=y
CONFIG_PID_NS=y
CONFIG_SYSVIPC=y
CONFIG_POSIX_MQUEUE=y
CONFIG_IPC_NS=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_CGROUPS=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
```

同时固定与现有厂商模块完全一致的 vermagic：

```text
CONFIG_LOCALVERSION="-perf-g59aa6cf45839-dirty"
# CONFIG_LOCALVERSION_AUTO is not set
```

最终内核名必须仍为：

```text
4.19.325-cip135-st19-perf-g59aa6cf45839-dirty
```

这用于保证现有 `rmnet_perf.ko` 和 `rmnet_shs.ko` 仍能加载。

内核配置变化后，vendor 中的预编译模块仍可能因 `module_layout` CRC 不同而拒绝加载。因此构建包还会带上同一次编译生成的 `rmnet_perf.ko`、`rmnet_shs.ko` 和 `gspca_main.ko`，通过 AnyKernel3 的 `ak3-helper` Magisk 模块进行 systemless 覆盖。

首次构建刻意不启用：

```text
CONFIG_USER_NS
CONFIG_CGROUP_PIDS
CONFIG_CFS_BANDWIDTH
```

这些是可选项，部分会改变内核核心结构布局。当前 ROM 仍有 `rmnet_perf.ko` 和 `rmnet_shs.ko` 厂商模块，所以先做最小改动可以降低模块 CRC 不匹配的风险。

构建流程会：

1. 按 SHA 下载精确的内核源码。
2. 应用 Droidspaces 官方 non-GKI 补丁。
3. 按设备树的真实顺序合并 defconfig。
4. 运行 `olddefconfig`，逐项验证必需配置。
5. 编译 `Image`，只替换 boot 中的内核。
6. 保留原 ROM 的 ramdisk、dtb、dtbo、vendor_boot 和 vendor 分区。

## 使用 GitHub Actions 构建

1. 在 GitHub 新建一个空仓库，不要添加 README 或模板。
2. 将本目录全部内容推送到仓库。

```bash
git init
git add -A
git commit -m "Build Droidspaces kernel for PixelOS psyche"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

3. 打开仓库的 **Actions**，选择 **Build Droidspaces Kernel (Xiaomi 12X / psyche)**。
4. 点击 **Run workflow**，保持默认参数即可：

```text
kernel_repo = Rocky7842/android_kernel_xiaomi_sm8250
kernel_ref  = 59aa6cf45839028d7a18f842e0941ec416ca1a00
publish_release = true
```

5. 构建通常需要 30 至 60 分钟。成功后下载：

```text
Droidspaces-psyche-59aa6cf45839.zip
```

## 刷入

使用支持 AnyKernel3 的卡刷工具，例如 KernelFlasher 或 EX Kernel Manager。确保工具识别到的设备是 `psyche`。

AnyKernel3 会保留当前 Magisk 修补过的 ramdisk，因此正常情况下 root 不会丢失。

## 刷入后验证

```sh
adb shell su -c 'ls -l /proc/self/ns/'
```

应能看到：

```text
mnt
uts
ipc
net
pid
```

还可以检查配置：

```sh
adb shell 'zcat /proc/config.gz | grep -E "CONFIG_(PID_NS|IPC_NS|SYSVIPC|POSIX_MQUEUE|DEVTMPFS)="'
```

最后重新运行 Droidspaces 的 **Settings > Requirements > Check Requirements**。最先需要保证两项：

```text
PID namespace
IPC namespace
```

其他可选项暂时显示为未启用不会阻止容器启动。

## 回滚

`sdcard` 中已保存当前 boot_b：

```text
/sdcard/boot_b-droidspaces-backup-20261023.img
```

如果刷入后无法开机，在 fastboot 中执行：

```bash
fastboot flash boot_b boot_b-droidspaces-backup-20261023.img
fastboot reboot
```

不要在当前状态下直接锁定 Bootloader。

## 已知限制

Linux 4.19 内核缺少 `openat2`，因此 systemd 258 及更新版本的发行版可能无法启动。建议先使用 Ubuntu 22.04/24.04、Debian 12/13、Alpine、Void 或其他仍使用较旧 systemd 的系统。

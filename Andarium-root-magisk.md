# Andarium 获取 Root 权限（Magisk 方案）完整指南

> 本文记录一套**可复现**的流程：从应用商店安装 Andarium → 脱壳取出 `system.img` → 解包分区 → Magisk 修补 `boot` → 重打包 → 封回 IPA → 安装已 Root 的 Andarium。
>
> **当前状态：已验证成功。** 按本文流程走完后，Andarium 内 `su` 可用，Magisk 正常挂载。

---

## 目录

- [0. 原理速览](#0-原理速览)
- [1. 前置准备](#1-前置准备)
- [2. 第一步：从应用商店下载 Andarium](#2-第一步从应用商店下载-andarium)
- [3. 第二步：脱壳，取出 system.img](#3-第二步脱壳取出-systemimg)
- [4. 第三步：解包 system.img，拿到分区文件](#4-第三步解包-systemimg拿到分区文件)
- [5. 第四步：回手机，用 Magisk 修补 boot](#5-第四步回手机用-magisk-修补-boot)
- [6. 第五步：回电脑，重打包成新的 system.img](#6-第五步回电脑重打包成新的-systemimg)
- [7. 第六步：封装回 IPA](#7-第六步封装回-ipa)
- [8. 第七步：安装修改后的 IPA，验证 Root](#8-第七步安装修改后的-ipa验证-root)
- [9. 原理说明：为什么改 boot 就能拿到 root](#9-原理说明为什么改-boot-就能拿到-root)
- [10. 常见问题与排错](#10-常见问题与排错)
- [11. 风险与合规提醒](#11-风险与合规提醒)
- [12. 附录 A：完整命令速查](#12-附录-a完整命令速查)
- [13. 附录 B：工具清单](#13-附录-b工具清单)
- [14. 附录 C：分区文件清单](#14-附录-c分区文件清单)

---

## 0. 原理速览

Andarium 在设备上运行的是一套**轻量虚拟化 / 容器化环境**：它的"系统"以镜像文件的形式随 IPA 一起分发，运行时被挂载当作 guest 的根文件系统。也就是说：

```
Andarium.ipa
└── Payload/Andarium.app/
    └── andarium-system/
        └── guest/
            └── system.img     ← 这就是"手机里的手机系统盘"
```

其中 `system.img` 是一个**多分区镜像**，内部包含 `boot`、`system`、`vendor` 等分区。而 **`boot` 分区里装着 Linux 内核 + 初始 ramdisk（initrd）**。

Root 的关键点在于：**Magisk 的工作原理就是"修补 boot 里的 ramdisk"**——往 ramdisk 塞入 Magisk 的 `init` 和 `magiskinit`，让系统在启动早期先执行 Magisk 的代码，从而建立 root 环境（`su`、magisk daemon、模块挂载等），然后再 `switch_root` 到真正的系统。

所以整套流程可以浓缩成一句：

> **把 boot 抠出来 → 让 Magisk 打补丁 → 塞回去 → 重新装包。**

这就是为什么不需要破解签名逻辑、不需要 hook 进程，只是"换了个会更早接管启动的 init"。

---

## 1. 前置准备

### 1.1 环境

| 用途 | 推荐环境 | 说明 |
| --- | --- | --- |
| 脱壳、解包、重打包 | Linux（Ubuntu 22.04+）或 WSL2 | 需要 root 权限执行 `mount` 等操作 |
| Magisk 修补 boot | Android 手机（真机） | 用 Magisk App 的"安装 → 选择并修补一个文件" |
| 解包 / 封包 IPA | macOS / Linux | IPA 本质是 zip，Linux 上也完全能做 |
| 安装 IPA | 越狱设备，或 TrollStore / 自签工具 | 取决于你的设备情况 |

> 建议全程在同一个工作目录里操作，所有产物都留一份原始备份。

### 1.2 在手上准备好这些工具

```bash
# Linux 侧依赖
sudo apt update
sudo apt install -y \
    android-sdk-libsparse-utils \   # simg2img / img2simg
    android-sdk-ext4-utils \        # make_ext4fs
    e2fsprogs \                     # resize2fs / debugfs / fsck.ext4
    cpio \
    lz4 lzop \
    python3 python3-pip \
    unzip zip \
    file xxd
```

> 如果发行版没有 `android-sdk-ext4-utils` 包，可以直接用 AOSP 里编好的 `make_ext4fs` 二进制，或者改用 `mkfs.ext4` + `e2fsck -f` 的方式。

### 1.3 需要准备的文件

- 一份**原始的、未修改的** `Andarium.ipa`
- Andarium 的 `.app` 目录（可以先把 IPA 当 zip 解开备份一份）
- 一份**原始的 `system.img` 备份**（重要！出错时要靠它回滚）

### 1.4 强烈建议：先做全量备份

```bash
mkdir -p ~/andarium-work/{original,extract,repack,output}
cp /path/to/andarium-system.img ~/andarium-work/original/system.img.bak
sha256sum ~/andarium-work/original/system.img.bak | tee ~/andarium-work/original/SHA256
```

记住：**整个过程里唯一不可再生的东西就是"原始镜像"**，其它都能重做。

---

## 2. 第一步：从应用商店下载 Andarium

1. 打开设备的应用商店，搜索 **Andarium**，正常安装最新版。
2. 首次启动一次，让它把运行环境初始化出来（有些版本会在首次启动时才把 `andarium-system/` 铺到数据目录，这一步能帮你确认路径到底在哪）。
3. 退出 App。

### 2.1 定位 App 安装包与数据目录

iOS 上 App 的 Bundle（只读的 `.app`）和数据（可写的 `Documents`、`Library`）是分开的：

```
# Bundle（安装包本体，system.img 通常在这里）
/var/containers/Bundle/Application/<APP-UUID>/Andarium.app/andarium-system/guest/system.img

# 数据目录（运行时产物）
/var/mobile/Containers/Data/Application/<DATA-UUID>/...
```

用 Shell / Filza / 文件管理工具找到这个 `system.img`，记下它的体积：

```bash
ls -lh /var/containers/Bundle/Application/*/Andarium.app/andarium-system/guest/
```

正常应该能看到类似：

```
-rw-r--r--  1 mobile mobile  3.2G  system.img
```

> **注意**：如果你手上拿到的不是 `.img` 而是 `.dmg`、`.qcow2` 之类，说明版本差异较大，思路一致（都是拿到 guest 的块设备镜像），但解包命令需要相应调整。本文以 `system.img` 为准。

---

## 3. 第二步：脱壳，取出 system.img

"脱壳"在这里的重点不是破解加密（Andarium 的镜像本身没有商业级加密），而是**把商店安装出来的、带数字签名的 IPA 还原成"可以重新打包的普通目录结构"**，同时拿到镜像文件。

### 3.1 最直接的办法：直接从安装目录拷贝

如果是越狱设备 / 有 TrollStore，最简单：

```bash
# 在设备上（通过 SSH 或本地 shell）
cp -a /var/containers/Bundle/Application/*/Andarium.app/andarium-system/guest/system.img \
      /var/mobile/Documents/system.img
```

然后通过 Filza / SCP / AirDrop 传到电脑。

这一步成功的话，**`3.1` 就已经足够**，后面 3.2 只是补充手段。

### 3.2 补充：从 IPA 直接从压缩包里抽出来

如果你手上有 IPA 文件（或者你从商店导出过），不需要解整个包：

```bash
# 只看里面有什么，不落盘
unzip -l Andarium.ipa | grep -i 'system.img\|guest'

# 单独抽出 system.img，保持原始目录结构
mkdir -p ~/andarium-work/extract/ipa
unzip Andarium.ipa \
      'Payload/Andarium.app/andarium-system/guest/system.img' \
      -d ~/andarium-work/extract/ipa

ls -lh ~/andarium-work/extract/ipa/Payload/Andarium.app/andarium-system/guest/
```

> IPA 的 `Payload/*.app/` 目录结构必须**原样保留**，封包时一个字节的结构差异都可能导致签名/安装失败，所以这里刻意不手动重建目录。

### 3.3 校验镜像类型

拿到文件后，先确认它是 **sparse image** 还是 **raw image**：

```bash
cd ~/andarium-work/extract
file system.img
```

常见结果：

| 输出 | 含义 | 处理 |
| --- | --- | --- |
| `Android sparse image` | Android sparse 格式 | 需要先 `simg2img` 转 raw |
| `Linux rev 1.0 ext4 filesystem data` | 已是 ext4 raw 镜像 | 可直接挂载 |
| `data`（无识别） | 可能是自定义头 / 分区表 | 用 `xxd` 看前 512 字节再判断 |

```bash
xxd -l 512 system.img
```

判断技巧：

- 开头出现 `3aff 26ed` → Android sparse 的 magic（`ED 26 FF 3A`），走 `simg2img`。
- 开头 512 字节里出现 `EFI PART` 或 MBR 签名 `55aa` 在末尾 → 带分区表，需要 `parted` / `fdisk -l` 看分区。
- 直接是 ext4 的超级块特征（偏移 `0x438` 附近有 `53ef`）→ 单分区 raw 镜像。

---

## 4. 第三步：解包 system.img，拿到分区文件

### 4.1 sparse → raw

```bash
cd ~/andarium-work/extract
simg2img system.img system.raw.img
rm -f system.img && mv system.raw.img system.img
file system.img    # 现在应该是 ext4 / DOS/MBR
```

### 4.2 情况 A：镜像本身就是一个 ext4 文件系统（最常见）

```bash
sudo mkdir -p /mnt/andarium
sudo mount -o loop,rw system.img /mnt/andarium
ls /mnt/andarium
```

典型内容：

```
bin  boot  dev  etc  home  lib  lib64  mnt  proc  root  sbin  sys  tmp  usr  var
```

这种情况下，`boot` 就是一个**普通目录**（里面是内核 `vmlinuz`/`Image` + `initrd.img`），非常友善。

### 4.3 情况 B：镜像带分区表（多分区）

```bash
sudo fdisk -l system.img
```

输出类似：

```
Device       Boot  Start       End   Sectors   Size Id Type
system.img1         2048    614399    612352   299M 83 Linux
system.img2       614400  12582911  11968512   5.7G 83 Linux
```

用 loop 的 `offset` 单独挂载每个分区：

```bash
# 计算字节偏移：Start × 512
sudo mount -o loop,ro,offset=$((2048*512))    system.img /mnt/part1
sudo mount -o loop,ro,offset=$((614400*512))  system.img /mnt/part2
```

或者用 `losetup -P` 让内核自动扫描分区：

```bash
sudo losetup -fP --show system.img
# 输出 /dev/loop0，之后即可用 /dev/loop0p1、/dev/loop0p2 …
lsblk /dev/loop0
```

### 4.4 你实际会拿到哪些文件

按你跑通的这套流程，解包后从镜像里**带出来的是 6 个分区相关文件**：`boot` 分区本体 + 另外 5 个配套分区（如 `system`、`vendor`、`recovery`、`dtbo`、`vbmeta` 之类，具体名字以你的镜像为准）。

把它们整组拷到电脑上同一目录，**保持"同源同版本"**——不同分区来自同一个镜像，这一点在重打包时很重要：

```bash
mkdir -p ~/andarium-work/extract/partitions
sudo cp -a /mnt/andarium/boot*       ~/andarium-work/extract/partitions/
sudo cp -a /mnt/andarium/*.img       ~/andarium-work/extract/partitions/   # 视结构而定
sudo chown -R "$USER":"$USER" ~/andarium-work/extract/partitions

cd ~/andarium-work/extract/partitions
sha256sum * > SHA256SUMS.orig
ls -lh
```

> **关键提醒**：这 6 个文件是"一套"。**修补只动 `boot`，其余 5 个保持原样**，重打包时原封不动放回去即可。

### 4.5 卸载

```bash
sudo umount /mnt/andarium
# 若用过 losetup -P：
sudo losetup -d /dev/loop0
```

---

## 5. 第四步：回手机，用 Magisk 修补 boot

这一步必须在 **Android 手机**上做，因为要借助 Magisk App 内置的 `magiskboot` 来正确解析并重建 boot 镜像。

### 5.1 安装 Magisk App

从 Magisk 官方 GitHub Releases 下载最新稳定版 APK 安装。

### 5.2 把 boot 分区文件传到手机

```bash
# 电脑 → 手机（示例用 adb，也可以用网盘/文件传输）
adb push ~/andarium-work/extract/partitions/boot_a.img /sdcard/Download/
```

> 注意：**Magisk 要求输入的是 `boot.img` 本身**（不是 `.gz`、不是解出来的内核），文件名不必叫 `boot.img`，但内容必须是完整 boot 镜像（含 header + kernel + ramdisk）。

### 5.3 在 Magisk 里修补

1. 打开 Magisk App。
2. 首页 **Magisk** 卡片 → **安装**。
3. 选择 **"选择并修补一个文件"**（Select and Patch a File）。
4. 选中刚才的 `boot_a.img`。
5. 等待输出 `All done!`。
6. 修补产物一般位于：

```
/sdcard/Download/magisk_patched-<版本号>_<随机串>.img
```

### 5.4 把修补结果传回电脑

```bash
adb pull /sdcard/Download/magisk_patched-27000_xxxxx.img ~/andarium-work/repack/boot_patched.img
ls -lh ~/andarium-work/repack/boot_patched.img
sha256sum ~/andarium-work/repack/boot_patched.img
```

### 5.5 验证修补确实生效

在电脑上粗略检查一下，确认 ramdisk 里多了 Magisk 的东西：

```bash
# 用 magiskboot（可从 Magisk APK 的 lib 里解出对应架构的二进制）
./magiskboot unpack boot_patched.img
ls ramdisk.cpio* 2>/dev/null
./magiskboot cpio ramdisk.cpio ls | grep -i magisk
```

正常情况下能看到：

```
magisk
magiskinit
.magisk/
```

> 如果 `magiskboot` 不方便拿，也可以用一个快速的"体积差"判断：修补后的镜像通常比原来大 1~3 MB（多了 magiskinit 和预置模块）。但**体积只是旁证，务必用 cpio 列表确认**。

---

## 6. 第五步：回电脑，重打包成新的 system.img

### 6.1 把修补后的 boot 放回分区目录

```bash
cd ~/andarium-work/extract/partitions
cp ~/andarium-work/repack/boot_patched.img ./boot_a.img    # 覆盖原 boot
sha256sum -c SHA256SUMS.orig 2>&1 | grep -v 'OK$' || echo "其余分区保持原样 ✅"
```

上面那条会列出**不一致**的项。理想输出只有 `boot_a.img` 一个 `FAILED`，其余全部 `OK`。

### 6.2 重新生成镜像（按情况选择）

#### 情况 A：原来的 `system.img` 是 ext4 单分区（直接把文件换回去）

如果第 4 步是**挂载目录后直接改文件**，那么其实不用"重打包"——你只要：

```bash
# 挂载 → 覆盖 boot 文件 → 卸载 即可
sudo mount -o loop,rw ~/andarium-work/extract/system.img /mnt/andarium
sudo cp ~/andarium-work/repack/boot_patched.img /mnt/andarium/boot/boot.img
sync
sudo umount /mnt/andarium
e2fsck -f -y ~/andarium-work/extract/system.img     # 修一下超级块/日志
```

> 注意 `boot` 在这里是**目录**，里面才有真正的 `boot.img`；替换的是目录里的文件，不是整个镜像里的分区表。

#### 情况 B：需要从分区文件重建原始镜像

```bash
cd ~/andarium-work/repack

# 1) 按 fdisk 记录的原始分区表重建空镜像
truncate -s "$(stat -c %s ~/andarium-work/extract/system.img.orig)" system_new.img
sudo parted -s system_new.img mklabel gpt

# 2) 按原来的 start/size 逐个建分区（示例）
sudo parted -s system_new.img mkpart boot   ext4 1MiB 300MiB
sudo parted -s system_new.img mkpart system ext4 300MiB 100%

# 3) 挂 loop 并写入各区内容
sudo losetup -fP --show system_new.img      # → /dev/loop0
sudo dd if=~/andarium-work/extract/partitions/boot_a.img   of=/dev/loop0p1 bs=4M conv=fsync
sudo dd if=~/andarium-work/extract/partitions/system.img   of=/dev/loop0p2 bs=4M conv=fsync
sudo losetup -d /dev/loop0
```

#### 情况 C：镜像原本是 sparse，需要转回去

如果 Andarium 读取的是 sparse 格式，最后一定要转回去，否则加载会失败：

```bash
img2simg ~/andarium-work/repack/system_new.img \
         ~/andarium-work/output/system.img
file ~/andarium-work/output/system.img    # 应显示 Android sparse image
```

> **格式必须和原来一致**：原来是 sparse 就出 sparse，原来是 raw 就出 raw。这是最容易翻车的一步，务必在第一次动手时记录下 `file` 的原始输出。

### 6.3 完整性自检

```bash
# 挂在只读模式验证能不能正常读
sudo mount -o loop,ro ~/andarium-work/output/system.img /mnt/check
ls /mnt/check
sudo umount /mnt/check

# 查看体积对比
ls -lh ~/andarium-work/original/system.img.bak ~/andarium-work/output/system.img
```

体积应该非常接近（差几百 KB 到几 MB 是正常的，来自 fs 元数据和对齐）。

---

## 7. 第六步：封装回 IPA

IPA 就是一个普通 zip，**结构必须严格是**：

```
Andarium.ipa
└── Payload/
    └── Andarium.app/
        ├── Andarium            (Mach-O 可执行文件)
        ├── Info.plist
        ├── _CodeSignature/     (原签名，会被工具重签覆盖)
        ├── andarium-system/
        │   └── guest/
        │       └── system.img  ← 换上我们的
        └── ...
```

### 7.1 换掉镜像

```bash
cp -a ~/andarium-work/extract/ipa ~/andarium-work/output/ipa-work
cp ~/andarium-work/output/system.img \
   ~/andarium-work/output/ipa-work/Payload/Andarium.app/andarium-system/guest/system.img

ls -lh ~/andarium-work/output/ipa-work/Payload/Andarium.app/andarium-system/guest/
```

### 7.2 打包

```bash
cd ~/andarium-work/output/ipa-work
zip -qr9 ../Andarium-rooted.ipa Payload
cd ..
ls -lh Andarium-rooted.ipa
unzip -l Andarium-rooted.ipa | head -20
```

> 用 `zip -r` 而不是 `ditto` / `tar`。macOS 上也可以：
> `ditto -c -k --sequesterRsrc --keepParent Payload Andarium-rooted.ipa`
> 但 Linux 环境下 `zip` 最稳，注意**不要**给 `Payload` 目录加额外的父目录层级。

---

## 8. 第七步：安装修改后的 IPA，验证 Root

### 8.1 安装

安装方式取决于你的设备：

| 设备情况 | 安装方式 |
| --- | --- |
| 越狱设备 | 直接 `dpkg`/包管理器安装，或 `appinst` |
| 非越狱但支持 TrollStore | TrollStore 打开 `.ipa` → Install |
| 其它 | 用自签工具（AltStore / Sideloadly / TrollStore 类工具）重签后安装 |

> **重要**：IPA 被你改过之后，**原有签名必然失效**。所以安装时必须走"重签"路径，用你自己的证书 / 描述文件，或者在越狱环境下跳过签名校验。不能指望直接 `ideviceinstaller` 装原签名包。

### 8.2 验证 Root

打开 Andarium，进入它的终端 / shell 环境，依次执行：

```bash
# 1) 是否有 su
which su
su --version

# 2) 是否真的能提权
su -c 'id'
# 期望输出：uid=0(root) gid=0(root) ...

# 3) Magisk 是否起来了
su -c 'magisk -v'
su -c 'magisk -V'

# 4) 挂载 / 模块状态
su -c 'magisk --path'
su -c 'ls /data/adb/modules'

# 5) 一个实际的根权限操作
su -c 'mount -o remount,rw /'
su -c 'cat /proc/mounts | head'
```

全部通过 = **成功拿到 root** ✅

---

## 9. 原理说明：为什么改 boot 就能拿到 root

### 9.1 一次正常启动发生了什么

```
固件 / 引导
   ↓
加载 boot.img  → 解压出 kernel + initrd
   ↓
内核启动，挂载 initrd 为临时根
   ↓
执行 initrd 里的 /init  ← 【Magisk 的藏身之处】
   ↓
init 做早期硬件初始化、加载驱动
   ↓
switch_root 到真正的根文件系统（system.img 里的内容）
   ↓
启动用户态服务（你平时看到的 Andarium）
```

### 9.2 Magisk 干了什么

Magisk 的修补本质上是把 ramdisk 变成一个"**二道贩子**"：

```
原 ramdisk 的 init        → 被改名为 /init.orig
新塞进去的 magiskinit     → 抢到 /init 这个名字
```

于是启动流程变成：

```
内核 → 执行 /init（其实是 magiskinit）
     → magiskinit 先解包 /data/adb/magisk 里的 magisk64
     → 建立 root 环境：挂载 magisk 分区、注入 su、启动 magiskd
     → 然后 exec /init.orig，把系统正常拉起来
```

**关键点**：Magisk 在**系统还没接手之前**就已经跑完了。所以它拿到的权限、能挂载的东西，比系统自己启动后能触及的层次更深、更早，因此无法被用户态程序剥夺。

这也解释了为什么：

- **只改系统分区（rootfs）不够** —— 你能改文件，但拿不到比系统更高的权限，还会被完整性校验发现。
- **改 boot 就够了** —— 因为"最初的那个 init"被换掉了，Magisk 抢在系统之前建立特权环境。
- **`vbmeta` / `dm-verity` 需要处理** —— 在带完整的 Android 校验链的镜像里，改 boot 会导致校验失败，通常要一起 patch `vbmeta`（关掉 verity）。Andarium 的镜像如果没开 verity，这一步就可省。

### 9.3 对 Andarium 这种虚拟化方案意味着什么

Andarium 的 guest 环境本质是"一个跑在 App 里的 Linux"。它的 root 语义分两层：

| 层级 | 是什么 | 本文是否涉及 |
| --- | --- | --- |
| **Guest 内 root** | guest Linux 里的 `uid 0` | ✅ 本文拿到的就是这个 |
| **宿主内 root** | iOS 系统层面的 root | ❌ 不涉及，也做不到 |

所以你拿到的 root 是**在 Andarium 这个"手机里的手机"内部的完整 root**——能 `su`、能装 Magisk 模块、能改 guest 的 `/system`。它**不是** iOS 越狱。

> 这一点很重要，避免误解：这是"容器内提权"，不是"突破沙盒"。在 guest 内你是上帝；出到 iOS 层面，你仍然只是一个普通 App。

---

## 10. 常见问题与排错

### Q1：装完启动就闪退 / 卡在启动屏

**原因**：镜像格式或结构不对，尤其是 sparse/raw 搞反了、分区偏移错了、镜像没有对齐。

**排查**：

```bash
# 和原始镜像逐项比对
file ~/andarium-work/original/system.img.bak
file ~/andarium-work/output/system.img
ls -l ~/andarium-work/original/system.img.bak ~/andarium-work/output/system.img

# 尝试只读挂载验证结构
sudo mount -o loop,ro ~/andarium-work/output/system.img /mnt/check
```

| 现象 | 可能原因 | 修复 |
| --- | --- | --- |
| 体积差好几倍 | sparse ↔ raw 搞反 | `simg2img` / `img2simg` 转对 |
| 挂载报 `bad magic number` | 超级块被破坏 | 用备份重新做，检查 `dd` 的 `bs`/`seek` |
| 挂载成功但启动卡住 | 文件权限 / SELinux label 丢 | `cp -a` 保住属性；必要时 `restorecon` |
| 启动后没有 `su` | boot 没换成功 | 用 `magiskboot cpio ls` 确认 |

### Q2：Magisk 修补报错 `Unsupported/Unknown image format`

**原因**：传进 Magisk 的不是完整 boot 镜像（比如只给了内核 `Image`，或给了个 `.img.gz`）。

**做法**：

```bash
file boot_a.img
# 必须是 "Android bootimg, ..." 或 "Linux kernel ARM64 boot executable ..." 之类
# 如果是 gzip/xz/lz4 压缩的，先解压再给 Magisk
```

### Q3：修补成功了，但进系统后 `su` 没反应 / 提示 permission denied

| 检查项 | 命令 | 期望 |
| --- | --- | --- |
| magiskd 有没有跑 | `ps -A \| grep magisk` | 有 `magiskd` |
| magisk 数据目录在不在 | `ls -la /data/adb/magisk` | 有一堆文件 |
| su 是不是 magisk 的 | `ls -l $(which su)` | 指向 `/data/adb/magisk/su` 或 `magisk` 挂载点 |
| 是否被 verity 拦了 | `dmesg \| grep -i verity` | 无 `dm-verity` 错误 |

### Q4：改了镜像但 IPA 装不上

**原因**：签名。IPA 一改就废签，必须重签。

**做法**：

```bash
# 越狱环境：直接装（绕过签名校验）
appinst Andarium-rooted.ipa

# 非越狱：用 TrollStore / 自签工具
# 注意：自签后 30 天左右描述文件会过期，需要重签
```

另外检查一下包结构：

```bash
unzip -l Andarium-rooted.ipa | head -30
# 第一层必须是 Payload/，不能是 Payload/Payload/ 或 Andarium-rooted/Payload/
```

### Q5：guest 里能 `su`，但装 Magisk 模块无效

**原因**：模块目录 `/data/adb/modules` 不可写，或 Magisk 的 mount 阶段没跑到。

```bash
su -c 'ls -ld /data/adb /data/adb/modules'
su -c 'magisk --path'          # 看 magisk 的挂载点
su -c 'mount | grep magisk'
```

如果 `/data/adb` 在 guest 里不存在（因为 guest 的 `/data` 可能没被持久化），需要确认镜像里 `/data` 的挂载方式——这属于镜像本身的布局问题。

### Q6：手机上的 `boot` 分区名不叫 `boot_a.img`

按实际情况改名即可。本文统一用 `boot_a.img` 只是示例。关键是**Magisk 修补的那个文件** = **你重打包时替换的那个分区**，两者必须是同一个分区。

### Q7：能不能只解包、不重打包，直接替换目录里的文件？

**可以，而且更安全。** 如果原始 `system.img` 是 ext4 单分区且能 `rw` 挂载，那第 6 步只需要"挂载 → 换文件 → 卸载 → fsck"，不需要重建分区表。**只要可能，优先走这条路。**

---

## 11. 风险与合规提醒

- **这是对 App 安装包的修改**。修改后的包与原包不再等价，原厂的任何完整性校验、在线校验、防盗版机制都可能因此失效或触发。
- **签名一定失效**。重签必须使用你自己的证书；把修改后的包分发给他人属于二次分发，可能违反原 App 的服务条款。
- **系统稳定性**：boot 是启动链的关键一环，改错了会直接开不了机（在 Andarium 里就是 App 起不来，重装即可恢复，不会影响 iOS 本身）。
- **请仅在你自己的设备、自己的副本上操作**；本文所有步骤的前提是"用户已合法获得该 App 并自主决定修改"。
- 建议保留完整备份链条：`原始IPA → 原始system.img → 原始分区 → 修补后的分区 → 最终IPA`，任何一步出问题都能回到上一步。

---

## 12. 附录 A：完整命令速查

按顺序执行，可直接复制（路径按需改）：

```bash
# ============ 0. 工作区 ============
mkdir -p ~/andarium-work/{original,extract,repack,output}
cd ~/andarium-work

# ============ 1. 备份原始镜像 ============
cp /path/to/system.img original/system.img.bak
sha256sum original/system.img.bak > original/SHA256

# ============ 2. 判断格式 & 转 raw ============
file original/system.img.bak
xxd -l 512 original/system.img.bak | head
# 若为 sparse:
simg2img original/system.img.bak extract/system.img || cp original/system.img.bak extract/system.img

# ============ 3. 解包分区 ============
# a) 单 ext4:
sudo mkdir -p /mnt/andarium
sudo mount -o loop,rw extract/system.img /mnt/andarium
sudo cp -a /mnt/andarium/* extract/partitions/ 2>/dev/null || true

# b) 带分区表:
# sudo fdisk -l extract/system.img
# sudo losetup -fP --show extract/system.img
# sudo cp -a /dev/loop0p* extract/partitions/

sudo umount /mnt/andarium 2>/dev/null
sudo losetup -d /dev/loop0 2>/dev/null

# ============ 4. Magisk 修补（在手机上） ============
#    Magisk App → 安装 → 选择并修补一个文件 → boot 分区
adb push extract/partitions/boot_a.img /sdcard/Download/
#  ... 修补 ...
adb pull /sdcard/Download/magisk_patched-*.img repack/boot_patched.img

# ============ 5. 验证修补 ============
./magiskboot unpack repack/boot_patched.img
./magiskboot cpio ramdisk.cpio ls | grep -i magisk

# ============ 6. 换回分区 & 重打包 ============
cp repack/boot_patched.img extract/partitions/boot_a.img

# a) 目录替换法（推荐，若原镜像是单 ext4）
sudo mount -o loop,rw extract/system.img /mnt/andarium
sudo cp repack/boot_patched.img /mnt/andarium/boot/boot.img
sync && sudo umount /mnt/andarium
e2fsck -f -y extract/system.img

# b) 若需 sparse 输出
# img2simg output/system_raw.img output/system.img

# ============ 7. 封 IPA ============
cp -a extract/ipa output/ipa-work
cp extract/system.img output/ipa-work/Payload/Andarium.app/andarium-system/guest/system.img
cd output/ipa-work && zip -qr9 ../Andarium-rooted.ipa Payload && cd ..
unzip -l output/Andarium-rooted.ipa | head

# ============ 8. 安装 & 验证 ============
# appinst output/Andarium-rooted.ipa   # 或 TrollStore / 自签
# 进 Andarium 终端：su -c 'id'
```

---

## 13. 附录 B：工具清单

| 工具 | 用途 | 获取方式 |
| --- | --- | --- |
| `simg2img` / `img2simg` | sparse ↔ raw 转换 | `android-sdk-libsparse-utils` |
| `make_ext4fs` | 生成 ext4 镜像 | `android-sdk-ext4-utils` / AOSP |
| `mount` / `losetup` / `fdisk` / `parted` | 挂载与分区 | 系统自带 |
| `e2fsck` / `resize2fs` / `debugfs` | ext4 检查与修复 | `e2fsprogs` |
| `magiskboot` | boot 镜像 unpack/repack/cpio | Magisk APK 内 (`lib/<abi>/libmagiskboot.so`) |
| `Magisk App` | 修补 boot | Magisk 官方 GitHub |
| `zip` / `unzip` | IPA 打包 | 系统自带 |
| `ssh` / `scp` / `adb` | 文件传输 | 系统自带 / platform-tools |

**提取 magiskboot**：

```bash
unzip Magisk-v27.0.apk 'lib/arm64-v8a/libmagiskboot.so' -d /tmp/magisk
chmod +x /tmp/magisk/lib/arm64-v8a/libmagiskboot.so
sudo mv /tmp/magisk/lib/arm64-v8a/libmagiskboot.so /usr/local/bin/magiskboot
magiskboot --help
```

---

## 14. 附录 C：分区文件清单

解包后从 `system.img` 带出来的 6 个分区（以你的镜像实际内容为准）：

| 序号 | 分区文件 | 典型作用 | 本文是否修改 |
| --- | --- | --- | --- |
| 1 | `boot`（如 `boot_a.img`） | 内核 + ramdisk，**root 的关键** | ✅ **修补** |
| 2 | `system`（如 `system_a.img`） | guest 的主系统文件系统 | ❌ 原样保留 |
| 3 | `vendor`（如 `vendor.img`） | 厂商驱动与 HAL | ❌ 原样保留 |
| 4 | `recovery`（如 `recovery.img`） | 恢复模式（部分镜像有） | ❌ 原样保留 |
| 5 | `dtbo`（如 `dtbo.img`） | 设备树 overlay | ❌ 原样保留 |
| 6 | `vbmeta`（如 `vbmeta.img`） | 校验元数据（avb） | ❌ 原样保留¹ |

> ¹ 如果你的镜像启用了 dm-verity / AVB，改 boot 后 `vbmeta` 校验会失败，此时需要用 `avbtool make_vbmeta_image --flags 2`（或 Magisk 一起 patch）重签 `vbmeta`，把 verity/verification 关掉。Andarium 的镜像若未启用 verity，则这一步可跳过。

**核对清单**（打勾确认再进行下一步）：

- [ ] 原始 `system.img` 已备份并记录 sha256
- [ ] 已确认原始镜像是 sparse 还是 raw
- [ ] 6 个分区文件已全部导出，且来自同一个镜像版本
- [ ] 只修改了 `boot` 分区，其余 5 个字节未变（`sha256sum -c`）
- [ ] Magisk 修补产物已用 `magiskboot cpio ls` 验证含 `magiskinit`
- [ ] 输出镜像的格式与原始镜像一致
- [ ] IPA 结构为 `Payload/Andarium.app/...`，无多余层级
- [ ] 已通过重签 / TrollStore 等方式安装
- [ ] `su -c 'id'` 返回 `uid=0(root)`

---

## 结语

整条链路上真正"有技术含量"的其实只有两件事：

1. **找到 `system.img` 并正确理解它的格式**（sparse？带不带分区表？）
2. **理解 Magisk 为什么改 boot 就够了**（抢在系统之前拿到 init）

剩下的都是"搬文件 + 保持格式"的体力活。把这几点想清楚，以后遇到类似"App 里跑了个 Linux"的场景（各种虚拟化容器、模拟器、Termux 类应用），拆除思路基本可以直接复用。

> 记录这份文档的目的不是鼓励改别人的 App，而是把"容器内提权"这件事的原理讲清楚——知道底层在发生什么，才有资格谈边界在哪里。

---

*最后更新：2026-11-20 ｜ 验证状态：✅ 已成功*

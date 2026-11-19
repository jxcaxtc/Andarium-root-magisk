# Andarium 获取 Root

## 结论

**Andarium 用不了内核层面的 root 管理器**，这是实测结论。

Magisk、KernelSU、APatch 这类工具的工作原理是接管内核启动、替换 init、建立 overlay 挂载。它们需要真正的 boot 分区和真正的内核启动流程。而 Andarium CONFIG_MODULES 未开，且 vermagic 也对不上 → 双重不满足所以就放弃了这条路

所以走的是另一条路：**改 guest 镜像里的 boot 分区**，让 guest 内部自己有一套 root 环境。这套 root 只在 Andarium 的 guest 里有效，出不到 iOS 层面。

## 流程

### 1. 拿到 system.img

从应用商店下载好，然后脱壳Andarium，直接进它的目录，`andarium-system/guest/system.img` 就是 guest 的系统盘。拷出来。

### 2. 解包

这个镜像是分区镜像，解开以后能拿到 6 个分区文件，其中 `boot` 是我们需要的，其余 5 个保持原样。

### 3. 修补 boot

把 `boot` 分区传到手机上，用 Magisk 选择"修补一个文件"，得到修补后的 boot。

### 4. 塞回去

把修补后的 boot 覆盖回原来的位置，重新打包成 system.img。
着重说一点修补过后的 boot 镜像必须和原 boot 镜像大小一致

### 5. 装回去

把新的 system.img 塞回 IPA 的对应位置，重新封装。因为改过包，签名失效，需要重签或用 TrollStore 之类的方式安装。


## 原理

为什么改 boot 就够了？

guest 启动的时候，内核会先执行 ramdisk 里的 `init`。Magisk 修补做的事很简单：把原来的 `init` 改名成 `init.orig`，自己塞一个 `magiskinit` 进去占住 `init` 这个名字。

于是启动顺序变成：内核执行 `magiskinit` → magiskinit 建立 root 环境（挂 su、起 magiskd、挂模块）→ 再 exec `init.orig` 把系统正常拉起来。

关键就在于 Magisk 跑在系统接手之前。它比系统启动得更早，所以能做的事比系统自己多。这也是为什么只改系统分区文件没用，改 boot 才有用。

## 注意

- 镜像格式必须和原来一致，原来是 sparse 就还是 sparse，否则加载不起来。
- 改过的 IPA 一定要重签，原签名必废。

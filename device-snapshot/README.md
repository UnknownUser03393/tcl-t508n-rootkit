# 设备状态快照

在 **2024 boot chain + Magisk root 工作正常** 的状态下，从设备只读导出的三份快照。
用来日后比对「什么时候变了什么」。

抓取时间：2026-09-16

| 文件 | 来源 | 行数 |
|---|---|---|
| `getprop.txt` | `adb shell getprop` | 734 |
| `mounts.txt` | `adb shell cat /proc/mounts` | 405 |
| `partitions.txt` | `adb shell su -c 'ls -l /dev/block/by-name'` | 81 |

## 抓取时状态

```
ro.boot.flash.locked          1
ro.boot.vbmeta.device_state   locked
ro.boot.verifiedbootstate     green
ro.boot.slot_suffix           _a
ro.build.fingerprint          TCL/OT5_CM_FIH/OT5:13/TP1A.220624.014/508NCM7S:user/release-keys
ro.serialno                   86PA02BAAHH00A9

boot chain   2024 版（uboot_a / trustos_a / sml_a / teecfg_a）
root         Magisk 26.3 (26300)，su -c id → uid=0(root)
```

> `flash.locked` 曾为 `0`，后来自动翻回 `1`（详见
> [`../bootchain-2024-unlocked/CAVEAT.md`](../bootchain-2024-unlocked/CAVEAT.md)）。
> **root 不受影响。**

## 分区对照

`partitions.txt` 是 `/dev/block/by-name` 的软链接表，例如：

```
boot_a   -> /dev/block/mmcblk0p36
boot_b   -> /dev/block/mmcblk0p37
```

配合 `../STOCK_508NCM7S/partitions.xml`（GPT 分区名 + 大小）
可以完整还原这台机器的分区布局。

# TCL T508N — 2024 boot chain（**实机证明可持久解锁**）

> 本目录是唯一一套**在真机上验证过、能把 `ro.boot.flash.locked` 从 `1` 变成 `0` 并持久保持**的 boot chain。
> 全部文件为设备只读回读（`read_part`），非构造。

---

## 结果（重启后实测）

```
ro.boot.flash.locked          0            ← 原来是 1
ro.boot.vbmeta.device_state   unlocked     ← 原来是 locked
ro.boot.verifiedbootstate     orange       ← 原来是 green
ro.build.fingerprint   TCL/OT5_CM_FIH/OT5:13/TP1A.220624.014/508NCM7S
ro.boot.slot_suffix           _a
```

**断电重启后三个值全部保持 → 持久解锁，非一次性状态。**

---

## 文件

| 文件 | 大小 | 说明 |
|---|---|---|
| `uboot_a.bin` | 3,145,728 | LK / u-boot，构建时间戳 `2024-01-16-17:41:42_LOCAL` |
| `trustos_a.bin` | 6,291,456 | TEE（TrustOS） |
| `sml_a.bin` | 1,048,576 | SML |
| `teecfg_a.bin` | 1,048,576 | TEE 配置 |
| `splloader.bin` | 262,144 | SPL — **与 2025 原厂逐字节相同，未改动** |
| `miscdata.bin` | 1,048,576 | 锁令牌所在分区（见下） |
| `misc.bin` | 1,048,576 | BCB |

校验值见 `SHA256SUMS.txt`。

---

## 复现步骤

前置：能进 BROM 下载模式；`spd_dump` + `fdl1-dl.bin` + `fdl2-dl.bin`（见上级目录 / Releases）。

```bash
adb reboot autodloader

spd_dump --wait 300 exec_addr 0x65012f48 \
  fdl fdl1-dl.bin 0x65000800 \
  fdl fdl2-dl.bin 0xb4fffe00 \
  exec \
  w uboot_a   uboot_a.bin \
  w trustos_a trustos_a.bin \
  w sml_a     sml_a.bin \
  w teecfg_a  teecfg_a.bin \
  reset
```

**`splloader` 不要动**（两版相同）。

刷完后设备会因 FBE 密钥换代而无法挂载既有 `/data`，需要双清一次重建用户数据。之后即可正常启动。

---

## 关键机制：锁判据在 boot chain 里，不在存储里

刷机**前后** `miscdata+0x2000` 的 64 字节锁令牌**完全没变**：

```
fb eb 22 49 d4 98 39 ed 42 04 f4 c8 fa 2f 10 a5
dc ef 99 27 dc f6 a4 09 cd ec d4 96 d9 83 85 d2
3e d2 5d d4 88 42 e9 fc 88 a1 b5 31 c2 51 21 5a
3e d2 5d d4 88 42 e9 fc 88 a1 b5 31 c2 51 21 5a
```

| | 同一份令牌 → |
|---|---|
| 2025 boot chain | `flash.locked = 1`（locked） |
| **2024 boot chain** | **`flash.locked = 0`（unlocked）** |

**结论：2025 版在原有 `trusty_unlock` / `uboot_verify_lockstatus` 之外新增的
`get_product_token` / `lock_token` / `sn8` 那两层，正是把该令牌判为 locked 的原因。**
2024 版没有这层，同一令牌判为 unlocked。

---

## 为什么 2024 与 2025 差别只在 boot chain

| 组件 | 2024 包 | 508NCM7S 现网 | |
|---|---|---|---|
| `splloader` | DHTB sha `b989f22a…` | 同 | **完全相同** |
| `vbmeta_a` | — | — | **完全相同** |
| `uboot_a` | `2024-01-16-17:41:42_LOCAL` | `2025-09-23-09:45:40_LOCAL` | 不同 |
| `trustos` / `sml` / `teecfg` | 2024 | 2025 | 内容不同 |

**反回滚（efuse `Trusted_rollback_version`）未拦截本次降级** —— SPL 那级的
`sprd_imgversion` 检查放行了 2024 的 uboot。

---

## ⚠️ 注意

1. **这是降级。** 回滚到 2025 原版（上级目录 `B/` 与设备原厂备份）即恢复 `locked`。
2. **`ro.boot.sn8` 会变空** —— 该字段是 2025 版引入的，2024 bootloader 不设置它。
   若 TCL 某些服务依赖 sn8，可能出现功能异常。
3. **`ro.boot.veritymode` 仍为 `enforcing`** —— bootloader 已解锁，但 dm-verity 仍在生效。
   要改 system 分区需另行处理；root 请走 Magisk 补丁 `boot` 镜像的路径。
4. `miscdata.bin` 含本机锁令牌与设备识别数据。

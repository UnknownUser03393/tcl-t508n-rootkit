# TCL T508N (Unisoc UMS9620) — BL 解锁 / Root / 降级资料

设备：**TCL T508N / OT5_CM_FIH / UMS9620 (T157) / Android 13 / A/B (VAB) / AVB 1.2**

本仓库归档两套社区流传的 T508N 工具与固件包，供研究与被锁设备自救使用。

---

## 提示
**第一次刷入后会出现一个非常哈人的页面，大概意思是你的数据炸了问你要不要重置，直接点重置即可。**
**不是炸机，不是变砖，就是你妈的这个破Google写的太烂了。**

## 包 1 — 解锁 BL & Root

| | |
|---|---|
| 文件 | `TCL508N解锁BL&Root.zip` |
| 大小 | 70,948,658 bytes (70.9 MB) |
| SHA256 | `fdce362492faa4476d37ad02a0d05a66b356eb6fe5c9720ac5466f534ff0e356` |
| 位置 | **本仓库（git）** |

内容：
- `1、驱动SPD_Driver_R4.20.4201/` — 展锐 USB 驱动（Win7 / Win10，含 sprdvcom / sprdvmdm / sprdadb）
- BL 解锁与 Root 脚本、配套工具（详见解压后目录）

---

## 包 2 — 2a 降级 flash_all 全分区镜像

| | |
|---|---|
| 文件 | `TCL 508N 2a降级flash_all_spd_dump.zip` |
| 大小 | 2,690,702,545 bytes (2.69 GB)，解压后约 7.45 GB |
| SHA256 | `be5e7ffd044fdc3256f2864474a284e3663bb1e9a3250970df6ed17884566e2d` |
| 位置 | **Releases**（因超过 GitHub 单文件 100 MB / Release 2 GB 限制，已 7z 分卷） |

分卷文件（在 Releases 页）：

```
TCL508N_2a_downgrade_flash_all_spd_dump.7z.001   1,887,436,800
TCL508N_2a_downgrade_flash_all_spd_dump.7z.002     803,265,903
```

**重组**（把两个分卷放同一目录，然后）：

```bash
7z x TCL508N_2a_downgrade_flash_all_spd_dump.7z.001
# 或用 7-Zip 图形界面右键 .001 → 提取
```

重组后的 SHA256 应等于上面的 `be5e7ffd...`。

### 内含分区（88 项）

绝大部分文件时间戳为 **2024-03-09**，是 2024 年的工厂/降级包：

```
2024-03-09  spl_a.img / spl_b.img              4,194,304
2024-03-09  mmcblk0boot0.img / boot1.img       4,194,304
2024-03-09  uboot_a.img / uboot_b.img          3,145,728
2024-03-09  trustos_a.img / trustos_b.img      6,291,456
2024-03-09  sml_a.img / sml_b.img              1,048,576
2024-03-09  teecfg_a.img / teecfg_b.img        1,048,576
2024-03-09  vbmeta_{a,b,odm,system,system_ext,vendor,product}_*.img
2024-03-09  misc.img / miscdata.img            1,048,576
2024-03-09  dtb_* / dtbo_* / init_boot_* / vendor_boot_*
2024-03-09  nr_modem_* / nr_phy_* / nr_fixnv*_* / nr_runtimenv*
2024-03-09  prodnv.img / persist.img / calinv.img / isedata.img
2024-03-09  super.img                          5,872,025,600
2025-04-29  boot_a.img / boot_b.img            67,108,864   ← 注意：非 2024 文件
2024-03-08  fdl1-dl.bin / fdl2-dl.bin
2025-02-21  spd_dump.exe
2024-03-16  part_tcl.xml
2024-05-21  Channel.ini / Channel9.dll
2024-03-09  fastboot.bat
2025-04-29  flash_all bat 的.bat
2024-01-25  custom_exec_no_verify_65012f48.bin
```

### 关键差异（相对 508NCM7S 现网版本）

| 组件 | 本包 (2024) | 508NCM7S (现网) |
|---|---|---|
| uboot 构建时间戳 | **`2024-01-16-17:41:42_LOCAL`** | `2025-09-23-09:45:40_LOCAL` |
| uboot 构建路径 | — | `/home/foxconn/T157/dailybuild_pfat_OT5_new01_FTM/...` |
| `get_product_token` | ❌ 无 | ✅ 有 |
| `lock_token` | ❌ 无 | ✅ 有 |
| `sn8`（SN 绑定） | ❌ 无 | ✅ 有 |
| `trusty_unlock` | ✅ 有 | ✅ 有 |
| `uboot_verify_lockstatus` | ✅ 有 | ✅ 有 |
| SPL (DHTB sha256) | `b989f22aa0950ea7…` | **相同** `b989f22aa0950ea7…` |

**重点：2024 与 2025 之间变化的是 u-boot / TrustOS / SML / teecfg，SPL 未变。**
2025-09 那次更新在原有 `trusty_unlock` / `uboot_verify_lockstatus` 之上，额外加了
**产品令牌（`get_product_token` / `lock_token`）** 与 **SN 绑定（`sn8`）** 两层。

---

## ⚠️ 警告

**1. 降级不可逆风险。**
展锐有独立于 AVB 的 `sprd_imgversion` 反回滚机制（RPMB 存储 + **efuse 基线**）。
若现网固件已抬高 efuse 中的 `Trusted_rollback_version`，回刷 2024 镜像会被
`SEC ANTI_ROLLBACK ERR` 拒绝。efuse 是单向的 —— **这不是"刷错还能用 FDL 救回来"的场景，执行前请自行评估变砖风险。**

**2. 本包含设备识别数据。**
`miscdata.img` / `prodnv.img` / `persist.img` / `nr_fixnv*.img` / `nr_runtimenv*.img`
来自真实设备，含序列号、provisioning、基带 NV 等可识别信息。**除研究外请勿用于其他用途。**

**3. 版权归原厂。**
固件内容版权归 TCL / 展锐（Foxconn FIH 代工）所有；`spd_dump` 等工具版权归其作者。
本仓库仅作归档与研究，**仓库根目录的 AGPL-3.0 LICENSE 不适用于包内的厂商固件与第三方二进制**。

---

## 校验

```bash
sha256sum "TCL508N解锁BL&Root.zip"
# fdce362492faa4476d37ad02a0d05a66b356eb6fe5c9720ac5466f534ff0e356
```

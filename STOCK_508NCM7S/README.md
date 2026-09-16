# 原厂 boot chain 完整备份 — 508NCM7S（2025-09 版）

> **这是这台机器的「出厂态」boot chain。降级前从设备只读 dump 出来的，
> 用于随时刷回。别删。**

设备：TCL T508N / OT5_CM_FIH / UMS9620 / Android 13 / A/B
指纹：`TCL/OT5_CM_FIH/OT5:13/TP1A.220624.014/508NCM7S:user/release-keys`
uboot 构建时间戳：`2025-09-23-09:45:40_LOCAL`
构建路径：`/home/foxconn/T157/dailybuild_pfat_OT5_new01_FTM/...`

---

## 文件

| 文件 | 大小 | 分区 | 说明 |
|---|---|---|---|
| `splloader.img` | 262,144 | splloader | SPL。**2024/2025 两版完全相同** |
| `uboot_a.img` / `uboot_b.img` | 3,145,728 | uboot_a / uboot_b | LK / u-boot（2025-09-23） |
| `trustos_a.img` / `trustos_b.img` | 6,291,456 | trustos_a / trustos_b | TEE (TrustOS) |
| `sml_a.img` / `sml_b.img` | 1,048,576 | sml_a / sml_b | SML |
| `teecfg_a.img` / `teecfg_b.img` | 1,048,576 | teecfg_a / teecfg_b | TEE 配置 |
| `vbmeta_*.img` | 1,048,576 ×6 | vbmeta_* | AVB 元数据（top/system/vendor/system_ext/product/odm） |
| `pgpt.bin` | 32,768 | — | 主 GPT 表 |
| `partitions.xml` | 3,184 | — | `spd_dump` 分区表导出 |

**A 槽和 B 槽的同名文件逐字节相同**（`uboot_a == uboot_b`，其余同理）。

校验值见 `SHA256SUMS.txt`。分区设备节点见 `../device-snapshot/partitions.txt`。

---

## 刷回方法

需要 BROM 下载模式 + `spd_dump` + `fdl1-dl.bin` / `fdl2-dl.bin`。

```bash
adb reboot autodloader

spd_dump --wait 300 exec_addr 0x65012f48 \
  fdl fdl1-dl.bin 0x65000800 \
  fdl fdl2-dl.bin 0xb4fffe00 \
  exec \
  w uboot_a   uboot_a.img   w uboot_b   uboot_b.img \
  w trustos_a trustos_a.img w trustos_b trustos_b.img \
  w sml_a     sml_a.img     w sml_b     sml_b.img \
  w teecfg_a  teecfg_a.img  w teecfg_b  teecfg_b.img \
  reset
```

`splloader` **不用刷**（两版相同，也没被改动过）。

> **注意**：刷回 2025 boot chain 会把 `ro.boot.flash.locked` 恢复为 `1`。
> 若当前 `/data` 是用 2024 TEE 加密的，刷回后可能出现 `init_user0_failed` —— 反之亦然。
> **TEE 换代会导致既有 `/data` 的 FBE 密钥解不开**，这是预期行为，不是变砖。

---

## 与本仓库其他目录的关系

- `../bootchain-2024-unlocked/` —— 2024 版（能解锁的那套）
- `../TCL508N解锁BL&Root.zip` —— 原版工具
- Releases `v1.0` —— 2024 工厂固件全分区包（2.69 GB）

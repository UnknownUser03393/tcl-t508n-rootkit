# 实测补充：解锁状态后来自己翻回 locked

写这份笔记是为了不让这个目录的说明误导后来的人。

## 现象

按 `README.md` 的步骤做完（刷 2024 boot chain → 双清 → 确认 `flash.locked=0`
→ 刷 Magisk 补丁的 boot → 重启确认仍为 0）之后，**某次普通 `adb reboot` 后
`flash.locked` 自己翻回了 `1`**，`verifiedbootstate` 变成 `green`。

关键事实：

- **boot chain 一个字节没动** —— `uboot_a` / `trustos_a` / `sml_a` / `teecfg_a`
  回读哈希与刷入时完全一致
- **`boot_a`（Magisk 补丁版）也没变**
- **`miscdata+0x2000` 的 64 字节锁令牌逐字节未变**
- **root 一直正常**，`su -c id` → `uid=0(root)`
- 重启 3 次，`flash.locked` 稳定为 `1`，不是抖动

## `miscdata` 里唯一的变化

整个 1 MB 分区只差了 **5 个字节**，全在 `+0x2900` 这个结构：

```
magic a5a55a5a | u32@0x2904 | 00000000 | u32@0x290c | ffffffff | 8B
解锁时          | 0x68d205ad |          | 0xb1 (177) |
翻回后          | 0x6aaa99d0 |          | 0xed (237) |
```

`0x2904` 解出来是 **Unix 时间戳**：

- `0x68d205ad` = 1758594477 = **2025-09-23**（2025 版 bootloader 的构建日期）
- `0x6aaa99d0` = 1789565392 = **2026-09-16**（当天）

uboot 里有对应字符串：`read miscdata timestamp error.`。
**bootloader 每次启动会更新这个时间戳记录。**

## 结论：**不知道原因**

相关性很强，但没有证据链。uboot 里 `androidboot.flash.locked=0/1` 的判定代码
在 2024 版的 `0xcfd60`–`0xcfe40`，会读若干 miscdata 字段拼 cmdline，
但「=0」那一支的引用用 ADRP+ADD xref 扫不到（疑似走计算偏移），**判据没吃透**。

未验证的猜测（按可能性排）：

1. 锁判据依赖 `miscdata+0x2900` 那个时间戳/计数器记录，它被更新后解锁失效
2. 解锁状态本来就绑定「双清后的首次启动」，后续启动会重新评估
3. 2024 TEE 的 `FUNCTYPE_CHECK_LOCK_STATUS` 有超时/次数限制

## 没试过的（可能有用）

**把同样那 4 个 2024 分区再刷一遍，不双清。**
TEE 同代，FBE 密钥不受影响，数据不会丢。
- 若 `flash.locked` 变回 0 → 说明有东西把状态冲掉了，可修
- 若仍为 1 → 说明得再双清

## 实用角度

当前状态是 **`green` + `locked` + 有 root**。
和 `orange` 相比，**`green` 对银行类 / 检测 bootloader 状态的 App 反而更友好**。
root 不受影响。

代价：`fastboot flashing unlock` 之类会被拒，刷写只能走 BROM / `spd_dump`。

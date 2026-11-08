# Meizu 20（kalama）GKI 兼容补丁集

目标：让纯 ACK `android13-5.15-lts`（5.15.195，commit 8a8f52308945）能够加载魅族原厂
`vendor_boot` / `vendor_dlkm` 中全部 643 个 `.ko`，且所有导入符号的 CRC 与原厂内核一致。

## 0001-android-gki-export-meizu-kmi-symbols.patch

魅族模块导入、但 ACK 未导出的三个内核符号：

| 符号 | 导入方 | 原厂导出类型 | 原厂 CRC |
|---|---|---|---|
| `__tracepoint_android_vh_extfrag_for_order` | `mz_compaction_opt.ko` | `ksymtab_gpl` | `0xf18a8605` |
| `__tracepoint_android_vh_smartzram_tune_swappiness` | `smartswap_zram.ko` | `ksymtab_gpl` | `0x1523d65f` |
| `mem_cgroup_id_put2` | `smartswap_zram.ko` | `ksymtab_gpl` | `0xf78296e5` |

`mz_compaction_opt.ko` 在 `vendor_boot/lib/modules/modules.load` 中，是开机第一阶段
必须加载的模块；缺符号会导致 `finit_module` 失败，Android init 在 strict 模式下打印
"Failed to load kernel modules" 后 abort，表现为无限重启。另两个符号供后续按需加载的
魅族 zram 模块使用。

`mem_cgroup_id_put2` 是 `mem_cgroup_id_put()` 的 out-of-line 导出包装，函数体必须保持
内联调用 `mem_cgroup_id_put()` 以维持原厂 CRC。

## 0002 / 0003：补回原厂内核的 hook 调用点

`extfrag_for_order()` 与 `lru_gen_shrink_lruvec()` 在原厂内核中分别调用了上述两个
tracepoint（已用原厂 vmlinux 反汇编确认调用点：`extfrag_for_order`、`lru_gen_shrink_lruvec`）。
只导出 tracepoint 而不调用，模块能加载但魅族内存调优功能失效，因此两个调用点必须补齐。

`#include <trace/hooks/mm.h>` 必须放在文件内最后一个 `EXPORT_SYMBOL` 之后、调用点之前：
该头文件在 `__GENKSYMS__` 下会引入 `linux/mm.h` 与 `mm/slab.h`，放在文件顶部会改变
同文件既有导出符号的 genksyms CRC（实测 `register_shrinker`、`shrink_slab`、
`try_to_free_mem_cgroup_pages`、`__mod_zone_page_state` 等全部漂移）。

## 0004-of-address-keep-pci-host-bridges-off-default-flags-bus.patch

5.15.127 引入的 `default-flags` bus 以 `#address-cells == 3` 作为匹配条件；魅族 DTB 的
`qcom,pcie@1c00000`（compatible `qcom,pci-msm`）没有 `device_type = "pci"`，会被该 bus
抢走并把 ranges 的第一个 cell（PCI 地址空间类型）当作 flags，导致根总线丢失
`IORESOURCE_MEM`、BAR 分配失败、WiFi 无法 probe。原厂 5.15.119 没有该 bus，等价于
plain default bus，因此本补丁排除该节点以恢复原厂行为。

## 验收判据

对产物 Image 重建符号表后，用 `_work/scan_compat.py` 扫描 `vendor_boot` 339 个 +
`vendor_dlkm` 304 个 `.ko`：

- 内核级缺失符号数 = 0
- 内核导出 CRC 与模块 `__versions` 失配数 = 0

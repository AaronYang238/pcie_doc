# 第14章　Linux PCI 的初始化过程

> **导读**：前面讲硬件「能做什么」，本章讲软件「怎么把它跑起来」。以 Linux 为例，走一遍 **PCI 子系统初始化 → ACPI 提供拓扑/资源 → 枚举总线 → 分配 BAR** 的完整代码路径。
>
> **现代化提示**：原书基于较早内核，函数名/路径有演进；本章保留原书调用链并标注「新内核对应」。
>
> **本页为大纲骨架**，正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| — | initcall | 内核按等级顺序调用的初始化函数机制 |
| ACPI | 高级配置与电源接口 | 固件向 OS 描述硬件的表与方法 |
| ACPI MCFG | MCFG 表 | 上报 ECAM 基址（供内存映射配置访问） |
| — | DSDT / `_CRS` / `_PRT` | 描述资源(`_CRS`)与中断路由(`_PRT`)的 ACPI 方法 |
| Host Bridge (PNP0A03/0A08) | 根桥 | ACPI 声明的 PCI 根桥 |
| DFS | 深度优先枚举 | 递归扫描分配 Bus 号（呼应第02章） |
| Resource Assignment | 资源分配 | 为设备/桥窗口分配 MMIO/IO 地址 |
| Device Tree (OF) | 设备树 | 非 x86（如 PowerPC/ARM）描述硬件的机制 |

---

## 14.1 Linux x86 对 PCI 总线的初始化
- **14.1.1 `pcibus_class_init` 与 `pci_driver_init` 函数**：注册 PCI 总线类型与设备/驱动模型。
- **14.1.2 `pci_arch_init` 函数**：选择配置访问方式（ECAM/CF8）。
- **14.1.3 `pci_slot_init` 和 `pci_subsys_init` 函数**：⭐真正触发总线扫描的入口。
- **14.1.4 与 PCI 总线初始化相关的其他函数**：`initcall` 顺序与依赖。

## 14.2 x86 处理器的 ACPI
- **14.2.1 ACPI 驱动程序与 AML 解释器**：ACPI 如何向 OS 描述硬件。
- **14.2.2 ACPI 表**：⭐**MCFG**（ECAM 基址，呼应 [第05章](../第二篇-PCIe体系结构/第05章-平台MCH与ICH.md)）、**DSDT**（`_CRS`/`_PRT`）、**MADT**（中断，呼应 [第15章](第15章-Linux-PCI中断处理.md)）。
- **14.2.3 ACPI 表的使用实例**：从表读出总线号范围与资源窗口。

## 14.3 基于 ACPI 机制的 Linux PCI 的初始化
- **14.3.1 基本的准备工作**：root bridge 的发现（`_HID` PNP0A03/0A08）。
- **14.3.2 初始化 PCI 总线号**：DFS 枚举（呼应 [第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)）。
- **14.3.3 检查 PCI 设备使用的 BAR 空间**：探测每个 BAR 大小与类型。
- **14.3.4 分配 PCI 设备使用的 BAR 寄存器**：⭐资源分配器把窗口分给设备/桥（`pci_assign_resource`），处理冲突与对齐。

## 14.4 Linux PowerPC 如何初始化 PCI 总线树
- 设备树（Device Tree, OF）驱动的初始化路径，与 x86 ACPI 路径对照。

## 14.5 小结
- 一张「上电 → 固件建表 → 内核枚举 → 分配资源 → 驱动 probe」的时间线。

---

## 📘 现代内核 / SPEC 对照清单
- **ECAM 经 MCFG**：`pci_mmconfig_*` 使用 ACPI MCFG 表提供的 ECAM 基址（呼应 [第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)/[第05章](../第二篇-PCIe体系结构/第05章-平台MCH与ICH.md)）。
- **函数演进**：原书部分函数在新内核已重构（如 host bridge 抽象为 `pci_host_bridge`、`pci_scan_root_bus_bridge`）；保留原名并加对应注记。
- **Resizable BAR / 大 BAR**：现代分配器需处理 64 位高窗口与 above-4G 资源（呼应 [第03章](../第一篇-PCI体系结构/第03章-PCI数据交换.md)）。
- **热插拔与 `pci=` 参数**：现代平台的 pciehp、资源重分配（`pci=realloc`）值得补一节。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第14章-初始化调用流程.svg` | 初始化调用流程 | initcall → arch_init → subsys_init → scan_bus → assign_resource 链 |
| `第14章-ACPI表关系.svg` | ACPI 表关系 | RSDP→XSDT→{MCFG,DSDT,MADT} 与各自用途 |
| `第14章-BAR分配.svg` | BAR 资源分配 | 探测大小→按对齐/窗口分配→写回基址 的分配器视图 |

## 与 SPEC / 规范对照（占位）
| 本章主题 | 规范 / 内核 |
|----------|-------------|
| ECAM / MCFG | ACPI 规范；Base Spec Enhanced Config |
| 枚举 / 资源分配 | Base Spec Configuration；Linux `drivers/pci/` |

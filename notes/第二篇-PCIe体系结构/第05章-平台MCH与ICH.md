# 第05章　Montevina 平台的 MCH 与 ICH

> **导读**：本章是原书用真实芯片组（Intel Montevina：MCH 北桥 + ICH 南桥）落地前几章抽象概念的「案例课」——看具体寄存器如何划分存储器空间、如何把 PCIe 配置空间映射进来。**现代化重点**：Montevina 属北桥/南桥分立时代，今天 MCH 功能已集成进 CPU（RC 内建），本章用「历史案例 + 现代对照」双轨讲。
>
> **本页为大纲骨架**，正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| MCH, Memory Controller Hub | 存储器控制器中心 | 北桥，含内存控制器与 PCIe Root Port |
| ICH, I/O Controller Hub | I/O 控制器中心 | 南桥，接慢速外设与传统 I/O |
| — | PCIEXBAR | 定义 ECAM 基址的寄存器 |
| — | MCHBAR / EPBAR / DMIBAR | 定位 MCH 内部/Egress/DMI 寄存器块的 BAR |
| Top of Low/Upper Usable DRAM | TOLUD / TOUUD | 划分可用 DRAM 上界，处理 4G 内存空洞 |
| — | Legacy 地址空间 | 低 1MB、VGA、SMM 等历史保留区 |
| Memory-Mapped I/O | MMIO | 用存储器地址访问设备寄存器 |
| ACPI MCFG | MCFG 表 | 固件上报 ECAM 基址的 ACPI 表 |

---

## 5.1 PCI 总线 0 的 Device 0 设备
- **5.1.1 EPBAR 寄存器**：Egress Port BAR，定位 RC 内部寄存器块。
- **5.1.2 MCHBAR 寄存器**：MCH 内部配置寄存器窗口。
- **5.1.3 其他寄存器**：DMIBAR、PCIEXBAR（ECAM 基址）等关键 BAR 的作用。

## 5.2 Montevina 平台的存储器空间的组成结构
- **5.2.1 Legacy 地址空间**：低 1MB、VGA、SMM 等历史保留区。
- **5.2.2 DRAM 域**：物理内存分布与 remap（TOLUD/TOUUD、内存空洞回收）。
- **5.2.3 存储器域**：MMIO 与 DRAM 的合成视图。

## 5.3 存储器域的 PCI 总线地址空间
- **5.3.1 PCI 设备使用的地址空间**：MMIO 窗口、prefetchable 区的划分。
- **5.3.2 PCIe 总线的配置空间**：⭐**PCIEXBAR** 定义 ECAM 基址 —— 把 `Bus/Dev/Fun/Reg` 映射为内存地址（呼应 [第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md) 的 ECAM）。

## 5.4 小结
- 一张平台地址映射图收束：DRAM / Legacy / MMIO / ECAM 各占何处。

---

## 📘 SPEC 7.0 现代化补充清单
- **北桥消失，RC 进 CPU**：MCH 的内存控制器与 Root Port 已集成到 SoC；DMI/QPI → 片内互联。ECAM（PCIEXBAR 的思想）由固件通过 ACPI **MCFG 表**上报（呼应 [第14章](../第三篇-Linux与PCI/第14章-Linux-PCI初始化.md)）。
- **地址空间扩展**：64 位 MMIO 高窗口（above 4G）成为常态，承载大 BAR / Resizable BAR。
- **平台固件**：现代平台以 UEFI + ACPI 描述这些窗口，Montevina 的具体寄存器仅作历史参考。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第05章-平台框图.svg` | Montevina 平台框图 | CPU–MCH–ICH 分立结构 + 现代 SoC 集成对照 |
| `第05章-存储器映射.svg` | 平台存储器映射 | 0→TOLUD→4G→above-4G 各区（DRAM/Legacy/MMIO/ECAM）|
| `第05章-ECAM映射.svg` | PCIEXBAR/ECAM 映射 | Bus/Dev/Fun/Reg → 内存地址 位域拼接 |

## 与 SPEC 7.0 章节对照（占位）
| 本章主题 | Base Spec 7.0 章节 / 相关规范 |
|----------|-------------------------------|
| ECAM | （填充：Enhanced Config Access；ACPI MCFG）|
| 地址空间 | （填充：Address Spaces）|

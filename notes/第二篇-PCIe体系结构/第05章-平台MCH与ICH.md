# 第05章　平台如何组织地址空间（附 MCH/ICH 历史案例）

> **导读**：前面几章讲的都是"抽象规则"——配置空间怎么组织（[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)）、地址怎么分域、BAR 怎么探测（[第03章](../第一篇-PCI体系结构/第03章-PCI数据交换.md)）、ECAM 怎么映射。但这些规则需要一个**平台落点**。这一章回答一个具体的问题：**"CPU 一上电，是怎么知道内存有多大、设备寄存器在哪、配置空间去哪访问的？"**
>
> 答案是一套**每台现代 x86 机器都在用的通用组织逻辑**：`Bus 0 / Device 0` 承载平台核心配置、存储器空间按"楼层"划分（DRAM / MMIO / ECAM，配 remap 回收 4G 空洞）、ECAM 基址把配置空间钉进内存。本章**先讲这套现役逻辑**，再以原书的 **Intel Montevina 平台（MCH 北桥 + ICH 南桥）** 作**历史案例**，看同样的逻辑在分立芯片组时代的具体寄存器形态——原书这一章是"案例课"，我们把叙事反过来：**逻辑为主、案例为注脚**。你要带走的是组织逻辑，Montevina 的寄存器只是它的一个历史切面。
>
> 本章按 [CLAUDE.md §5.1](../../CLAUDE.md) 八节骨架展开，配 3 张图。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| Host Bridge (Bus0 Dev0) | 主桥设备 | 承载平台核心配置的特殊设备，现代在 CPU 片内 |
| TOLUD / TOUUD | 低/高端可用 DRAM 顶 | 划分可用 DRAM 上界，处理 4G 内存空洞 |
| Memory Remap | 内存重映射 | 把被 MMIO 遮住的 DRAM 搬到 4G 以上 |
| Legacy Address Space | Legacy 地址空间 | 低 1MB、VGA、SMM 等历史保留区 |
| MMIO | 存储器映射 I/O | 用存储器地址访问设备寄存器 |
| ECAM | 增强配置访问机制 | 把 Bus/Dev/Fun/Reg 映射为内存地址（[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)） |
| ACPI MCFG | MCFG 表 | 现代固件上报 ECAM 基址的 ACPI 表 |
| MCH, Memory Controller Hub | 存储器控制器中心 | 历史案例：北桥，含内存控制器与 Root Port |
| ICH, I/O Controller Hub | I/O 控制器中心 | 历史案例：南桥，接慢速外设；后继为 PCH |
| PCIEXBAR | ECAM 基址寄存器 | Montevina 上定义 ECAM 基址的寄存器 |
| MCHBAR / EPBAR / DMIBAR | 内部寄存器窗口 BAR | Montevina 上定位北桥内部寄存器块的 BAR |

---

## 核心概念：Bus0 Dev0 是"平台配置的总开关"

在展开之前，先立一个贯穿全章、且**在今天每台机器上都能验证**的认知：

**每个 PCIe 平台上，`Bus 0 / Device 0` 这个特殊设备承载着"整个平台如何组织地址空间"的核心配置。** 你现在敲 `lspci`，第一行的 `00:00.0 Host bridge` 就是它——如今它是 **CPU 里集成的 Root Complex/内存控制器**，在 Montevina 时代它是**北桥 MCH 芯片本身**。物理载体变了，职责一字未变。平台固件通过它（及配套的固件表）告诉系统三件事：

- **ECAM 配置空间放在哪**（现代经 ACPI MCFG 表；历史上是 `PCIEXBAR` 寄存器）；
- **可用 DRAM 到哪、MMIO 窗口在哪**（`TOLUD`/`TOUUD` 这类边界）；
- **平台内部机构的寄存器在哪**（历史上的 `MCHBAR`/`EPBAR`/`DMIBAR` 一类窗口）。

所以本章的三节机制（5.1–5.3）讲的是**现役逻辑**：总开关是谁、地址空间怎么分层、ECAM 怎么落位；5.4 再回到 Montevina，看这套逻辑最初的寄存器形态。

---

## 5.1 平台的总开关：Bus 0 / Device 0（从北桥到片内 RC）

先看"总开关"的物理载体如何演进——这张图同时回答"MCH/ICH 是什么"和"它们去哪了"：

![平台演进：分立到 SoC](../../assets/svg/第05章-平台框图.svg)

如图右侧（**现代形态，主线**）：**内存控制器和 Root Complex 都集成在 CPU/SoC 里**，`Bus0 Dev0` 就在片内；南桥演化为 **PCH（Platform Controller Hub）**，经片内 DMI 承接 USB/SATA 等慢速 I/O（低功耗 SoC 甚至把 PCH 也整合了）。图左侧（**历史案例**）：Montevina 是三段式——CPU（仅计算）经 FSB 前端总线连 **MCH 北桥**（内存控制器 + PCIe Root Port，即当时的 Bus0 Dev0），MCH 再经 DMI 连 **ICH 南桥**。

不变的是职责：**Bus0 Dev0 把平台自己的"内部机构"暴露成一组特殊寄存器/窗口，让固件和 OS 能配置平台本身**——它不是给外设用的普通设备，而是"平台的控制面板"。5.2 和 5.3 就是这块面板上最重要的两组旋钮。

## 5.2 平台存储器映射：一张"楼层图"

平台的物理地址空间不是一整块 DRAM，而是被切成好几层。这张图是本章的核心，也是每台现代机器 `/proc/iomem` 的骨架：

![平台存储器映射](../../assets/svg/第05章-存储器映射.svg)

### 5.2.1 Legacy 地址空间

最低的 **1MB** 及若干历史区域是**传统保留区**：实模式的低 1MB、**VGA** 帧缓冲区间、**SMM（系统管理模式）** 内存、BIOS 影子区等。这些是从 PC 诞生沿袭下来的固定约定，现代平台仍要兼容保留。

### 5.2.2 DRAM 域与 remap ⭐

这里有个新手最困惑、但今天依然真实存在的现象：**MMIO 窗口必须占掉 4G 以下的一段地址空间**（设备 BAR、ECAM、APIC 都要落地址），于是**物理内存里"地址与 MMIO 重叠"的那一段 DRAM 就无处安放**——不处理就浪费（这就是著名的"**4G 内存空洞**"：装 4GB 内存只见 3GB 多）。

平台的解法是 **remap（重映射）**：把被遮住的那段 DRAM **搬到 4G 以上**。两条分界线标出可用 DRAM 的范围：

- **TOLUD（Top of Low Usable DRAM）**：4G 以下可用 DRAM 的顶——再往上就是 MMIO 窗口。
- **TOUUD（Top of Upper Usable DRAM）**：4G 以上可用 DRAM 的顶——被 remap 上来的 DRAM 落在这里。

如图，从下到上的楼层是：`Legacy(低1M) → 可用DRAM主体 → [TOLUD] → 4G以下MMIO → [4G] → remap的DRAM → [TOUUD] → 64位MMIO高窗口`。

### 5.2.3 存储器域

把 DRAM 和 MMIO 合起来看，就是 CPU 眼中的完整**存储器域**（[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md) 的域概念）——一个地址落下去，可能命中 DRAM、可能命中某设备的 BAR、可能命中 ECAM。谁在哪一层，由固件设置的这些边界界定。**设备的 MMIO 窗口**（BAR）就落在"4G 以下 MMIO"和"64 位高窗口"两段；现代大 BAR（GPU 显存）与 **Prefetchable 区**普遍放高窗口（[第03章](../第一篇-PCI体系结构/第03章-PCI数据交换.md)），4G 以下装不下。

## 5.3 把配置空间钉进地址空间：ECAM 基址 ⭐

[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md) 讲了 ECAM 的机制，但留了一个问题：**基址从哪来？** 答案：由平台固件设定，并告知 OS——现代经 **ACPI MCFG 表**上报（[第14章](../第三篇-Linux与PCI/第14章-Linux-PCI初始化.md) 会看到内核怎么读它）；历史上（Montevina）则是 Bus0 Dev0 里一个叫 **`PCIEXBAR`** 的寄存器直接定义。基址定了之后，设备坐标直接编码进地址高位：

![ECAM 基址与位拼接](../../assets/svg/第05章-ECAM映射.svg)

如图，ECAM 地址的拼接公式是：

> **ECAM 地址 = 基址 + (Bus << 20) + (Dev << 15) + (Fun << 12) + Reg**

各位段：Bus 占 8 位（256 条总线）、Dev 占 5 位（32 设备）、Fun 占 3 位（8 功能）、Reg 占 12 位（每设备 **4KB** 配置空间）。一个 Segment 因此占 `256 × 32 × 8 × 4KB = 256MB` 地址。

举例：访问 `Bus 3 / Dev 5 / Fun 0` 的偏移 `0x10`（BAR0），地址就是 `基址 + 0x300000 + 0x28000 + 0x010`——CPU 对这个地址做一次普通内存读，就读到了该设备的 BAR0。

**对比老办法 CF8/CFC**（[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)）：端口机制要"两步、加锁、只能碰前 256 字节"；ECAM 一步到位、天然原子、覆盖完整 4KB、能访问 Extended Capabilities（AER/SR-IOV，[第13章](第13章-虚拟化技术.md)）。**"固件把 ECAM 基址钉进地址空间"就是让这一切成立的那把钥匙。**

## 5.4 历史案例：Montevina 的 MCH 与 ICH

现役逻辑讲完，回到原书的案例，看它在 2008 年前后的寄存器形态。**这一节仅作历史参考**——价值在于：读懂老平台手册/老资料的思路，并印证"逻辑不变、载体演进"。

Montevina 的 **MCH（北桥）就是当时的 Bus0 Dev0**，它的配置空间里有一组特殊 BAR，把平台"内部机构"暴露成 MMIO 窗口：

- **`PCIEXBAR`**：⭐案例里最重要的一个——**直接定义 ECAM 基址**（5.3 的逻辑在当时的落地方式）。今天这个职责由固件设定 + MCFG 表上报接替。
- **`MCHBAR`**：指向 MCH 的**内部配置寄存器窗口**——内存时序、通道配置等，是"北桥的控制面板"，固件初始化内存时大量读写。今天对应 CPU 片内内存控制器的私有寄存器。
- **`EPBAR`**：定位 RC 的 **Egress Port**（内部通往下游的出口）寄存器块。
- **`DMIBAR`**：定位 **DMI 链路**（MCH↔ICH 之间）的寄存器块。今天 DMI 仍在（CPU↔PCH），只是形态演进。
- **`TOLUD`/`TOUUD`** 等边界寄存器：5.2 楼层图的分界线，同样由 MCH 承载。

而 **ICH（南桥）** 承接 USB/SATA/传统 I/O——它的后继就是今天的 PCH。对照 5.1 的图：**每一个 Montevina 部件都能在现代 SoC 里找到对应物**，这正是"学逻辑、不背寄存器"的依据。

## 5.5 小结

一句话收束本章：

> **平台组织地址空间的通用逻辑：`Bus0 Dev0` 是总开关（现代在 CPU 片内，历史上是 MCH 北桥）——它界定 ECAM 基址（现代经 MCFG 上报）、用 `TOLUD/TOUUD` 划分 DRAM 与 MMIO 的楼层（靠 remap 回收 4G 空洞）、把平台内部机构暴露成寄存器窗口。载体从北桥搬进 CPU，逻辑一字未变。**

三个要点：

1. **Bus0 Dev0 是总开关**：`lspci` 第一行的 Host bridge，承载平台核心配置——今天仍然如此。
2. **地址空间是分层的**：Legacy / DRAM / MMIO / ECAM 各占其位；MMIO 遮住的 DRAM 靠 remap（TOLUD/TOUUD）搬到 4G 以上。
3. **ECAM 基址由固件钉定**：现代经 ACPI MCFG 表上报（历史上是 PCIEXBAR 寄存器），是 [第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md) ECAM 机制的平台落地。

带着"平台如何组织地址空间"的认知，第Ⅲ篇会从软件侧接上：[第14章](../第三篇-Linux与PCI/第14章-Linux-PCI初始化.md) 讲 Linux 如何读 ACPI（含 **MCFG 表**）来初始化 PCI 子系统。

---

## 📘 SPEC 7.0 现代化对照

> 📘 **北桥消失，RC 进 CPU**。Montevina 的 MCH 是独立芯片；现代平台把**内存控制器和 PCIe Root Complex 都集成进 CPU/SoC**，南桥变成 **PCH**，通过**片内互联**（DMI 的后继）相连。变化的是物理封装，**不变的是"Bus0 Dev0 承载平台核心配置"这套逻辑**——`lspci` 里你仍能看到 Bus0 上代表 RC/内存控制器的设备。（对照 5.1 框图两侧）

> 📘 **ECAM 基址由 ACPI MCFG 表上报**。Montevina 用 `PCIEXBAR` 寄存器给 ECAM 基址；现代平台由 **UEFI 固件通过 ACPI 的 MCFG 表**把 ECAM 基址（可能多个 Segment）告诉 OS。OS 启动时解析 MCFG，才知道去哪访问配置空间（[第14章](../第三篇-Linux与PCI/第14章-Linux-PCI初始化.md)）。**PCIEXBAR 的"思想"仍在，只是上报方式标准化成了 ACPI 表。**

> 📘 **64 位 MMIO 高窗口成为常态**。原书时代设备 BAR 主要挤在 4G 以下；现代 GPU/加速器动辄几十 GB 显存，必须用 **64 位 MMIO 高窗口 + 64 位可预读 BAR + Resizable BAR**（[第03章](../第一篇-PCI体系结构/第03章-PCI数据交换.md)）。5.2 楼层图顶部的"64 位高窗口"就是为此而设，TOUUD 之上的巨大地址空间正好容纳它们。

> 📘 **平台描述全面转向 UEFI + ACPI**。Montevina 的具体寄存器（MCHBAR/EPBAR/DMIBAR 的偏移与位定义）**仅作历史参考**——现代平台用 UEFI + ACPI（MCFG、DMAR、SRAT 等表）以**标准化、可移植**的方式描述这些窗口和拓扑，OS 不再需要硬编码某颗北桥的寄存器细节。

---

## 常见误区 / FAQ

**Q1：都用 SoC 了，学 Montevina 的 MCH/ICH 还有意义吗？**
有，但要抓对重点。**不要背** Montevina 的具体寄存器偏移——那些确实过时了，所以本章把它们收进 5.4 一节作历史案例。**要学**的是 5.1–5.3 的现役逻辑：Bus0 Dev0 承载核心配置、地址空间分层与 remap、ECAM 基址如何钉定。这套逻辑在你手边的机器上原样运行（`lspci`/`/proc/iomem` 可验证），只是载体从北桥搬进了 CPU、上报方式换成了 ACPI。

**Q2：什么是"4G 内存空洞"？为什么装 4GB 内存只能用 3GB 多？**
因为 MMIO 窗口（设备 BAR、ECAM、APIC 等）必须占用 4G 以下的一段真实地址空间——这段地址被 MMIO 用了，物理上同地址的那段 DRAM 就被"遮住"、无法通过该地址访问。若不处理就浪费了（表现为"少了近 1GB"）。解法是 **remap**：把被遮的 DRAM 搬到 4G 以上（TOUUD 之下），64 位系统就能重新用上它。

**Q3：TOLUD 和 TOUUD 到底划分什么？**
两条"可用 DRAM 的天花板"线。**TOLUD（Top of Low Usable DRAM）**=4G 以下可用 DRAM 的顶，再往上是 MMIO 窗口。**TOUUD（Top of Upper Usable DRAM）**=4G 以上可用 DRAM 的顶（remap 上来的 DRAM 落在 4G 与 TOUUD 之间）。固件设好这两个值，OS 就知道哪些物理地址是真内存、哪些是 MMIO。

**Q4：ECAM 基址现代和历史的来源有什么区别？和 CF8/CFC 又是什么关系？**
来源：历史上（Montevina）由 Bus0 Dev0 的 `PCIEXBAR` 寄存器直接定义；现代由固件设定并经 **ACPI MCFG 表**上报给 OS——思想相同、上报方式标准化。与 CF8/CFC 的关系：端口方式是 PCI 遗留机制（[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)），只能访问前 256 字节；只有 ECAM 能访问完整 4KB 配置空间和扩展能力，现代 OS 优先用 ECAM。

**Q5：MCHBAR、EPBAR、DMIBAR 这些 BAR 和普通设备的 BAR 一样吗？**
形式上都是"指向一段 MMIO 窗口的基地址寄存器"，但用途不同。普通设备 BAR 暴露**外设**的寄存器给驱动用；这些平台 BAR 暴露的是**平台自身的内部机构**（内存控制器、Egress Port、DMI 链路），给固件配置平台用。它们是"平台把自己的控制面板挂到地址空间上"——这个模式今天依然存在，只是寄存器藏进了 CPU。

**Q6：现代平台上还有"北桥/南桥"的说法吗？**
北桥基本消失了——它的功能（内存控制器、PCIe RC、显示输出）都进了 CPU。"南桥"演化为 **PCH（Platform Controller Hub）**，仍然承接 USB/SATA/慢速 I/O 和传统总线，通过片内/片间的 DMI 与 CPU 相连。所以现在更常说"CPU + PCH"两片（笔记本/低功耗 SoC 甚至把 PCH 也整合了）。

---

## 与 SPEC 7.0 章节对照

| 本章主题 | Base Spec 7.0 章节 / 相关规范 |
|----------|-------------------------------|
| ECAM / 基址钉定 | Enhanced Configuration Access Mechanism（[第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)） |
| ECAM 基址上报 | ACPI MCFG 表（[第14章](../第三篇-Linux与PCI/第14章-Linux-PCI初始化.md)） |
| Root Complex / Root Port | Root Complex（[第04章](第04章-PCIe总线概述.md)） |
| 地址空间 / MMIO / DRAM | Address Spaces / Memory Mapped I/O |
| 64 位 / 可预读 BAR / Resizable BAR | Base Address Registers / Resizable BAR（[第03章](../第一篇-PCI体系结构/第03章-PCI数据交换.md)） |
| 平台拓扑描述 | UEFI / ACPI（MCFG、DMAR、SRAT） |
| MCH/ICH 具体寄存器（历史案例） | Intel 芯片组数据手册（非 Base Spec 内容） |

> 说明：PCIEXBAR/MCHBAR/EPBAR/DMIBAR/TOLUD/TOUUD 属于 **Intel 芯片组数据手册**，非 PCIe Base Spec 内容，本章作历史案例讲；ECAM、地址空间、Resizable BAR 等**通用概念**在 Base Spec 有对应章节。具体以工程内 `NCB-PCI_Express_Base_7.0.pdf` 及相应平台手册为准。

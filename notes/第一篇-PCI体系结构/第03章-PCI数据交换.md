# 第03章　PCI 总线的数据交换

> **导读**：枚举完成后设备如何真正搬数据？本章讲 **BAR 空间初始化 → 译码 → DMA → Cache 一致性 → 预读**。其中 Cache 一致性与预读是「跨 PCI/PCIe 通用」的硬骨头，直接决定 DMA 的正确性与性能，PCIe 的 `No Snoop`/`Relaxed Ordering` 属性即源于此。
>
> **本页为大纲骨架**，正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| BAR | 基地址寄存器 | 设备申请的 MMIO/IO 窗口在总线域的位置与大小 |
| Positive / Subtractive Decode | 正向 / 负向译码 | 按窗口认领地址 / 无人认领时兜底认领 |
| Direct Memory Access | DMA | 设备作为 Master 直接读写主存 |
| Cache Coherency | Cache 一致性 | 保证 DMA 数据与 CPU Cache 内容一致 |
| Snoop | 侦听 | 硬件检查/使 Cache 行失效以维持一致性 |
| Prefetchable Memory | 可预读窗口 | 可被桥预读、合并、无副作用的存储器区 |
| NS / RO | No Snoop / Relaxed Ordering | PCIe TLP 属性位，放宽一致性/排序以提性能 |
| Atomic Op | 原子操作 | PCIe 的 FetchAdd/Swap/CAS 事务 |

---

## 3.1 PCI 设备 BAR 空间的初始化
- **3.1.1 存储器地址与 PCI 总线地址的转换**：BAR 决定设备在 PCI 域占用的窗口。
- **3.1.2 BAR 寄存器与桥 Base/Limit 寄存器的初始化**：⭐**写全 1 探测大小 → 回写基址**的经典流程；桥窗口须覆盖其下所有设备。

## 3.2 PCI 设备的数据传递
- **3.2.1 正向译码与负向译码**：谁认领这个地址 —— positive（按窗口认领）vs subtractive（兜底，通常 ISA 桥）。
- **3.2.2 处理器到 PCI 设备的数据传送**：outbound，MMIO 写/读。
- **3.2.3 PCI 设备的 DMA 操作**：inbound，设备做 Master 读写主存。
- **3.2.4 PCI 桥的 Combining / Merging / Collapsing**：桥对写事务的合并优化及其**允许/禁止规则**（Collapsing 一般禁止，会丢语义）。

## 3.3 与 Cache 相关的 PCI 总线事务
- **3.3.1 Cache 一致性的基本概念**：MESI、snoop、write-back/write-through。
- **3.3.2 对不可 Cache 空间进行 DMA 读写**：无需 snoop 的简单情形。
- **3.3.3 对可 Cache 空间进行 DMA 读写**：需硬件保证一致性。
- **3.3.4 DMA 写时发生 Cache 命中**：⭐一致性关键场景（invalidate vs update）。
- **3.3.5 DMA 写时命中的优化**：减少 snoop 开销的手段。

## 3.4 预读机制
- **3.4.1 指令 Fetch** / **3.4.2 数据预读** / **3.4.3 软件预读** / **3.4.4 硬件预读**：从 CPU 侧建立「预读」概念。
- **3.4.5 PCI 总线的预读机制**：**Prefetchable memory** 属性 —— 桥可预读、可合并、无副作用的存储器窗口（对应 BAR 的 prefetch 位）。

## 3.5 小结
- DMA 正确性 = 地址转换正确 + Cache 一致性正确；性能 = 合并 + 预读。

---

## 📘 SPEC 7.0 现代化补充清单
- **一致性属性显式化**：PCIe TLP 头用 `Attr` 位携带 **No Snoop** 与 **Relaxed Ordering**（源自本章 Cache/预读思想），见 [第06章](../第二篇-PCIe体系结构/第06章-事务层.md)/[第11章](../第二篇-PCIe体系结构/第11章-总线的序.md)。
- **Prefetchable + 64 位 BAR**：现代设备（大显存/大 BAR）普遍用 64 位可预读 BAR，配合 **Resizable BAR**。
- **原子操作**：PCIe 增加 `FetchAdd/Swap/CAS` 原子事务（[第06章](../第二篇-PCIe体系结构/第06章-事务层.md)），补足 PCI 时代缺失的一致性原语。
- **DMA 与 IOMMU**：现代 DMA 地址是「IOVA」，经 IOMMU 翻译（[第13章](../第二篇-PCIe体系结构/第13章-虚拟化技术.md)），与本章「PCI 总线地址」形成对照。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第03章-BAR探测.svg` | BAR 探测与地址转换 | 写全 1 → 读回掩码 → 求大小 → 回写基址 四步 |
| `第03章-正负向译码.svg` | 正向 vs 负向译码 | 地址落入窗口的认领判定流程 |
| `第03章-DMA与Cache一致性.svg` | DMA 写命中 Cache | 设备 DMA 写 → snoop → invalidate/回写 时序 |
| `第03章-预读.svg` | 可预读窗口 | Prefetchable 与 non-prefetchable 桥窗口行为对比 |

## 与 SPEC 7.0 章节对照（占位）
| 本章主题 | Base Spec 7.0 章节 |
|----------|--------------------|
| No Snoop / Relaxed Ordering | （填充：TLP Attr、Transaction Ordering）|
| 原子操作 | （填充：Atomic Operations）|
| Resizable BAR | （填充：Resizable BAR Capability）|

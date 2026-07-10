# 第15章　Linux PCI 的中断处理

> **导读**：设备的中断如何最终变成 CPU 上运行的 handler？本章讲两条路径：传统 **INTx 中断路由**（irq 号从哪来），以及现代 **MSI/MSI-X 向量申请**。是把 [第10章](../第二篇-PCIe体系结构/第10章-MSI和MSI-X中断.md) 硬件机制落到内核 API 的收尾章。
>
> **现代化提示**：现代内核以 **irqdomain / 层次化中断域**统一管理，MSI 走通用框架。
>
> **本页为大纲骨架**，正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| irq | 中断号 | 内核层面标识一个中断源的编号 |
| ACPI `_PRT` | PCI 中断路由表 | 把 (设备, INTA-D) 映射到 GSI |
| GSI | 全局系统中断 | ACPI 下统一编号的中断线 |
| INTx Swizzle | swizzle | INTx 经桥旋转的路由规则（呼应第01章） |
| IRQ Affinity | 中断亲和 | 把中断向量绑定到特定 CPU |
| irqdomain | 中断域 | 内核层次化管理中断映射的框架 |
| `pci_alloc_irq_vectors` | 统一向量申请 | 自动在 MSI-X/MSI/INTx 间选型 |
| Managed IRQ | 托管中断 | 内核按队列自动分配并绑核的中断 |

---

## 15.1 PCI 总线的中断路由
- **15.1.1 PCI 设备如何获取 irq 号**：`dev->irq` 的来源；`pci_enable_device` 后的 irq 绑定。
- **15.1.2 PCI 中断路由表**：⭐ACPI `_PRT`（PCI Routing Table）把 `(设备, INTA-D)` 映射到 GSI（呼应 [第14章](第14章-Linux-PCI初始化.md) DSDT）；INTx swizzle（呼应 [第01章](../第一篇-PCI体系结构/第01章-PCI总线基础.md)）。
- **15.1.3 PCI 插槽使用的 irq 号**：物理槽位与中断线的对应。

## 15.2 使用 MSI/MSI-X 中断机制申请中断向量
- **15.2.1 Linux 如何使能 MSI 中断机制**：`pci_enable_msi`（旧）分配连续向量、写 Capability。
- **15.2.2 Linux 如何使能 MSI-X 中断机制**：⭐`pci_enable_msix` 按 MSI-X Table 分配独立向量、设置亲和（呼应 [第10章](../第二篇-PCIe体系结构/第10章-MSI和MSI-X中断.md)）。

## 15.3 小结
- 「_PRT/GSI（INTx）」与「MSI-X 向量池」两条路径的取舍：现代设备一律优先 MSI-X。

---

## 📘 现代内核 / SPEC 对照清单
- **统一 API**：`pci_alloc_irq_vectors(dev, min, max, PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX)` 取代旧的 `pci_enable_msi/msix`，一把接口自动降级选型；`pci_irq_vector()` 取向量。
- **irqdomain 层次化**：MSI 经 `irq_domain` → 中断重映射域（VT-d/AMD-Vi，呼应 [第13章](../第二篇-PCIe体系结构/第13章-虚拟化技术.md)）→ vector 域，逐层分配与亲和。
- **多队列亲和**：`irq_set_affinity_hint` / 托管中断（managed IRQ）把每队列向量绑到 CPU，配合 NVMe/网卡多队列。
- **IMS**：超大规模虚拟设备的中断存储（呼应 [第10章](../第二篇-PCIe体系结构/第10章-MSI和MSI-X中断.md) / [第13章](../第二篇-PCIe体系结构/第13章-虚拟化技术.md)）。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第15章-INTx路由表.svg` | INTx → GSI 路由 | _PRT 把 (Dev,INTx) 映射到 GSI，swizzle 示意 |
| `第15章-MSI使能流程.svg` | MSI-X 使能流程 | pci_alloc_irq_vectors → 写 Table → request_irq → handler |
| `第15章-irqdomain层次.svg` | irqdomain 层次 | PCI-MSI 域→重映射域→vector 域 的分配链 |

## 与 SPEC / 规范对照（占位）
| 本章主题 | 规范 / 内核 |
|----------|-------------|
| INTx 路由 / _PRT | ACPI 规范；Base Spec Message（INTx 模拟）|
| MSI / MSI-X | Base Spec MSI/MSI-X；Linux `drivers/pci/msi/` |

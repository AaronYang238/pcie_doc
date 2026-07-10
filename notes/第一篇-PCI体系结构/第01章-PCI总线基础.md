# 第01章　PCI 总线的基本知识

> **导读**：PCIe 是 PCI 的「串行化 + 分层化」演进，但**软件模型几乎完全继承自 PCI**。本章打地基 —— 讲清 PCI 的组成、信号、总线事务、Posted/Non-Posted 语义与中断机制，这些概念在后续所有 PCIe 章节都会复用。
>
> **本页为大纲骨架**（术语表 + 小节提要 + 配图清单 + 现代化补充清单 + SPEC 对照占位），正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| Host Bridge | HOST 主桥 | 连接处理器/存储器域与 PCI 总线域的枢纽 |
| Master / Target | 发起方 / 目标 | 总线事务的发起方 / 响应方 |
| Posted | Posted 事务 | 发出后不等完成响应（如存储器写） |
| Non-Posted | Non-Posted 事务 | 需要完成响应（如读、I/O 写、配置写） |
| Delayed Transaction | Delayed 事务 | PCI 对 Non-Posted 读的「延迟」处理方式 |
| Arbitration | 仲裁 | `REQ#`/`GNT#` 决定谁占用共享总线 |
| INTA#–INTD# | INTx 中断 | 电平触发、低有效、可共享的物理中断线 |
| Split Transaction | Split 事务 | PCI-X 中接收方主动回完成，是 PCIe 请求/完成模型的前身 |

---

## 1.1 PCI 总线的组成结构
- **1.1.1 HOST 主桥**：连接处理器/存储器域与 PCI 总线域的枢纽；负责地址空间转换与配置请求发起。
- **1.1.2 PCI 总线**：并行、多设备共享的总线；讲「共享介质 + 仲裁」的本质，与 PCIe「端到端」形成对照埋点。
- **1.1.3 PCI 设备**：Master/Target 角色；单功能 vs 多功能设备。
- **1.1.4 HOST 处理器**：作为发起者如何看待 PCI 地址空间。
- **1.1.5 PCI 总线的负载**：电气负载与扇出限制 —— 正是这个物理瓶颈催生了 PCIe 串行链路。

## 1.2 PCI 总线的信号定义
- **1.2.1 地址和数据信号**：`AD[31:0]` 地址/数据复用、`C/BE#`、`PAR`。
- **1.2.2 接口控制信号**：`FRAME#`、`IRDY#`、`TRDY#`、`DEVSEL#`、`STOP#` 的握手含义。
- **1.2.3 仲裁信号**：`REQ#`/`GNT#` 与中央仲裁器。
- **1.2.4 中断请求等其他信号**：`INTA#–INTD#`（电平触发、低有效、共享）。

## 1.3 PCI 总线的存储器读写总线事务
- **1.3.1 PCI 总线事务的时序**：地址期 + 数据期；`IRDY#/TRDY#` 插入等待周期。
- **1.3.2 Posted 和 Non-Posted 传送方式**：⭐核心概念 —— Posted（写，不等完成）vs Non-Posted（读/部分写，需完成回应）。**这是理解 PCIe TLP 分类与「序」的根基**。
- **1.3.3 HOST 处理器访问 PCI 设备**：下行（outbound）路径。
- **1.3.4 PCI 设备读写主存储器**：上行（DMA）路径。
- **1.3.5 Delayed 传送方式**：PCI 对 Non-Posted 读的「延迟事务」处理，及其局限（为 PCIe Split 事务铺垫）。

## 1.4 PCI 总线的中断机制
- **1.4.1 中断信号与中断控制器的连接关系**：INTx → PIC/APIC。
- **1.4.2 中断信号与 PCI 总线的连接关系**：INTx 的 swizzle（跨桥旋转）路由。
- **1.4.3 中断请求的同步**：电平触发共享中断的去抖与同步问题。

## 1.5 PCI-X 总线简介
- **1.5.1 Split 总线事务**：用 Split 替代 Delayed，接收方主动回完成 —— **直接对应 PCIe 的 Request/Completion 模型**。
- **1.5.2 总线传送协议**：属性阶段、按序规则的强化。
- **1.5.3 基于数据块的突发传送**：突发长度协商。

## 1.6 小结
- PCI 的「共享并行总线 + INTx + Posted/Non-Posted + 配置空间」四大要素如何被 PCIe 继承或替换。

---

## 📘 SPEC 7.0 现代化补充清单
- **共享总线 → 端到端链路**：PCIe 无仲裁、无 `DEVSEL#`；点对点全双工，用交换（Switch）替代共享介质。
- **INTx → MSI/MSI-X → （现代）IMS**：物理中断线退化为「虚拟 INTx 消息」；主流已是 MSI-X（详见 [第10章](../第二篇-PCIe体系结构/第10章-MSI和MSI-X中断.md)）。
- **Posted/Non-Posted 概念保留**：在 PCIe 中演化为 TLP 的三类（Posted / Non-Posted / Completion），并成为「序」规则（[第11章](../第二篇-PCIe体系结构/第11章-总线的序.md)）的基础。
- **Delayed → Split → PCIe Completion**：延迟事务的思想在 PCIe 中以 Completion TLP 完整实现。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第01章-PCI拓扑.svg` | PCI 总线拓扑 | HOST 主桥 + 共享 PCI 总线 + 多设备 + PCI 桥分层 |
| `第01章-读事务时序.svg` | PCI 存储器读时序 | FRAME#/IRDY#/TRDY#/DEVSEL# 波形，标注地址期与数据期、等待周期 |
| `第01章-Posted对比.svg` | Posted vs Non-Posted | 两条时间线对比：写「发后不管」vs 读「等完成」 |
| `第01章-INTx路由.svg` | INTx swizzle 路由 | INTA#–INTD# 经 PCI 桥旋转到中断控制器 |

## 与 SPEC 7.0 章节对照（占位）
| 本章主题 | Base Spec 7.0 章节 |
|----------|--------------------|
| 事务类型 / Posted 语义 | （填充：Transaction Ordering、TLP 章） |
| 中断（INTx 模拟） | （填充：Message 相关章）|

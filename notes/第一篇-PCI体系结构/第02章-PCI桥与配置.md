# 第02章　PCI 总线的桥与配置

> **导读**：本章讲两件事 —— **地址空间怎么划分域并相互转换**，以及 **PCI 设备怎么被枚举与配置**。配置空间、Type 0/1 配置请求、总线号/设备号分配这套机制，PCIe **原样继承**（只是把访问方式从端口 I/O 换成 ECAM 内存映射），是理解一切枚举与 BAR 分配的前提。
>
> **本页为大纲骨架**，正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| Memory / PCI Bus Domain | 存储器域 / PCI 总线域 | 同一物理地址在不同「域」中的含义不同 |
| Configuration Space | 配置空间 | 每设备 256B（PCIe 扩展到 4KB）的即插即用寄存器区 |
| Type 0 / 1 Header | Type 00h / 01h 头 | 普通设备头 / 桥设备头 |
| BAR, Base Address Register | 基地址寄存器 | 设备申请 MMIO/IO 窗口的寄存器 |
| DFS Enumeration | 深度优先枚举 | 递归扫描总线树、分配 Bus 号的过程 |
| Enhanced Config Access Mechanism | ECAM | 把 Bus/Dev/Fun/Reg 映射为内存地址访问配置空间 |
| NTB, Non-Transparent Bridge | 非透明桥 | 两侧各自枚举、经地址翻译窗口互联的桥 |
| Alternative Routing-ID | ARI | 突破每设备 8 功能限制（配合 SR-IOV） |

---

## 2.1 存储器域与 PCI 总线域
- **2.1.1 CPU 域、DRAM 域与存储器域**：为什么要区分「域」—— 同一物理地址在不同域含义不同。
- **2.1.2 PCI 总线域**：PCI 设备眼中的地址空间。
- **2.1.3 处理器域**：CPU 视角，及经 HOST 主桥的映射关系。

## 2.2 HOST 主桥
- **2.2.1 PCI 设备配置空间的访问机制**：`CONFIG_ADDRESS`(0xCF8)/`CONFIG_DATA`(0xCFC) 端口机制。
- **2.2.2 存储器域 → PCI 总线域 的转换**：outbound 窗口。
- **2.2.3 PCI 总线域 → 存储器域 的转换**：inbound 窗口（DMA 目标）。
- **2.2.4 x86 处理器的 HOST 主桥**：北桥/MCH 集成实例。

## 2.3 PCI 桥与 PCI 设备的配置空间
- **2.3.1 PCI 桥**：Primary/Secondary/Subordinate 号；桥的地址过滤窗口（Memory Base/Limit 等）。
- **2.3.2 PCI Agent 设备的配置空间**：⭐**Type 00h 头**，256 字节；Vendor/Device ID、Command/Status、6 个 BAR、Capabilities 指针。
- **2.3.3 PCI 桥的配置空间**：**Type 01h 头**，含 Bus Number 与 Base/Limit 窗口寄存器。

## 2.4 PCI 总线的配置
- **2.4.1 Type 01h 和 Type 00h 配置请求**：桥如何根据目标 Bus 号决定「透传」还是「转 Type0」。
- **2.4.2 PCI 总线配置请求的转换原则**：Type1→Type0 转换规则。
- **2.4.3 PCI 总线树 Bus 号的初始化**：⭐**深度优先枚举（DFS）**分配 Bus 号。
- **2.4.4 PCI 总线 Device 号的分配**：IDSEL 与 Device 号的对应。

## 2.5 非透明 PCI 桥
- **2.5.1 Intel 21555 中的配置寄存器**：NTB 两侧各一套。
- **2.5.2 通过非透明桥片进行数据传递**：地址翻译窗口 + 门铃/scratchpad，用于双主机互联。

## 2.6 小结
- 「配置空间 + Type0/1 请求 + DFS 枚举」= PCIe 即插即用的地基。

---

## 📘 SPEC 7.0 现代化补充清单
- **配置访问：CF8/CFC → ECAM**：PCIe 用 **Enhanced Configuration Access Mechanism**，把 `Bus/Device/Function/Offset` 直接映射为内存地址，取代端口机制。
- **配置空间：256B → 4KB**：PCIe 扩展配置空间（0x100–0xFFF）承载 **Extended Capabilities**（AER、SR-IOV、ATS…）。
- **枚举模型不变**：Type 0/1 头、Base/Limit、DFS 枚举在 PCIe 完整保留（RC/Switch/Endpoint 仍以 PCI-PCI 桥语义呈现）。
- **ARI**：Alternative Routing-ID，突破每设备 8 功能限制（配合 SR-IOV，见 [第13章](../第二篇-PCIe体系结构/第13章-虚拟化技术.md)）。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第02章-地址域.svg` | 存储器域 vs PCI 总线域 | 三域并列 + HOST 主桥 inbound/outbound 双向翻译箭头 |
| `第02章-配置头Type0.svg` | Type 00h 配置头布局 | 256 字节字段表，突出 ID/Command/BAR0-5/Cap 指针 |
| `第02章-配置头Type1.svg` | Type 01h 桥配置头 | 突出 Primary/Secondary/Subordinate、Base/Limit 窗口 |
| `第02章-Bus枚举DFS.svg` | Bus 号 DFS 枚举 | 总线树 + 编号顺序步骤标注 |
| `第02章-非透明桥.svg` | 非透明桥数据通路 | 两侧主机 + 地址翻译窗口 + 门铃 |

## 与 SPEC 7.0 章节对照（占位）
| 本章主题 | Base Spec 7.0 章节 |
|----------|--------------------|
| 配置空间 / ECAM | （填充：Configuration Space、Enhanced Config Access）|
| 扩展 Capability | （填充：Extended Capabilities）|
| ARI | （填充：ARI）|

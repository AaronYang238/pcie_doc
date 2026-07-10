# 第12章　PCIe 总线的应用

> **导读**：前面各层的知识，在这一章「合龙」。原书用一块虚拟示例卡 **Capric 卡**，从硬件工作原理 → TLP 数据流 → Linux 驱动 → 延时/带宽分析，走通一个完整设备的生命周期。这是把 第04–11章 串起来的实战章。
>
> **现代化补充**：用当代速率（Gen4/5/6）重算带宽与延时数量级。
>
> **本页为大纲骨架**，正文按 [CLAUDE.md §5.1](../../CLAUDE.md) 骨架填充。

---

## 术语表

| 英文 / 缩写 | 中文 | 一句话释义 |
|-------------|------|-----------|
| — | Capric 卡 | 原书用于贯穿讲解的虚拟示例 PCIe 设备 |
| DMA Engine | DMA 引擎 | 设备内主动搬运数据的逻辑 |
| Descriptor | 描述符 | 描述一次 DMA 的地址/长度等的内存结构 |
| Effective Bandwidth | 有效带宽 | 载荷/(载荷+开销)，与 MPS 正相关 |
| Outstanding Requests | 在途请求数 | 未完成的 Non-Posted 请求个数（受 Tag 限制） |
| Coherent / Streaming DMA | 一致性/流式映射 | 内核 DMA 映射的两种模型 |
| Top/Bottom Half | 中断上/下半部 | 中断处理的即时部分与延后部分 |
| I/O Virtual Address | IOVA | IOMMU 下设备看到的虚拟 DMA 地址 |

---

## 12.1 Capric 卡的工作原理
- **12.1.1 BAR 空间**：寄存器 BAR + 数据缓冲 BAR 的划分。
- **12.1.2 Capric 卡的初始化**：使能 Memory Space、设置 MPS/MRRS、配置 MSI。
- **12.1.3 DMA 写** / **12.1.4 DMA 读**：设备作为 Requester 读写主存的寄存器编程模型。
- **12.1.5 中断请求**：完成后发 MSI（呼应 [第10章](第10章-MSI和MSI-X中断.md)）。

## 12.2 Capric 卡的数据传递
- **12.2.1 DMA 写使用的 TLP**：⭐MWr TLP 序列（地址、Length、按 MPS 分片）。
- **12.2.2 DMA 读使用的 TLP**：MRd 发出 → 多个 CplD 返回（按 RCB 拆分，呼应 [第06章](第06章-事务层.md)）。
- **12.2.3 Capric 卡的中断请求**：MSI 写 TLP 与数据写的保序（呼应 [第11章](第11章-总线的序.md)）。

## 12.3 基于 PCIe 总线的设备驱动
- **12.3.1 驱动程序的加载与卸载**：`probe/remove`、`pci_enable_device`、BAR `ioremap`。
- **12.3.2 初始化与关闭**：DMA mask、中断申请。
- **12.3.3 DMA 读写操作**：一致性/流式 DMA 映射、`dma_map_single`。
- **12.3.4 中断处理**：MSI handler、top/bottom half。
- **12.3.5 存储器地址到 PCI 总线地址的转换**：驱动视角的地址域（呼应 [第02章](../第一篇-PCI体系结构/第02章-PCI桥与配置.md)）。
- **12.3.6 存储器与 Cache 的同步**：`dma_sync_*`（呼应 [第03章](../第一篇-PCI体系结构/第03章-PCI数据交换.md)）。

## 12.4 Capric 卡的延时与带宽
- **12.4.1 TLP 的传送开销**：⭐头/成帧/DLLP/编码开销 → **有效带宽 = 载荷/(载荷+开销)**；MPS 越大效率越高。
- **12.4.2 PCIe 设备的 DMA 读写延时**：读延时 = 往返 + Completer 处理；在途请求数（Tag）× MPS 决定吞吐（小 Tag 会限速）。
- **12.4.3 Capric 卡的优化**：增大 MPS/MRRS、增加在途请求、合并描述符、RO/IDO。

## 12.5 小结
- 一张「一次 DMA 的完整 TLP 时序 + 带宽账」收束全篇。

---

## 📘 SPEC 7.0 现代化补充清单
- **带宽重算**：用 128b/130b（~1.5% 编码开销）与 Gen4/5/6 速率重算有效带宽，替换原书 8b/10b（20% 开销）的数字。
- **在途并发**：**10-bit/14-bit Tag** 让高带宽×高延迟链路能填满管道（原书 8-bit Tag 会成为瓶颈）。
- **驱动现代化**：`pci_alloc_irq_vectors()` 统一 MSI/MSI-X 申请；`dma_map_*` 在 IOMMU 下返回 IOVA（呼应 [第13章](第13章-虚拟化技术.md)）。
- **Flit 模式开销模型**：Gen6+ 的开销结构（FEC/CRC 固定占比）与非 Flit 不同，带宽账需相应调整。

## 配图清单（SVG）
| 文件 | 图注 | 画什么 |
|------|------|--------|
| `第12章-Capric框图.svg` | Capric 卡框图 | 寄存器/缓冲 BAR + DMA 引擎 + 中断单元 |
| `第12章-DMA写TLP.svg` | DMA 写 TLP 序列 | 描述符→MWr 分片→MSI 的时间线 |
| `第12章-DMA读TLP.svg` | DMA 读 TLP 序列 | MRd→多 CplD(按 RCB)→完成→MSI |
| `第12章-带宽分解.svg` | 有效带宽分解 | 载荷 vs 头/DLLP/编码开销 的堆叠占比 + MPS 影响曲线 |

## 与 SPEC 7.0 章节对照（占位）
| 本章主题 | Base Spec 7.0 章节 |
|----------|--------------------|
| TLP 开销 / 带宽 | （填充：Transaction Layer / Performance）|
| Tag 扩展 | （填充：10-bit/14-bit Tag）|

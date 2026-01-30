# AXI (Advanced eXtensible Interface) 总线协议深度学习笔记

> 本笔记基于 ARM AMBA AXI4 Specification (IHI0022E) 整理。旨在提供极尽详细的参考，涵盖信号定义、时序关系、复杂的地址计算逻辑及乱序机制。

## 目录
1. [协议概述](#1-协议概述)
2. [接口信号详解](#2-接口信号详解)
    * [全局信号](#21-全局信号)
    * [写地址通道 (AW)](#22-写地址通道-aw)
    * [写数据通道 (W)](#23-写数据通道-w)
    * [写响应通道 (B)](#24-写响应通道-b)
    * [读地址通道 (AR)](#25-读地址通道-ar)
    * [读数据通道 (R)](#26-读数据通道-r)
3. [核心机制：握手协议](#3-核心机制握手协议)
    * [握手规则](#31-握手规则)
    * [通道间的依赖关系](#32-通道间的依赖关系)
4. [高级传输特性：Burst计算与地址对齐](#4-高级传输特性burst计算与地址对齐)
    * [AxSIZE 与 AxLEN](#41-axsize-与-axlen)
    * [AxBURST 突发类型](#42-axburst-突发类型)
    * [地址计算公式 (核心算法)](#43-地址计算公式-核心算法)
    * [窄传输 (Narrow Transfers)](#44-窄传输-narrow-transfers)
    * [非对齐传输 (Unaligned Transfers)](#45-非对齐传输-unaligned-transfers)
5. [数据通道细节](#5-数据通道细节)
    * [WSTRB 选通信号生成](#51-wstrb-选通信号生成)
    * [字节不变性 (Byte Invariance)](#52-字节不变性-byte-invariance)
6. [事务排序与乱序 (Ordering & Out-of-Order)](#6-事务排序与乱序-ordering--out-of-order)
    * [ID 信号的作用](#61-id-信号的作用)
    * [读写顺序模型](#62-读写顺序模型)
7. [原子访问 (Atomic Accesses)](#7-原子访问-atomic-accesses)
    * [独占访问 (Exclusive Access)](#71-独占访问-exclusive-access)
    * [锁定访问 (Locked Access)](#72-锁定访问-locked-access)
8. [AXI 版本变体对比](#8-axi-版本变体对比)

---

## 1. 协议概述

AXI (Advanced eXtensible Interface) 是 AMBA 3.0 (2003年) 和 AMBA 4.0 (2010年) 规范中定义的片上总线标准。

**主要特性：**
*   **分离的地址/控制和数据阶段**：地址和数据可以使用不同的总线带宽。
*   **支持非对齐数据传输**：通过 Byte Strobe 信号支持任意字节偏移的访问。
*   **基于突发 (Burst) 的传输**：只需发送首地址，后续数据地址自动计算。
*   **独立的读写数据通道**：读写可并发进行 (Full Duplex)，带宽翻倍。
*   **支持乱序 (Out-of-Order) 传输**：通过 Transaction ID 区分不同事务，允许慢速设备不阻塞快速设备。
*   **易于时序收敛**：所有通道单向传输，支持 Register Slice 插入。

---

## 2. 接口信号详解

AXI 定义了 5 个独立通道。以下信号命名中，`x` 代表通道前缀 (AW/W/B/AR/R)。

### 2.1 全局信号
| 信号 | 来源 | 极性 | 描述 | 注意事项 |
|:---|:---|:---|:---|:---|
| `ACLK` | Clock | Rising Edge | 全局时钟 | 所有输入信号在上升沿采样，所有输出在上升沿变化。 |
| `ARESETn` | Reset | Active Low | 全局复位 | **异步复位，同步释放**。复位期间 `VALID` 信号必须为低，其他信号无要求。 |

### 2.2 写地址通道 (AW)
负责传输写操作的元数据（Metadata）。

| 信号 | 位宽 | 来源 | 描述 | 详细说明 |
|:---|:---|:---|:---|:---|
| `AWID` | ID_WIDTH | Master | 事务ID | 用于标记组别，同一ID的写事务必须顺序完成，不同ID可乱序。 |
| `AWADDR`| ADDR_WIDTH | Master | 写首地址 | 突发传输中第一个字节的地址。 |
| `AWLEN` | 8 (AXI4) / 4 (AXI3) | Master | 突发长度 | **实际传输次数 = AWLEN + 1**。<br>AXI3: 1~16 transfers<br>AXI4: 1~256 transfers (INCR only) |
| `AWSIZE`| 3 | Master | 突发大小 | 每次传输（Beat）的字节数。编码如下：<br>000=1B, 001=2B, 010=4B, 011=8B, 100=16B... 111=128B<br>**不能超过数据总线宽度。** |
| `AWBURST`| 2 | Master | 突发类型 | 00=FIXED (FIFO), 01=INCR (RAM), 10=WRAP (Cache), 11=Reserved |
| `AWLOCK`| 1 (AXI4) / 2 (AXI3) | Master | 锁类型 | AXI4: 0=Normal, 1=Exclusive<br>AXI3: 00=Normal, 01=Exclusive, 10=Locked |
| `AWCACHE`| 4 | Master | 存储属性 | [3:0]分别代表：Bufferable, Cacheable, Read-allocate, Write-allocate。<br>例如 `4'b0000`=Device Non-bufferable, `4'b0011`=Bufferable Cacheable。 |
| `AWPROT`| 3 | Master | 保护类型 | [0]=Privileged/Normal, [1]=Secure/Non-secure, [2]=Instruction/Data |
| `AWQOS` | 4 | Master | QoS | AXI4新增。0-15用于仲裁优先级控制，协议未定义具体策略。 |
| `AWREGION`| 4 | Master | 区域 | AXI4新增。用于同一个Slave内的多地址段解码。 |
| `AWVALID`| 1 | Master | 有效 | 指示地址控制信号有效。**不能等待 AWREADY**。 |
| `AWREADY`| 1 | Slave | 准备好 | 指示 Slave 已准备好接收地址。 |

### 2.3 写数据通道 (W)
负责传输具体的写数据。

| 信号 | 位宽 | 来源 | 描述 | 详细说明 |
|:---|:---|:---|:---|:---|
| `WID` | ID_WIDTH | Master | 写ID | **仅在 AXI3 存在**。AXI4 中已移除，要求写数据必须严格按照写地址的顺序发送。 |
| `WDATA` | DATA_WIDTH | Master | 写数据 | 通常为 32, 64, 128, 256 bits。 |
| `WSTRB` | DATA_WIDTH/8 | Master | 写选通 | **关键信号**。每一位对应 WDATA 的一个字节。1=该字节有效，0=该字节无效（不写入）。用于掩码操作。 |
| `WLAST` | 1 | Master | 最后一拍 | 指示这是当前 Burst 的最后一次传输。 |
| `WVALID`| 1 | Master | 有效 | 指示数据有效。 |
| `WREADY`| 1 | Slave | 准备好 | 指示 Slave 准备好接收数据。 |

### 2.4 写响应通道 (B)
Master 发起写，Slave 必须回复一个响应，告知“写成功了没”。**注意：响应是针对整个 Burst 的，而不是针对每一个 Beat。**

| 信号 | 位宽 | 来源 | 描述 | 详细说明 |
|:---|:---|:---|:---|:---|
| `BID` | ID_WIDTH | Slave | 响应ID | 必须与对应的 `AWID` 匹配。Master 据此知道是哪个写事务完成了。 |
| `BRESP` | 2 | Slave | 响应状态 | 00=OKAY, 01=EXOKAY (独占成功), 10=SLVERR (从机错误), 11=DECERR (解码错误)。 |
| `BVALID`| 1 | Slave | 有效 | 指示响应有效。 |
| `BREADY`| 1 | Master | 准备好 | 指示 Master 准备好接收响应。 |

### 2.5 读地址通道 (AR)
定义与 AW 通道几乎完全一样，只是前缀变为 `AR`。

### 2.6 读数据通道 (R)
负责返回读出的数据和读状态。

| 信号 | 位宽 | 来源 | 描述 | 详细说明 |
|:---|:---|:---|:---|:---|
| `RID` | ID_WIDTH | Slave | 读ID | 必须与 `ARID` 匹配。Master 通过 RID 把乱序回来的数据拼回去。 |
| `RDATA` | DATA_WIDTH | Slave | 读数据 | 读出的数据。 |
| `RRESP` | 2 | Slave | 读状态 | 00=OKAY, 01=EXOKAY, 10=SLVERR, 11=DECERR。<br>**注意**：与写响应不同，RRESP 是跟着每一个 Data Beat 回来的（因为可能中途出错）。 |
| `RLAST` | 1 | Slave | 最后一拍 | 指示这是当前 Burst 的最后一个 Data Beat。 |
| `RVALID`| 1 | Slave | 有效 | 指示数据有效。 |
| `RREADY`| 1 | Master | 准备好 | Master 准备好接收数据。 |

---

## 3. 核心机制：握手协议

### 3.1 握手规则
所有 5 个通道均采用 `VALID/READY` 握手。
1.  **Source 产生 VALID**：指明地址/数据已置于总线上且平稳。
2.  **Destination 产生 READY**：指明它已准备好接收。
3.  **传输生效**：当 `VALID` 和 `READY` 在时钟上升沿**同时为高**时，传输完成。

**不允许的死锁依赖：**
*   **VALID 绝不能等待 READY**：Source 必须在自己准备好后拉高 VALID，不能等看到 READY 高了才拉 VALID。否则如果 Destination 也在等 VALID 变高才拉 READY，就会死锁。
*   **READY 可以等待 VALID**：Destination 可以默认拉低 READY，看到 VALID 来了再拉高；也可以默认拉高 READY（推荐，效率高）。

### 3.2 通道间的依赖关系
除了通道内的握手，通道之间也有时序约束：

1.  **读事务依赖**：
    *   Slave 必须先看到 `ARVALID` & `ARREADY` (地址握手成功)，才能发出 `RVALID` (读数据)。
2.  **写事务依赖**：
    *   Slave 必须先看到 `AWVALID` & `AWREADY` (写地址握手) **并且** 看到最后的 `WVALID` & `WREADY` & `WLAST` (数据传完)，才能发出 `BVALID` (写响应)。
    *   **注意**：写地址 (AW) 和写数据 (W) 通道之间**没有**严格的先后依赖。Master 可以先发数据再发地址，或者发地址的同时发数据（只要 Slave 支持）。

---

## 4. 高级传输特性：Burst计算与地址对齐

这是 AXI 协议中最复杂也是最重要的部分。

### 4.1 AxSIZE 与 AxLEN
*   **Beat**: 一次单一的数据传输（由时钟沿上的 VALID & READY 触发）。
*   **Burst**: 由多个 Beat 组成的一个完整的 AXI 事务。
*   `AxLEN`:  Burst 长度 = `AxLEN + 1`。
*   `AxSIZE`: 每个 Beat 的字节数 = `2 ^ AxSIZE`。

**总传输字节数 (Total Bytes) = (AxLEN + 1) × (2 ^ AxSIZE)** （仅针对 INCR/FIXED 且对齐的情况）。

### 4.2 AxBURST 突发类型

| 编码 | 类型 | 描述 | 应用场景 |
|:---|:---|:---|:---|
| 2'b00 | **FIXED** | 地址固定不变。所有 Data Beat 写/读同一个地址。 | FIFO 访问 |
| 2'b01 | **INCR** | 地址递增。`Addr[n] = Addr[n-1] + Beat_Size`。 | 访问 RAM, DMA |
| 2'b10 | **WRAP** | 回环突发。地址递增，达到上边界后跳回下边界。 | Cache Line 填充 |

**WRAP 突发的特殊规则**：
*   长度必须是 2, 4, 8, 16。
*   起始地址必须按 total size 对齐。

### 4.3 地址计算公式 (核心算法)

假设：
*   `Start_Addr` = `AxADDR` (起始地址)
*   `Number_Bytes` = `2 ^ AxSIZE` (每拍字节数)
*   `Burst_Len` = `AxLEN + 1` (总拍数)

**计算变量：**
1.  **对齐起始地址** (Aligned_Address):
    `Aligned_Address = (Start_Addr / Number_Bytes) * Number_Bytes` (向下取整)
2.  **第一个 Beat 传输的字节数**:
    *   如果地址对齐：`Start_Addr` 指向 `Number_Bytes` 的边界，传输所有字节。
    *   如果地址不对齐：传输从 `Start_Addr` 到下一个对齐边界的数据。
3.  **后续 Beat 的地址 (INCR)**:
    `Address_N = Aligned_Address + (N * Number_Bytes)`

**示例计算**：
> `AxADDR = 0x1002`, `AxSIZE = 2 (4 bytes)`, `AxBURST = INCR`.
*   **Beat 0**: 地址 `0x1002`。因为是非对齐，且数据宽 4 字节，对齐地址是 `0x1000`。
    *   实际上只传输 `0x1002`, `0x1003` 两个字节（取决于 WSTRB）。
*   **Beat 1**: 地址 = `0x1000 + 1*4` = `0x1004`。
*   **Beat 2**: 地址 = `0x1000 + 2*4` = `0x1008`。

### 4.4 窄传输 (Narrow Transfers)
当 `Burst_Size` < `Data_Bus_Width` 时，称为窄传输。
例如：64-bit 数据总线，但 Master 只想传 8-bit 数据 (`AxSIZE=0`)。

*   Master 必须将数据放到数据总线的**正确 Byte Lane** 上。
*   Slave 必须从正确的 Byte Lane 获取数据。
*   这由地址的低位决定。例如地址 `0x1001`，数据必须在 Data Bus 的 [15:8] 上。

### 4.5 非对齐传输 (Unaligned Transfers)
AXI 支持首地址非对齐。Master 需做两件事：
1.  发送实际的非对齐地址 `AxADDR`。
2.  在第一个 Data Beat，正确设置 `WSTRB`，将不需要写的字节（起始地址之前的字节）Strobe 拉低。

---

## 5. 数据通道细节

### 5.1 WSTRB 选通信号生成
`WSTRB[n]` 对应 `WDATA[8n+7 : 8n]`。

生成伪代码逻辑：
```c
// Lower_Byte_Lane: 当前传输数据的最低字节所在的车道索引
// Upper_Byte_Lane: 当前传输数据的最高字节所在的车道索引
// Data_Bus_Bytes: 数据总线宽度(字节)

// 1. 确定本次传输的地址范围
Address_N = ... (基于Burst计算)
Lower_Byte_Lane = Address_N - (Address_N / Data_Bus_Bytes) * Data_Bus_Bytes; // 即 Address_N % Bus_Width

if (这是第一个Beat) {
    Upper_Byte_Lane = Aligned_Address + Number_Bytes - 1 - (Start_Data_Address / Data_Bus_Bytes) * Data_Bus_Bytes;
} else {
    Upper_Byte_Lane = Lower_Byte_Lane + Number_Bytes - 1;
}

// 2. 填充 Strobe
for (i = 0; i < Data_Bus_Bytes; i++) {
    if (i >= Lower_Byte_Lane && i <= Upper_Byte_Lane)
        WSTRB[i] = 1; // 有效
    else
        WSTRB[i] = 0; // 无效（填充或掩码）
}
```

### 5.2 字节不变性 (Byte Invariance)
AXI 采用**小端模式 (Little Endian)** 系统模型。
*   地址的低位对应数据的低位字节。
*   即使在大端系统中，数据也需要在总线接口处进行字节交换，以符合 AXI 定义。

---

## 6. 事务排序与乱序 (Ordering & Out-of-Order)

乱序是 AXI 提升效率的关键。

### 6.1 ID 信号的作用
每个读写通道都有 ID (AWID, WID, BID, ARID, RID)。
*   **ID 匹配**：Master 给一个 Transaction 打上 ID 标签。Slave 或 Interconnect 返回数据时必须带上相同的 ID。
*   **同 ID 顺序**：具有**相同 ID** 的 Transaction 必须严格按照发起的顺序执行和返回。
*   **异 ID 乱序**：具有**不同 ID** 的 Transaction 可以任意乱序执行。
    *   例如：Master 先发 ID=0 的读请求，再发 ID=1 的读请求。
    *   Slave 如果处理 ID=1 更快，可以先返回 ID=1 的数据。Master 利用 RID=1 识别这是后发的请求。

### 6.2 读写顺序模型
1.  **Read after Write (RAW)**: 如果 Master 写了一个地址，紧接着读这个地址。
    *   如果 ID 不同，AXI **不保证**读到的是新数据（可能写还没完成读就执行了）。Master 必须保证前一个写回应 (`BVALID`) 回来后，再发起读，或者强制使用相同 ID。
2.  **Write after Write (WAW)**: 对同一地址的连续写。
    *   必须使用相同 ID 才能保证写入顺序正确。使用不同 ID 可能导致后写的数据被先写的数据覆盖（如果它们在 Interconnect 中路径不同）。

---

## 7. 原子访问 (Atomic Accesses)

AXI 不支持总线锁定整个周期，而是用更高效的方式。

### 7.1 独占访问 (Exclusive Access)
用于实现信号量 (Semaphore)。
过程：
1.  **Master**: 发起一个读操作，`ARLOCK=1` (Exclusive)。
2.  **Slave**: 记录下这个地址被由于 ID=X 的 Master 监控了。返回 `EXOKAY`。
3.  **Master**: 修改数据。
4.  **Master**: 发起一个写操作，`AWLOCK=1`，写回该地址。
5.  **Slave**: 检查自刚才读之后，该地址有没有被**其他 Master** 写过？
    *   **没有被改过**：写入成功，返回 `EXOKAY`。原子操作完成。
    *   **被改过**：写入失败，不执行写入，返回 `OKAY`。原子操作失败，Master 需要重试。

### 7.2 锁定访问 (Locked Access)
`AxLOCK=1` (AXI4Legacy/AXI3)。
这会强制 Interconnect 锁定通道，直到该 Master 发出一个 `AxLOCK=0` 的传输。这会严重影响总线性能，AXI4 中已取消对 Locked 的强制支持。

---

## 8. AXI 版本变体对比

| 特性 | AXI3 | AXI4 | AXI4-Lite | AXI4-Stream |
|:---|:---|:---|:---|:---|
| **应用场景** | 高性能 SoC | 高性能 SoC (主流) | 寄存器配置，低速外设 | 高速流数据 (视频/网络) |
| **Burst 长度** | 1~16 | 1~256 (INCR)<br>1~16 (WRAP/FIXED) | 1 (不支持 Burst) | 无限 (Frame Based) |
| **WID 信号** | 有 (支持写数据乱序) | **无** (写数据必须顺序) | 无 | 无 (TID) |
| **QoS 信号** | 无 | 有 (AxQOS) | 无 | 无 |
| **Region 信号**| 无 | 有 (AxREGION) | 无 | 无 |
| **LOCK 信号** | 2-bit (支持 Atomic/Locked) | 1-bit (仅支持 Atomic) | 无 | 无 |
| **通道结构** | 5 通道 | 5 通道 | 5 通道 (简化) | 1 通道 (TDATA/TVALID...) |
| **数据位宽** | 8~1024 | 8~1024 | 32 / 64 | 任意 |
| **握手** | VALID/READY | VALID/READY | VALID/READY | VALID/READY |

> **总结建议**：学习时以 **AXI4** 为主，它是目前的工业标准。如果不做复杂的 Interconnect 设计，通常可以忽略 Locked 访问和 Write Data Interleaving (这是 WID 移除的原因)。重点掌握 **握手时序** 和 **Burst 地址计算**。

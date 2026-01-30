# SystemVerilog 核心语法笔记

## 1. 位切片与索引 (Bit-Slicing)

### 1.1 固定范围位切片 (Constant Part-Select)

这是最基础的位选取方式。它可以用于变量声明时定义位宽，也可以用于逻辑操作中截取一段固定的位。

* **用法示例**：
  * **声明**：`logic [4:0] a;` 定义了一个 5 bit 宽的向量 `a`，最高位索引为 4，最低位索引为 0。
  * **访问**：`b = a[4:0];` 将 `a` 的 0 到 4 位（全部位）赋值给 `b`。

### 1.2 缩减运算符 (Reduction Operators)

当你看到类似 `| a[4:0]` 的表达式时，这里的 `|` 是**一元缩减运算符**。

* **含义**：它对操作数的所有位进行逻辑“或”运算，最终结果是 **1 bit**。
* **计算逻辑**：`res = a[4] | a[3] | a[2] | a[1] | a[0];`
* **用途**：常用于判断“向量中是否有任何一位为 1”。
  * 如果 `a[4:0]` 中有任何一位是 `1`，结果就是 `1'b1`。
  * 如果全部是 `0`，结果就是 `1'b0`。
* **汇总表**：

| 运算符 | 功能 | 例子 |
| :--- | :--- | :--- |
| **&** | 缩减与 (Reduction AND) | `(& bit_vector)` |
| **\|** | 缩减或 (Reduction OR) | `(| bit_vector)` |
| **^** | 缩减异或 (Reduction XOR) | `(^ bit_vector)` |

---

### 1.3 索引位切片 (Indexed Part-Select)

在 SV 中，可以使用 `+:` 或 `-:` 来选取固定宽度的位段，这在循环或参数化模块中非常有用。它可以允许起始位是变量。

* **语法**：`[base_expr +: width]`
* **解释**：从起始位 `base_expr` 开始，**向上**选取 `width` 位。`width` 必须是常量。
* **示例**：

    ```systemverilog
    // 假设 start_type = 4
    dma_tr.dmain_blk_type[start_type +: 2] 
    // 等效于: dma_tr.dmain_blk_type[5 : 4] (起始位 4，向上选 2 位)
    ```

---

## 2. 自定义数据类型 (Typedefs)

### 2.1 枚举类型 (Enum)

枚举用于定义一组命名的常量，提高代码可读性。

* **基础用法**：

    ```systemverilog
    typedef enum {
        STATE_IDLE,    // 默认 32-bit int, 值为 0
        STATE_BUSY,    // 值为 1
        STATE_ERROR    // 值为 2
    } State_e;
    ```

* **指定基础类型与值**：

    ```systemverilog
    typedef enum bit [2:0] { 
        OP_NOP   = 3'b000,
        OP_READ  = 3'b001,
        OP_WRITE = 3'b010,
        OP_RESET = 3'b111 
    } Opcode_e;
    ```

    > **注意**：直接赋值需要符合枚举类型，或者使用 `State_e'(int_val)` 进行强制类型转换。

### 2.2 压缩结构体 (Packed Struct)

`packed` 结构体在存储上是连续的位，可以像向量一样进行位运算 or 赋值。

```systemverilog
typedef struct packed {
    logic        write_en;      // 位宽较高位
    logic [3:0]  write_id;      
    logic [31:0] write_addr;    
    logic [63:0] write_data;    
    logic [7:0]  write_strb;    
    logic [7:0]  write_len;     // 位宽最低位
} axi_write_transaction_t;

// 声明数组
axi_write_transaction_t axi_write_transaction[6];

// 调试打印 (使用 %p 格式)
$display("Transaction info: %p", axi_write_transaction[0]);
```

---

## 3. 函数与任务 (Task/Function)

### 3.1 参数传递方式

调用时可以通过位置或名称来匹配参数。

* **定义示例**：

    ```systemverilog
    task my_transfer_task(
        input int address,
        input int data_len,
        input bit [7:0] flags = '0, // 默认参数
        output int status
    );
        // ... 内容 ...
    endtask
    ```

* **按名称传递 (推荐)**：

    ```systemverilog
    // 更加稳健，顺序可变
    my_transfer_task(.data_len(len_v), .address(addr_v), .flags(flags_v), .status(status_v));
    ```

* **简写形式 (.name)**：

    ```systemverilog
    // 当变量名与参数名一致时，可以简写
    my_transfer_task(.address, .data_len, .flags, .status);
    ```

---

## 4. 字符串操作 (Strings)

### 4.1 substr 用法

用于提取子字符串。

```systemverilog
string str = "Hello, SystemVerilog!";
string sub_str;

// 语法: str.substr(start_index, end_index)
// 注意: 包含 end_index 处的字符
sub_str = str.substr(7, 16); // 结果: "SystemVerilog"

// 提取直至末尾
sub_str = str.substr(7);      // 结果: "SystemVerilog!"
```

---

## 5. 编译预处理与宏 (Macros)

### 5.1 宏定义的灵活调整

利用 `` `ifdef `` 结合编译选项实现动态配置。

* **代码内部**：

    ```systemverilog
    `define AXI_DLY_VAL 1000 // 默认值

    `ifdef AXI_DLY_VAL
        axi_dly(`AXI_DLY_VAL); // 注意使用 ` 引用宏值
    `endif
    ```

* **仿真命令行修改** (无需重新修改源码)：
  * **VCS/Verdi**: `+define+AXI_DLY_VAL=2000`
  * **Xcelium**: `-define AXI_DLY_VAL=2000`

---
> *Created by Antigravity*

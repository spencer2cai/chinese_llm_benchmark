请用 Verilog 设计一个 I2C slave 转 APB master 的桥接模块，并提供一个完整的 testbench（含 APB slave 模型和 I2C master 模型），要求用 iverilog 仿真生成 VCD 波形，波形中必须能清楚看到 I2C 读操作时的时钟拉伸现象（SCL 被 slave 拉低等待 APB 读完成）。

### 设计要求
- 工作时钟 clk = 100MHz，异步复位 rst_n。
- I2C slave 接口：scl（inout），sda（inout），设备地址 7'h50。
- APB master 接口（标准 APB3）：psel, penable, pwrite, paddr[31:0], pwdata[31:0], prdata[31:0], pready, pslverr。
- 桥接功能：将 I2C 写操作转换为 APB 写，将 I2C 读操作转换为 APB 读。
- I2C 传输格式（固定格式）：
  - 寄存器地址：1 字节（8 位），对应 APB 地址：0x00, 0x04, 0x08, 0x0C（低两位忽略，实际用地址[7:0] << 2 映射到 32 位地址空间）。
  - 数据：固定 4 字节（32 位），MSB first（即第一个数据字节是 bit31-24，第二个是 bit23-16，第三个是 bit15-8，第四个是 bit7-0）。
  - 写操作完整流程：起始 -> 设备地址(7'h50) + 写位(0) -> ACK -> 1字节寄存器地址 -> ACK -> 4字节数据（每字节后 ACK）-> 停止。
  - 读操作完整流程：起始 -> 设备地址+写位 -> ACK -> 1字节寄存器地址 -> ACK -> 重复起始 -> 设备地址+读位(1) -> ACK -> 读取4字节数据（每字节 master 发送 ACK，最后一字节后发送 NACK）-> 停止。
- **必须实现时钟拉伸**：
  - 当收到 I2C 读命令（重复起始后的设备地址+读）后，需要先发起 APB 读并等待 pready。在等待期间，将 scl 拉低（通过三态输出使能），直到 APB 读完成再释放 scl。
  - 写操作无需拉伸（可在收到停止后再发起 APB 写，此时总线已空闲）。
  - scl 必须定义为 inout 类型，内部有 scl_out_en 和 scl_out 控制。

### Testbench 要求
- 例化桥接模块，例化一个 APB slave 模型，包含 4 个 32 位寄存器（地址 0x00,0x04,0x08,0x0C）。
  - **关键**：该 APB slave 的 pready 信号必须延迟 2us（即 200 个 clk 周期）。例如，在收到 APB 读或写传输后，将 pready 拉低 200 个周期后再拉高。写操作同样延迟（但不影响 I2C 拉伸，因为写发生在停止之后）。
- 编写一个行为级 I2C master 模型，支持时钟拉伸检测（即当 scl 被 slave 拉低时，master 必须等待 scl 释放后再继续产生时钟）。速率 1MHz（SCL 周期 1us）。
- I2C master 执行以下操作序列：
  1. 写地址 0x00 数据 0x00000001
  2. 写地址 0x04 数据 0x00000002
  3. 写地址 0x08 数据 0x00000003
  4. 写地址 0x0C 数据 0x00000004
  5. 读地址 0x00（期望 1），读 0x04（期望 2），读 0x08（期望 3），读 0x0C（期望 4）
- 仿真结束时打印 "PASS" 或 "FAIL"。
- 生成 VCD 文件，记录 clk, scl, sda, psel, paddr, pready 等信号。

### 输出要求
输出单个 Verilog 文件，包含：
- i2c_to_apb_bridge 模块（顶层）
- apb_slave 模型（带 1us 延迟的 pready）
- i2c_master 行为模型（含拉伸检测）
- testbench 顶层
- testbench测试通过后的vcd波形文件，让我检查你的结果

Common module list:
- [base_fifo_sync](#entity-base_fifo_sync)
- [skid_buffer](#entity-skid_buffer)
- [ram_1r1w_sync](#entity-ram_1r1w_sync)
- [ram_2r1w_sync](#entity-ram_2r1w_sync)
- [multiple_bank_ram_2r1w_sync](#entity-multiple_bank_ram_2r1w_sync)


# Entity: base_fifo_sync

- **File**: base_fifo_sync.v

## Diagram

![](./img/top_fifo.png)

## Generics

| Generic name | Type | Value | Description   |
| ------------ | ---- | ----- | ------------- |
| Width_g           | parameter      | 8     | Data Width    |
| Depth_g           | parameter     | 16    | Depth of FIFO |
| RamStyle_g | parameter string | "auto" | Through this generic, the exact resource to use for FPGA implementation can be controlled. This generic is applied to the attributes ram_style and ramstyle which vendors offer to control RAM implementation. Commonly used values are given below.<br>__AMD (Xilinx)__: <t style="color:red">"auto" (default)</t>, "block", "distributed", "ultra" - see ug901 for details<br>__Intel (Quartus)__: "M4K", "M9K", "M20K", "M144K", "MLAB" - see quartus-help for details<br>__Efinix__: "block_ram", "registers" - see efinity-synthesis for details<br>__Synplify(Lattice/Microchip)__: "block_ram", "registers", "distributed" - see microchip-attributes-guide for details<br>__Gowin__: "block_ram", "distributed_ram", "registers", "rw_check", "no_rw_check" - see GowinSynthesis User Guide for details.
|RamBehavior_g|	parameter string|	"RBW"	| Controls the RAM behavior. Must match the behavior of RAM resources of the target technology for efficient implementation. If you are unsure what behavior your target device offers, try both settings and check which one is correctly mapped to RAM resources using the synthesis report.<br>__"RBW"__: Read-Before-Write - more common, hence the <t style="color:red">default</t><br>__"WBR"__: Write-Before-Read

## Ports

| Port name    | Direction | Type  | Description                                                                             |
| ------------ | :-------: | :---- | --------------------------------------------------------------------------------------  |
| clk         | input     |       | Tín hiệu đồng hồ. Module hoạt động theo cạnh lên của `clk`. |
| rst         | input     |       | Reset đồng bộ tích cực cao.                                |
| full | input  | | Báo hiệu FIFO đầy.
| din  | output | [Width_g-1:0] | Dữ liệu ghi vào FIFO.
| wr_en | output | | Ghi dữ liệu vào FIFO.
| empty | input  | | Báo hiệu FIFO trống.
| din   | input  | [Width_g-1:0] | Dữ liệu đọc từ FIFO.
| rd_en | output | | Đọc dữ liệu từ FIFO.

## Functional description

This component implements a FIFO that:
- __Synchronous__: Same clock for write and read port.
- __Symmetric__: Same read and write port width.
- __Fall-through__: Valid read data output is present when possible (FIFO not empty).
- __FPGA oriented__: allows flexible, efficiently implementing FIFOs for different technologies.
    - The memory is described in a way that it utilizes RAM resources (Block-RAM, distributed RAM, etc.) available in FPGAs with commonly used tools.
    - The RAM behavior can be selected.

__NOTE__: No detailed architecture documentation is required for a FIFO. The behavior of the FIFO is sufficiently defined without details about the internals being required.

# Entity: skid_buffer

## Diagram

![](./img/top_skid_buffer.png)

## Generic

| Generic name | Type | Value | Description   |
| ------------ | ---- | ----- | ------------- |
| DW           | parameter      | 8     | Data Width    |

## Ports

| Port name | Direction | Type     | Description                                                 |
| --------- | --------- | -------- | ----------------------------------------------------------- |
| clk       | input     |          | Tín hiệu đồng hồ. Module hoạt động theo cạnh lên của `clk`. |
| rst       | input     |          | Reset đồng bộ khi `rst = 1`.                                |
| enable    | input     |          | Module chỉ có thể raise `wready` nếu `enable = 1`.
| wdata     | input| [DW-1:0]| Dữ liệu đầu vào.
| wvalid|input||Dữ liệu đầu vào hợp lệ.
|wready|output||Sẵn sàng nhận dữ liệu đầu vào.
|rdata|output|[DW-1:0]|Dữ liệu đầu ra.
|rvalid|output||Dữ liệu đầu ra hợp lệ.
|rready|input||Sẵn sàng lấy dữ liệu đầu ra.

## Functional description

Skid buffer (đôi khi gọi là skid FIFO) là bộ đệm nhỏ, chèn giữa 2 khối giao tiếp bắt tay (`valid` - `ready`), có tác dụng "trượt" dữ liệu khi deassert `ready` mà không làm mất dữ liệu trong thiết kế pipeline, giúp duy trì throughput 1 cycle/transfer.

Ví dụ trong 1 pipeline dữ liệu, để break critical path timing ta chèn vào 1 register stage giữa Producer và Consumer. Lúc này nếu Consumer deassert `ready`, Producer assert `valid` nhưng do register stage mà Producer thấy `ready` được cập nhật chậm hơn 1 clock cycle (tạo bubble), mẫu dữ liệu trước đó sẽ bị mất. Thay vào đó, ta sử dụng _skid buffer_ sẽ tránh được tình trạng trên.

__Ứng dụng__: Valid-ready style pipeline stage insertion, break critical path mà không tạo bubble, giữ thoughput 1 cycle/transfer.

|<p style="text-align:center">FSM</p>|<p style="text-align:center">Description</p>|
|-|-|
|![](./img/skid_buffer_FSM.png)|__EMPTY__: Thanh ghi đệm `data_buff` trống, thanh ghi đầu ra `rdata` trống. Raise `wready` báo hiệu sẵn sàng nhận `wdata` mới. Nếu `whandshaked` (`wvalid && wready`), cập nhật `rdata` theo `wdata` đồng thời chuyển trạng thái __BUSY__.<br><br>__BUSY__: Thanh ghi đệm `data_buff` trống, thanh ghi đầu ra `rdata` đầy. Vẫn raise `wready` nhận data mới, đồng thời raise `rvalid` báo hiệu có `rdata`.<br>__- Case #1__: Nếu `whandshaked` và `rhandshaked` (`rvalid && rready`), thanh ghi `rdata` đầu ra được cập nhật theo `wdata` mới.<br>__- Case #2__: Nếu chỉ `rhandshaked`, thanh ghi `rdata` được tiêu thụ và không có `wdata` để cập nhật nên sẽ "trống", chuyển trạng thái __EMPTY__.<br>__- Case #3__: Nếu chỉ `whandshaked`, thanh ghi `rdata` chưa được tiêu thụ nên không thể cập nhật `wdata` mới mà cần sử dụng thanh ghi đệm `data_buff` để tạm giữ lại, đồng thời chuyển trạng thái __FULL__.<br><br>__FULL__: Thanh ghi đệm `data_buff` đầy, thanh ghi đầu ra `rdata` đầy. Raise `rvalid` báo hiệu có `rdata`. Nếu `rhandshaked`, thanh ghi `rdata` được tiêu thụ, đồng thời cập nhật giá trị đệm từ `data_buff` sẵn sàng cho lần `rhandshake` tiếp theo và chuyển trạng thái __BUSY__.

# Wave

|<p style="text-align:center">Data pipeline với Producer và Consumer kết nối trực tiếp</p>
|-|
|![](./wave/skid_buffer_pipe_norm.png)|

|<p style="text-align:center">Data pipeline với Producer registed outputs, Consumer thấy data trễ 1 clock cycle</p>
|-|
|![](./wave/skid_buffer_pipe_reg.png)|

|<p style="text-align:center">Data pipeline với skid buffer chèn giữa Producer và Consumer</p>
|-|
|![](./wave/skid_buffer_pipe_skid.png)|

# Entity: ram_1r1w_sync

_RAM 1 Read ports 1 Write port, read-write SYNChronous_

- **File**: ram_1r1w_sync.sv

## Diagram

![](./img/top_ram_1r1w_sync.png)

## Generics

| Generic name | Type | Value | Description   |
| ------------ | ---- | ----- | ------------- |
| DW           |      | 2     | Data Width    |
| AW           |      | 12    | Address Width |

## Ports

| Port name | Direction | Type     | Description                                                 |
| --------- | --------- | -------- | ----------------------------------------------------------- |
| clk       | input     |          | Tín hiệu đồng hồ. Module hoạt động theo cạnh lên của `clk`. |
| rst       | input     |          | Reset đồng bộ khi `rst = 1`.                                |
| i_waddr   | input     | [AW-1:0] | Write Address                                               |
| i_wdata   | input     | [DW-1:0] | Write Data                                                  |
| i_wvalid  | input     |          | Write Valid                                                |
| i_raddr   | input     | [AW-1:0] | Read Address                                    |
| o_rdata   | output    | [DW-1:0] | Read Data                                       |

## Functional description

Khi `i_wvalid` = 1, dữ liệu `i_wdata` được ghi vào địa chỉ `i_waddr` tại chu kỳ `clk` kế tiếp. Khi `i_raddr` thay đổi hoặc dữ liệu tại `i_raddr` thay đổi, giá trị `o_rdata` đọc được từ `i_raddr` được cập nhật tại chu kỳ `clk` kế tiếp. Có 1 port để ghi dữ liệu, 2 port để đọc dữ liệu, các port có thể hoạt động độc lập và không ảnh hưởng tới nhau, ưu tiên dữ liệu tại các port là đọc trước, ghi sau.

## Latency

Độ trễ Write = 1. Độ trễ Read = 1.

Độ trễ Write của module được tính từ lúc `i_wvalid` được kích hoạt tới lúc `i_wdata` được ghi vào địa chỉ `i_waddr`.

Độ trễ Read của module được tính từ lúc giá trị địa chỉ `i_raddr` hoặc dữ liệu tại `i_raddr` thay đổi tới lúc `o_rdata` được đọc từ địa chỉ `i_raddr`.

# Entity: ram_2r1w_sync

_RAM 2 Read ports 1 Write port, read-write SYNChronous_

- **File**: ram_2r1w_sync.sv

## Diagram

![](./img/top_ram_2r1w_sync.png)

## Generics

| Generic name | Type | Value | Description   |
| ------------ | ---- | ----- | ------------- |
| DW           |      | 2     | Data Width    |
| AW           |      | 12    | Address Width |

## Ports

| Port name | Direction | Type     | Description                                                 |
| --------- | --------- | -------- | ----------------------------------------------------------- |
| clk       | input     |          | Tín hiệu đồng hồ. Module hoạt động theo cạnh lên của `clk`. |
| rst       | input     |          | Reset đồng bộ khi `rst = 1`.                                |
| i_waddr   | input     | [AW-1:0] | Write Address                                               |
| i_wdata   | input     | [DW-1:0] | Write Data                                                  |
| i_wvalid  | input     |          | Write Valid                                                |
| i_raddr_A | input     | [AW-1:0] | Read Address from port A                                    |
| i_raddr_B | input     | [AW-1:0] | Read Address from port B                                    |
| o_rdata_A | output    | [DW-1:0] | Read Data from port A                                       |
| o_rdata_B | output    | [DW-1:0] | Read Data from port B                                       |

## Functional description

Khi `i_wvalid` = 1, dữ liệu `i_wdata` được ghi vào địa chỉ `i_waddr` tại chu kỳ `clk` kế tiếp. Khi `i_raddr_X` thay đổi hoặc dữ liệu tại `i_raddr_X` thay đổi, giá trị `o_rdata_X` đọc được từ `i_raddr_X` được cập nhật tại chu kỳ `clk` kế tiếp. Có 1 port để ghi dữ liệu, 1 port để đọc dữ liệu, các port có thể hoạt động độc lập và không ảnh hưởng tới nhau, ưu tiên dữ liệu tại các port là đọc trước, ghi sau.

## Latency

Độ trễ Write = 1. Độ trễ Read = 1.

Độ trễ Write của module được tính từ lúc `i_wvalid` được kích hoạt tới lúc `i_wdata` được ghi vào địa chỉ `i_waddr`.

Độ trễ Read của module được tính từ lúc giá trị địa chỉ `i_raddr` hoặc dữ liệu tại `i_raddr` thay đổi tới lúc `o_rdata` được đọc từ địa chỉ `i_raddr`.

# Entity: multiple_bank_ram_2r1w_sync

_Multiple bank 2 Read ports 1 Write port, read-write SYNChronous_.

- **File**: multiple_bank_ram_2r1w_sync.sv

## Diagram

![Diagram](./img/top_multiple_bank_ram_2r1w_sync.png "Diagram")

## Generics

| Generic name | Type | Value | Description     |
| ------------ | ---- | ----- | --------------- |
| DW           |      | 2     | Data Width      |
| AW           |      | 12    | Address Width   |
| NB           |      | 3     | Number of Banks |

## Ports

| Port name | Direction | Type        | Description |
| --------- | --------- | ----------- | ----------- |
| clk       | input     |             | Tín hiệu đồng hồ. Module hoạt động theo cạnh lên của `clk`.
| rst       | input     |             | Reset đồng bộ khi `rst = 1`.
| i_wvalid  | input     | [NB-1:0]    | Write Valid
| i_waddr   | input     | [AW-1:0]    | Write Address
| i_wdata   | input     | [DW-1:0]    | Write Data
| i_raddr_A | input     | [AW-1:0] [NB] | Read Address from port A
| i_raddr_B | input     | [AW-1:0] [NB] | Read Address from port B
| o_rdata_A | output    | [DW-1:0] [NB] | Read Data from port A
| o_rdata_B | output    | [DW-1:0] [NB] | Read Data from port B

## Functional description

Khối này là ghép của `NB` khối [ram_2r1w_sync](#entity-ram_2r1w_sync), gọi là các bank.

## Latency

Độ trễ của module phụ thuộc vào module [ram_2r1w_sync](#entity-ram_2r1w_sync)
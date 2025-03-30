# I2C-Verilog
I2C slave Controller Verilog Design

### 1.架構關係圖

![image](https://github.com/user-attachments/assets/da3ef305-be8e-4f66-8ee3-9828777cfca7)

### 2. i2cs_fsm.v：I2C Slave 狀態機（核心讀寫控制）

i2cs_fsm.v 是 I2C Slave 接收/傳送的核心程式，負責解析 I2C 訊號、送出 ACK、知道何時要輸入/輸出資料等

I2C Slave 的狀態機 (FSM)：

1.監控 SDA、SCL 的線路變化（利用 Debounce ）來判斷何時進入起始條件 (START)、停止條件 (STOP)，以及在讀寫流程中的各個時序

2.比對 Slave Address，判定是否正確地址後再回應 ACK。

3.接收 I2C Master 發出的讀寫要求，如 Write（包含目標暫存器地址、數據等）與 Read（Slave 回傳數據）

4.與內部的暫存器介面或資料線路（透過 data_from_reg / data_write 與 i2cs_raxi_reg.v 連動）進行交換，當讀取命令時，狀態機將所需的資料從內部暫存器讀出；當寫入命令時，狀態機把資料寫入內部暫存器

5.在正確或錯誤的時序下觸發對應的中斷訊號，如 Start Interrupt、Stop Interrupt 或 Error Interrupt 等（以 irq_start、irq_stop、irq_error 呈現）


### 3.i2cs_raxi_reg.v : RAXI 接口 + 暫存器，儲存 I2C 傳入/傳出的資料與設定

可以視為 I2C Slave 模組的「可記憶、可配置」暫存器區，也可以讓外部透過 RAXI 介面來控制 I2C Slave 的行為或讀取其狀態與資料

RAXI 暫存器介面: 

1.對外提供一組類似 AXI / Memory Mapped 讀寫介面 ( raxi_valid、raxi_rw、raxi_address、raxi_data_i / raxi_data_o )，讓「上位層的 CPU / Bus」可以直接讀寫這些「模組內部的暫存器」

2.內部維護一組暫存器，包括：

 - Slave Address 設定 (reg_slave_addr)：I2C Slave 所使用的 7 位地址

 - Debounce 設定 (reg_debounce)：FSM 用來判斷 SDA、SCL 穩定狀態的參數

 - Data 暫存器組：用來儲存或提供 I2C 傳輸的實際資料（例如 REG_Data[0..15]），支援 32-bit 寫入/讀取

 - Interrupt Enable / Status 暫存器 (int_en, int_status)：控制是否使能中斷，以及中斷發生時狀態如何

3.與 FSM 之間有一些訊號連動：

 - i2c_reg_write_en / i2c_reg_index / i2c_reg_data：FSM 偵測到 Master 要寫哪個暫存器、寫的資料是什麼，會以這些訊號告訴此 RAXI 模組，以完成對暫存器組的更新

 - fsm_status_in, i2c_rw_in, i2c_word_in 等，用來在 RAXI 端可以觀察到目前 I2C 的狀態、讀寫指令等（做 debug 或記錄）

 - 中斷訊號整合 (irq_start_in, irq_stop_in, irq_error_in)：RAXI 模組會把這些資訊寫進 int_status 之類的暫存器，並根據 int_en 決定是否要對外輸出最終的 i2c_int


### 4.i2c_slave_top.v : 頂層模組，把 FSM 與 RAXI 結合，定義對外的介面

Top-Level 模組，主要是將 i2cs_fsm.v 與 i2cs_raxi_reg.v 整合在一起:

1.i2cs_raxi_reg，將 RAXI 相關輸入和寄存器連線到各個內部變數

2.i2cs_fsm，處理 I2C 訊號，並與寄存器模組之間交換資料

3.統一將 FSM 輸出的 irq_start, irq_stop, irq_error 與 int_en 結合，得出最終 i2c_int

4.將 sda_out 固定為 0，並用 sda_oe 來控制是否驅動 SDA（符合 I2C Slave 拉低 SDA 的動作）

因為通常 I2C 的 SDA 會是 雙向（bidirectional），所以在 FPGA 裡會用到 sda_in 與 sda_oe 這樣的設計方式。sda_oe 為 1 時代表 Slave 要拉低 SDA（輸出），否則就呈現 Hi-Z，交由外部上拉。

## 模擬 （iverilog / gtkwave）

1. iverilog / gtkwave :
```
 iverilog -o simulation i2cs_fsm.v i2c_slave_top.v i2cs_raxi_reg.v tb_i2c_slave.v

 vvp simulation
```

2.用 gtkwave 觀察波形

![image](https://github.com/user-attachments/assets/041de813-5130-4848-b483-53a335ea531b)

-











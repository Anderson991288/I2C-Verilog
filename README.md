# I2C-Verilog
I2C slave Controller Verilog Design

### 1.架構關係圖

![image](https://github.com/user-attachments/assets/da3ef305-be8e-4f66-8ee3-9828777cfca7)

### 2. i2cs_fsm.v：I2C Slave 狀態機（核心讀寫控制）

i2cs_fsm.v 是 I2C Slave 接收/傳送的核心程式，負責解析 I2C 訊號、送出 ACK、知道何時要輸入/輸出資料等

實作 I2C Slave 的狀態機 (FSM)：

1.監控 SDA、SCL 的線路變化（利用 Debounce ）來判斷何時進入起始條件 (START)、停止條件 (STOP)，以及在讀寫流程中的各個時序

2.比對 Slave Address，判定是否正確地址後再回應 ACK。

3.接收 I2C Master 發出的讀寫要求，如 Write（包含目標暫存器地址、數據等）與 Read（Slave 回傳數據）

4.與內部的暫存器介面或資料線路（透過 data_from_reg / data_write 與 i2cs_raxi_reg.v 連動）進行交換，當讀取命令時，狀態機將所需的資料從內部暫存器讀出；當寫入命令時，狀態機把資料寫入內部暫存器

5.在正確或錯誤的時序下觸發對應的中斷訊號，如 Start Interrupt、Stop Interrupt 或 Error Interrupt 等（以 irq_start、irq_stop、irq_error 呈現）







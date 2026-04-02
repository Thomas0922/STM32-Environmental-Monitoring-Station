# STM32 SPI 通訊 & BME280 感測器學習筆記
**DevEBox STM32F407VET6 | Week 3**

---

## 一、背景知識：SPI 通訊協定

### 1.1 什麼是 SPI？

SPI（Serial Peripheral Interface）是一種同步串列通訊協定，比 I2C 更快，需要 4 條線：

```
STM32 (Master)          BME280 (Slave)
    SCK  ──────────────→  SCK   時脈線（Master 控制）
    MOSI ──────────────→  SDI   資料線（Master → Slave）
    MISO ←──────────────  SDO   資料線（Slave → Master）
    CS   ──────────────→  CSB   片選線（LOW = 選中）
```

| 縮寫 | 全名 | 說明 |
|------|------|------|
| SCK | Serial Clock | 時脈，由 Master 產生 |
| MOSI | Master Out Slave In | Master 送資料給 Slave |
| MISO | Master In Slave Out | Slave 送資料給 Master |
| CS/NSS | Chip Select | LOW = 選中該裝置 |

### 1.2 SPI vs I2C 比較

| | SPI | I2C |
|--|-----|-----|
| 線數 | 4 條 | 2 條 |
| 速度 | 快（可達數十 MHz）| 較慢（100KHz~400KHz）|
| 裝置數 | 每個裝置需要獨立 CS | 共用 2 條線，用地址區分 |
| 複雜度 | 硬體簡單 | 軟體協定複雜 |
| 適用 | 高速感測器、Flash、顯示器 | 多裝置、低速感測器 |

### 1.3 SPI 時序模式（CPOL/CPHA）

SPI 有 4 種時序模式，BME280 支援 Mode 0 和 Mode 3：

| Mode | CPOL | CPHA | 說明 |
|------|------|------|------|
| 0 | 0 | 0 | 時脈閒置 LOW，第一個邊沿採樣 ← **本專案使用** |
| 1 | 0 | 1 | 時脈閒置 LOW，第二個邊沿採樣 |
| 2 | 1 | 0 | 時脈閒置 HIGH，第一個邊沿採樣 |
| 3 | 1 | 1 | 時脈閒置 HIGH，第二個邊沿採樣 |

### 1.4 SPI 讀寫流程

BME280 的 SPI 讀寫規則：
- **讀取**：第一個 byte 的最高位（bit7）設為 **1**，剩下 7 位是暫存器地址
- **寫入**：第一個 byte 的最高位（bit7）設為 **0**，剩下 7 位是暫存器地址

```
讀取：CS LOW → 送 [reg | 0x80, 0x00] → 收 [dummy, data] → CS HIGH
寫入：CS LOW → 送 [reg & 0x7F, value] → CS HIGH
```

---

## 二、背景知識：BME280 感測器

### 2.1 BME280 簡介

BME280 是 Bosch 出品的環境感測器，可同時測量：
- 溫度（-40°C ~ 85°C，精度 ±1°C）
- 濕度（0% ~ 100%，精度 ±3%）
- 氣壓（300~1100 hPa）

### 2.2 重要暫存器

| 暫存器 | 地址 | 說明 |
|--------|------|------|
| id | 0xD0 | Chip ID，BME280 固定回傳 0x60 |
| ctrl_hum | 0xF2 | 濕度取樣設定，**必須在 ctrl_meas 之前寫** |
| ctrl_meas | 0xF4 | 溫度/壓力取樣設定 + 工作模式 |
| 溫度原始值 | 0xF7~0xF9 | 20-bit 原始溫度資料 |
| 濕度原始值 | 0xFD~0xFE | 16-bit 原始濕度資料 |
| 溫度校準 | 0x88~0x8D | 6 bytes 溫度校準係數 |
| 濕度校準 | 0xA1、0xE1~0xE7 | 8 bytes 濕度校準係數 |

### 2.3 為什麼需要校準參數？

BME280 的感測元件每一顆都有細微差異，出廠時 Bosch 會對每顆晶片進行校準，把校準係數燒錄在晶片的 ROM 裡。讀取原始數值後，必須套用這些係數才能得到正確的溫度/濕度值。

### 2.4 工作模式

| 模式 | 說明 |
|------|------|
| Sleep mode | 不量測，省電 |
| Forced mode | 量測一次後回到 Sleep |
| Normal mode | 持續量測（本專案使用）|

---

## 三、硬體接線

### 3.1 腳位對應

| BME280 | 說明 | 接到 STM32 |
|--------|------|-----------|
| VCC | 電源 | 3.3V |
| GND | 接地 | GND |
| SCK | SPI 時脈 | PB13（SPI2_SCK）|
| SDI | SPI 資料輸入（MOSI）| PC3（SPI2_MOSI）|
| SDO | SPI 資料輸出（MISO）| PC2（SPI2_MISO）|
| CSB | 片選（CS）| PB14（GPIO Output）|

**注意：**
- BME280 的 `SDI` = SPI 的 `MOSI`（名稱不同，功能一樣）
- CSB 接 PB14（軟體控制），不是 GND 也不是 3.3V
- 這個模組有板載上拉電阻，不需要外接

### 3.2 為什麼用 SPI2 而不是 SPI1？

SPI1 的 MISO/MOSI 腳位（PA6/PA7）和板子的 LED D2/D3 衝突，因此改用 SPI2（PB13/PC2/PC3）。

---

## 四、CubeMX 配置說明

### 4.1 SPI2 配置

**路徑：** Connectivity → SPI2

| 設定 | 值 | 原因 |
|------|-----|------|
| Mode | Full-Duplex Master | 同時收發資料 |
| NSS | Disabled（Software）| 用軟體手動控制 CS |
| CLKPolarity | Low（CPOL=0）| BME280 Mode 0 要求 |
| CLKPhase | 1 Edge（CPHA=0）| BME280 Mode 0 要求 |
| BaudRate Prescaler | 64 | 速度慢一點，除錯更穩定 |
| FirstBit | MSB | 最高位先送，SPI 標準做法 |

**為什麼 NSS 選 Software？**
NSS（CS）由軟體控制，可以精確掌握何時拉低、何時拉高，比硬體自動控制更靈活，也適合同時掛多個 SPI 裝置的場景。

### 4.2 PB14 配置

PB14 設成 `GPIO_Output`，作為 BME280 的 CS 片選腳。預設值設為 HIGH（不選中）。

---

## 五、程式碼詳細說明

### 5.1 SPI 驅動函數

```c
// CS 控制 Macro
// GPIO_PIN_RESET = LOW = 選中 BME280，開始通訊
// GPIO_PIN_SET   = HIGH = 取消選中，結束通訊
#define BME280_CS_LOW()  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_14, GPIO_PIN_RESET)
#define BME280_CS_HIGH() HAL_GPIO_WritePin(GPIOB, GPIO_PIN_14, GPIO_PIN_SET)
```

```c
uint8_t BME280_ReadReg(uint8_t reg) {
    // SPI 讀取時，第一個 byte 的 bit7 必須是 1
    // reg | 0x80：把暫存器地址的最高位強制設為 1
    // 第二個 byte 送 0x00（dummy byte），讓時脈繼續跑以接收資料
    uint8_t tx[2] = {reg | 0x80, 0x00};
    uint8_t rx[2] = {0x00, 0x00};

    BME280_CS_LOW();  // 拉低 CS，告訴 BME280「要跟你說話了」

    // HAL_SPI_TransmitReceive：同時送和收
    // &hspi2：使用 SPI2
    // tx：送出的資料
    // rx：接收的資料
    // 2：送/收 2 個 bytes
    // 100：超時 100ms
    // 第一個收到的 byte（rx[0]）是 dummy，有效資料在 rx[1]
    HAL_SPI_TransmitReceive(&hspi2, tx, rx, 2, 100);

    BME280_CS_HIGH(); // 拉高 CS，結束通訊
    return rx[1];     // 回傳實際讀到的暫存器值
}
```

```c
void BME280_WriteReg(uint8_t reg, uint8_t value) {
    // SPI 寫入時，第一個 byte 的 bit7 必須是 0
    // reg & 0x7F：把暫存器地址的最高位強制設為 0
    uint8_t tx[2] = {reg & 0x7F, value};

    BME280_CS_LOW();

    // HAL_SPI_Transmit：只送，不收
    // 寫入不需要接收回傳值
    HAL_SPI_Transmit(&hspi2, tx, 2, 100);

    BME280_CS_HIGH();
}
```

### 5.2 感測器初始化

```c
void Task_BME280(void *argument) {

    // Step 1：讀取 Chip ID 驗證連線
    // 0xD0 是 BME280 的 ID 暫存器，正常應該回傳 0x60
    // 如果不是 0x60，代表 SPI 通訊有問題（接線錯誤、腳位錯誤等）
    uint8_t id = BME280_ReadReg(0xD0);
    if (id != 0x60) {
        while (1) osDelay(1000); // 卡住，不繼續
    }

    // Step 2：設定感測器模式
    // 0xF2 = ctrl_hum 暫存器
    // 0x01 = 濕度 oversampling x1（量一次取平均）
    // 重要：必須先寫 0xF2，再寫 0xF4，否則濕度設定不生效
    BME280_WriteReg(0xF2, 0x01);

    // 0xF4 = ctrl_meas 暫存器
    // 0x27 = 0b00100111
    //   bit[7:5] = 001 → 溫度 oversampling x1
    //   bit[4:2] = 001 → 壓力 oversampling x1
    //   bit[1:0] = 11  → Normal mode（持續量測）
    BME280_WriteReg(0xF4, 0x27);
    osDelay(100); // 等感測器完成第一次量測
```

### 5.3 讀取校準參數

```c
    // Step 3：讀取溫度校準參數（暫存器 0x88~0x8D，共 6 bytes）
    uint8_t calT[6];
    for (int i = 0; i < 6; i++) calT[i] = BME280_ReadReg(0x88 + i);

    // Little-Endian 格式：低位元組在前，高位元組在後
    // calT[0] = T1 低 8 位，calT[1] = T1 高 8 位
    dig_T1 = (uint16_t)((calT[1] << 8) | calT[0]); // 無號數（uint16）
    dig_T2 = (int16_t) ((calT[3] << 8) | calT[2]); // 有號數（int16）
    dig_T3 = (int16_t) ((calT[5] << 8) | calT[4]); // 有號數（int16）

    // Step 4：讀取濕度校準參數
    // dig_H1 在 0xA1（單獨一個 byte）
    dig_H1 = BME280_ReadReg(0xA1);

    // dig_H2~H6 在 0xE1~0xE7（7 bytes，格式比較複雜）
    uint8_t calH[7];
    for (int i = 0; i < 7; i++) calH[i] = BME280_ReadReg(0xE1 + i);

    dig_H2 = (int16_t)((calH[1] << 8) | calH[0]);
    dig_H3 = calH[2];
    // H4 和 H5 共用 calH[4] 的位元
    dig_H4 = (int16_t)((calH[3] << 4) | (calH[4] & 0x0F)); // 低 4 位
    dig_H5 = (int16_t)((calH[5] << 4) | (calH[4] >> 4));   // 高 4 位
    dig_H6 = (int8_t)calH[6];
```

### 5.4 讀取原始數值並補償

```c
    for (;;) {
        // Step 5：讀取原始感測數據
        // 0xF7~0xFE 包含壓力（3 bytes）、溫度（3 bytes）、濕度（2 bytes）
        uint8_t raw[8];
        for (int i = 0; i < 8; i++) raw[i] = BME280_ReadReg(0xF7 + i);

        // 解析溫度原始值（20-bit）
        // raw[3] = press MSB, raw[4] = press LSB, raw[5] = press XLSB
        // raw[3]~raw[5] 其實是壓力，raw[3] << 12 開始才是溫度
        // 正確應該是 raw[3]=press_msb, raw[4]=press_lsb, raw[5]=press_xlsb
        //            raw[6]=temp_msb... 但這裡從 0xF7 開始讀
        // 實際上：0xF7=press_msb, 0xFA=temp_msb, 0xFD=hum_msb
        // raw[0]=0xF7(press_msb), raw[3]=0xFA(temp_msb)
        int32_t adc_T = (int32_t)((raw[3] << 12) | (raw[4] << 4) | (raw[5] >> 4));
        // 20-bit 溫度：raw[3] 的 8 位 + raw[4] 的 8 位 + raw[5] 的高 4 位

        int32_t adc_H = (int32_t)((raw[6] << 8) | raw[7]);
        // 16-bit 濕度：raw[6] 的 8 位 + raw[7] 的 8 位

        // Step 6：補償計算
        // 必須先算溫度！因為 Compensate_T 會更新 t_fine
        // t_fine 是溫度的中間值，濕度補償公式也需要它
        float temp = BME280_Compensate_T(adc_T);
        float hum  = BME280_Compensate_H(adc_H);

        // 用整數拆開印（避免開啟 float printf 支援）
        // 例如 26.17°C：t_int=26, t_frac=17
        int t_int  = (int)temp;
        int t_frac = (int)((temp - (float)t_int) * 100);
        int h_int  = (int)hum;
        int h_frac = (int)((hum  - (float)h_int) * 100);

        safe_printf("溫度: %d.%02d C | 濕度: %d.%02d %%\r\n",
                    t_int, t_frac, h_int, h_frac);

        osDelay(2000); // 每 2 秒讀一次
    }
```

### 5.5 溫度補償公式說明

```c
float BME280_Compensate_T(int32_t adc_T) {
    int32_t var1, var2, T;

    // var1 和 var2 是 Bosch 官方定義的中間計算值
    // 這是固定公式，不需要理解每個數字的含義
    // dig_T1/T2/T3 是從晶片讀出的校準係數
    var1 = ((((adc_T >> 3) - ((int32_t)dig_T1 << 1))) * ((int32_t)dig_T2)) >> 11;
    var2 = (((((adc_T >> 4) - ((int32_t)dig_T1)) *
              ((adc_T >> 4) - ((int32_t)dig_T1))) >> 12) *
             ((int32_t)dig_T3)) >> 14;

    // t_fine：溫度的精細中間值，濕度補償也需要它
    t_fine = var1 + var2;

    // 最終溫度 = t_fine * 5 + 128，右移 8 位
    // 結果單位是 0.01°C，除以 100 得到 °C
    T = (t_fine * 5 + 128) >> 8;
    return (float)T / 100.0f;
}
```

---

## 六、常見問題排查

| 現象 | 可能原因 | 解法 |
|------|---------|------|
| Chip ID 不是 0x60 | 接線錯誤 | 確認 SCK/MOSI/MISO/CS 接線 |
| 溫度全是 0.00 | ID 讀到但資料錯 | 確認 CPOL/CPHA 設定 |
| 板子一直重置 | CS 預設沒有 HIGH | `BME280_CS_HIGH()` 在初始化後馬上呼叫 |
| SPI1 和 LED 衝突 | PA6/PA7 是 LED 腳位 | 改用 SPI2（PB13/PC2/PC3） |

---

## 七、面試常考問題

**Q: SPI 和 I2C 各適合什麼場景？**
A: SPI 速度快，適合 Flash、顯示器、高速感測器；I2C 線少，適合多裝置、低速感測器（溫濕度、加速度計等）。

**Q: SPI 的 CPOL 和 CPHA 是什麼？**
A: CPOL 決定時脈閒置狀態（0=LOW, 1=HIGH），CPHA 決定在哪個邊沿採樣資料（0=第一個, 1=第二個）。兩者組合成 4 種 Mode，裝置必須配對相同 Mode 才能通訊。

**Q: BME280 為什麼需要校準參數？**
A: 每顆晶片的感測元件有製造誤差，出廠時 Bosch 對每顆晶片進行校準，把係數存在 ROM 裡。讀取原始數值後套用公式才能得到準確數值。

**Q: 為什麼必須先寫 0xF2（ctrl_hum）再寫 0xF4（ctrl_meas）？**
A: BME280 的設計是 ctrl_hum 的設定只有在 ctrl_meas 被寫入後才生效，順序寫反的話濕度 oversampling 不會套用。

**Q: 為什麼溫度補償必須在濕度補償之前呼叫？**
A: `Compensate_T` 會計算並更新全域變數 `t_fine`，`Compensate_H` 的計算公式依賴 `t_fine` 的值，所以必須先算溫度。

---

## 八、目前完成進度

- ✅ GPIO Output（LED 控制）
- ✅ GPIO Input Polling
- ✅ EXTI 中斷 + Debounce + Pull-up
- ✅ Timer 中斷（TIM3）
- ✅ UART printf 除錯
- ✅ FreeRTOS Task / Queue / Semaphore / Mutex
- ✅ SPI 通訊
- ✅ BME280 溫濕度感測器

---

## 九、下一步

- **SPI OLED 顯示器**：把溫濕度數值顯示在螢幕上
- **FatFS SD 卡記錄**：把感測資料寫入 SD 卡
- **整合**：多 Task 環境監測系統完整版

# STM32 GPIO 學習筆記
**DevEBox STM32F407VET6 | Day 1**

---

## 一、背景知識：GPIO 是什麼？

GPIO（General Purpose Input/Output）是 MCU 上最基本的腳位，可以透過程式控制它：

- **Output 模式**：讓腳位輸出 HIGH（3.3V）或 LOW（0V），用來控制 LED、繼電器等
- **Input 模式**：讀取外部訊號（按鈕、感測器），判斷是 HIGH 還是 LOW

STM32 的 GPIO 分成幾個 Port（GPIOA、GPIOB、GPIOC... GPIOE），每個 Port 有 16 個腳位（PIN_0 ~ PIN_15）。

---

## 二、LED 控制（GPIO Output）

### 2.1 板子 LED 腳位
| LED | 腳位 | 邏輯類型 |
|-----|------|---------|
| D2 | PA6 | Active Low |
| D3 | PA7 | Active Low |

### 2.2 什麼是 Active Low？

LED 的硬體接法：
```
PA6 ──── LED ──── VCC (3.3V)
```

- PA6 輸出 **LOW（0V）** → 兩端有電壓差 → 電流流動 → **亮**
- PA6 輸出 **HIGH（3.3V）** → 兩端沒電壓差 → 電流不流 → **滅**

所以「Active Low」= LOW 訊號代表啟動，與直覺相反。

| HAL 函數參數 | 電位 | D2/D3 狀態 |
|-------------|------|-----------|
| `GPIO_PIN_RESET` | LOW (0V) | 亮 ✅ |
| `GPIO_PIN_SET` | HIGH (3.3V) | 滅 |

### 2.3 基本閃爍程式碼

```c
while (1) {
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_RESET); // D2 亮
    HAL_Delay(500);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_SET);   // D2 滅
    HAL_Delay(500);
}
```

### 2.4 更簡潔：TogglePin

```c
while (1) {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6); // 每次呼叫就反轉狀態
    HAL_Delay(500);
}
```

### 2.5 HAL_Delay 底層原理

STM32 開機時啟動 **SysTick** 硬體計時器，每 1ms 產生一次中斷，把全域變數 `uwTick` 加 1。

```c
// HAL_Delay 概念上等同於：
uint32_t start = uwTick;
while ((uwTick - start) < ms) {
    // 空轉等待
}
```

`HAL_GetTick()` 可以讀取目前的 ms 數，常用來做計時邏輯。

---

## 三、按鈕輸入（GPIO Input）

### 3.1 板子按鈕腳位
| 按鈕 | 腳位 | 按下時訊號 | 說明 |
|------|------|-----------|------|
| K0 | PE4 | LOW | Active Low |
| K1 | PE3 | LOW | Active Low |
| K_UP | PA0 | HIGH | Active High |
| RST | — | — | 硬體重置，不可用 |

### 3.2 Polling 方式讀取按鈕

```c
while (1) {
    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_4) == GPIO_PIN_RESET) {
        // K0 按下（Active Low → RESET = LOW = 按下）
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_RESET); // D2 亮
    } else {
        // K0 放開
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_SET);   // D2 滅
    }
}
```

**Polling 的缺點**：MCU 100% 時間都在檢查按鈕，無法做其他事情。

---

## 四、中斷（Interrupt / EXTI）

### 4.1 Polling vs Interrupt

| | Polling | Interrupt |
|--|---------|-----------|
| 運作方式 | 主程式一直輪詢 | 硬體事件主動通知 MCU |
| CPU 使用率 | 100%（等待中） | 低（平常可做別的事） |
| 反應時機 | 下一次輪詢到才處理 | 事件發生瞬間立刻處理 |
| 適用場景 | 簡單、低頻率偵測 | 即時反應、多任務環境 |

### 4.2 CubeMX 配置 EXTI

1. PE4 → 選 `GPIO_EXTI4`
2. GPIO mode → **External Interrupt Mode with Falling edge trigger**
   - Falling edge = 電位從 HIGH 降到 LOW 的瞬間（= 按下按鈕）
3. NVIC → 啟用 `EXTI line4 interrupt`

### 4.3 Interrupt Callback 程式碼

中斷不寫在 `while(1)` 裡，而是寫在特殊的 callback 函數：

```c
/* USER CODE BEGIN 4 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_4) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6); // 每次按下切換 D2
    }
}
/* USER CODE END 4 */
```

**`HAL_GPIO_EXTI_Callback`** 是 HAL 預定義的弱函數（weak function），每次 EXTI 觸發時自動被呼叫。

### 4.4 背景知識：中斷的運作流程

```
1. MCU 正在執行 main() 的 while(1)
2. PE4 電位變化 → 硬體 EXTI 控制器偵測到 Falling Edge
3. EXTI 向 NVIC（中斷控制器）發出請求
4. NVIC 打斷 main()，跳到 EXTI4_IRQHandler()
5. IRQHandler 呼叫 HAL_GPIO_EXTI_Callback()
6. Callback 執行完畢，回到 main() 被打斷的地方繼續
```

---

## 五、按彈跳（Debounce）

### 5.1 問題：為什麼按一次可能觸發多次？

按鈕在物理按下的瞬間，金屬接點會快速抖動幾毫秒：

```
理想：  ▔▔▔╲___________
實際：  ▔▔▔╲_╱╲_╱╲____  ← 抖動，EXTI 被觸發多次！
```

### 5.2 Software Debounce 解法

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_4) {
        static uint32_t last_press = 0;  // static：函數結束後值不消失
        uint32_t now = HAL_GetTick();

        if ((now - last_press) > 50) {   // 距離上次超過 50ms 才算有效
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
            last_press = now;
        }
    }
}
```

**`static` 變數**：每次進入函數都保留上次的值，普通區域變數則每次重置。

---

## 六、Pull-up 電阻

### 6.1 問題：浮空（Floating）腳位

沒按按鈕時，PE4 什麼都沒接：
```
PE4 ──── 按鈕 ──── GND
         （開路）
```
PE4 電位不確定，手靠近或環境干擾都可能讓電位亂飄，觸發假中斷。

### 6.2 Pull-up 電阻解法

```
3.3V ──── 電阻（Pull-up） ──── PE4 ──── 按鈕 ──── GND
```

- **沒按時**：PE4 透過電阻被拉到 3.3V → 穩定讀到 HIGH
- **按下時**：PE4 直接接到 GND → 讀到 LOW
- **電阻的作用**：防止 3.3V 和 GND 直接短路，同時給 PE4 一個預設電位

STM32 晶片內建上拉/下拉電阻，在 CubeMX 設定 **Pull-up** 即可啟用，不需外接元件。

### 6.3 為什麼人體靠近會觸發？

浮空腳位像天線，人體帶有靜電和電磁干擾，可以把雜訊耦合進去讓電位短暫變化。加了 Pull-up 後，3.3V 把 PE4 牢牢固定，這點干擾能量不足以拉動電位。

---

## 七、面試常考問題整理

**Q: Active Low 是什麼？為什麼要用？**
A: Low 訊號代表啟動。硬體設計上有時 VCC 接 LED 另一端，這樣 LOW 才能讓電流流過。

**Q: Polling 和 Interrupt 的差異？各自適用場景？**
A: Polling 持續輪詢浪費 CPU，適合簡單場景；Interrupt 由硬體事件觸發，節省 CPU，適合即時系統和多任務環境。

**Q: 什麼是 Debounce？怎麼實作？**
A: 按鈕機械抖動導致多次觸發，解法是記錄上次觸發時間，50ms 內的重複觸發一律忽略。

**Q: 為什麼 GPIO Input 需要 Pull-up/Pull-down？**
A: 沒接東西的腳位電位浮空不確定，加上拉/下拉電阻給予預設值，避免誤讀。

---

## 八、後續學習步驟大綱

### 即將學習（Week 1 剩餘）

1. **Timer 中斷（TIM）**
   - 用硬體計時器定期觸發中斷
   - 理解 Prescaler、Period 設定
   - 為 FreeRTOS 任務切換打基礎

2. **UART 串列通訊**
   - 透過 USB 把資料印到電腦終端機
   - 學會 `printf` 重導向到 USART
   - 基礎除錯工具

### Week 2：FreeRTOS 多任務

3. **FreeRTOS 基礎**
   - Task 建立與優先權
   - `osDelay` vs `HAL_Delay` 的差異
   - Stack 大小設定

4. **FreeRTOS 任務間通訊**
   - Queue：傳遞資料
   - Semaphore：同步任務
   - Mutex：保護共享資源

5. **實際應用：多任務架構**
   - Task 1：讀取按鈕
   - Task 2：控制 LED
   - 兩個 Task 用 Queue 溝通

### Week 3-4：感測器與整合

6. **I2C 通訊 → BME280 溫濕度感測器**
7. **SPI 通訊 → OLED 顯示器**
8. **ADC → 光感測器**
9. **FatFS → SD 卡資料記錄**
10. **整合專案：環境監測系統**

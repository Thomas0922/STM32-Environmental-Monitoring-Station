# STM32 Timer 中斷 & UART 學習筆記
**DevEBox STM32F407VET6 | Day 1 續**

---

## 一、背景知識：STM32 程式碼區間說明

### 為什麼需要 USER CODE BEGIN/END？

CubeMX 每次 Generate Code 都會**重新覆蓋** `main.c`，如果你的程式碼不在指定區間內，就會被刪掉！

`/* USER CODE BEGIN xxx */` 和 `/* USER CODE END xxx */` 之間的程式碼是**受保護的**，Generate Code 不會動到它。

### 各區間的位置與用途

```c
/* USER CODE BEGIN Includes */
// 放 #include，例如 <stdio.h>
// 為什麼放這裡：確保 include 在所有 HAL 初始化之前載入
/* USER CODE END Includes */

/* USER CODE BEGIN 0 */
// 放全域變數、自定義函數宣告、printf 重導向
// 為什麼放這裡：在 main() 之前就存在，整個檔案都可以用
/* USER CODE END 0 */

int main(void) {

    /* USER CODE BEGIN 1 */
    // 放區域變數宣告
    // 為什麼放這裡：HAL_Init() 之前，但在 main() 裡面
    /* USER CODE END 1 */

    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_TIM3_Init();
    MX_USART1_UART_Init();

    /* USER CODE BEGIN 2 */
    // 放初始化程式碼，例如啟動 Timer、發送開機訊息
    // 為什麼放這裡：所有 HAL 初始化完成後，while loop 之前
    /* USER CODE END 2 */

    while (1) {
        /* USER CODE BEGIN WHILE */
        // 放主要邏輯（但用中斷的話通常留空）
        /* USER CODE END WHILE */

        /* USER CODE BEGIN 3 */
        // while loop 的補充邏輯
        /* USER CODE END 3 */
    }
}

/* USER CODE BEGIN 4 */
// 放中斷 Callback 函數
// 為什麼放這裡：在 main() 之外，和 HAL 的弱函數（weak function）同層
/* USER CODE END 4 */
```

### 放錯地方會怎樣？

| 錯誤情境 | 結果 |
|---------|------|
| 程式碼放在 USER CODE 區間外 | Generate Code 後被刪除 |
| 把 Callback 放在 USER CODE BEGIN 2 | 編譯錯誤（函數不能在函數裡面定義）|
| 把 HAL_TIM_Base_Start_IT 放在 while loop | Timer 每圈都重啟，行為不正確 |

---

## 二、背景知識：STM32 時脈架構

### 時脈樹（Clock Tree）

STM32F407 的時脈來源有三種：
- **HSI**：內部 RC 振盪器，16MHz，不需外部元件，但精度較差
- **HSE**：外部晶振（你的板子是 8MHz），精度高
- **PLL**：鎖相迴路，可以把低頻倍頻到 168MHz

你的板子配置：
```
HSE (8MHz) → PLL (/8 × 168 /2) → SYSCLK 168MHz
→ AHB Bus  → HCLK 168MHz（CPU、記憶體）
→ APB1 Bus → PCLK1 42MHz → Timer × 2 = 84MHz（TIM2~7）
→ APB2 Bus → PCLK2 84MHz → Timer × 2 = 168MHz（TIM1、TIM8~11）
```

### 重要：不同 Timer 用不同時脈！

| Timer | 掛在哪個 Bus | Timer 時脈 |
|-------|------------|-----------|
| TIM1 | APB2 | 168MHz |
| TIM2~7 | APB1 | **84MHz** |
| TIM8~11 | APB2 | 168MHz |

**TIM3 掛在 APB1，時脈是 84MHz，不是 168MHz！** 這是很常見的配置錯誤。

---

## 三、Timer 中斷（TIM3）

### 3.1 為什麼用 Timer 中斷？

| 方法 | 問題 |
|------|------|
| `HAL_Delay(500)` | 阻塞式，等待期間 MCU 什麼都不能做 |
| while loop 計時 | 浪費 CPU，且不精確 |
| **Timer 中斷** | 硬體自動計時，時間到了通知 MCU，CPU 平常可做別的事 ✅ |

### 3.2 觸發頻率公式

```
觸發頻率 = Timer時脈 / (Prescaler + 1) / (Period + 1)
```

想要每 500ms 觸發一次（TIM3，84MHz）：
```
84,000,000 / (8399 + 1) / (4999 + 1)
= 84,000,000 / 8400 / 5000
= 2 Hz（每 500ms 一次）✅
```

### 3.3 CubeMX 配置說明

**路徑：** Timers → TIM3

| 設定項目 | 值 | 原因 |
|---------|-----|------|
| Clock Source | Internal Clock | 使用內部時脈，不需外部訊號 |
| Prescaler | 8399 | 把 84MHz 降到 10KHz，方便計算 |
| Counter Period | 4999 | 10KHz 數 5000 次 = 0.5 秒 |
| NVIC → TIM3 global interrupt | Enabled | 允許 Timer 觸發中斷，沒打勾就不會呼叫 Callback |

**Prescaler 的意義：**
把高頻時脈降成低頻，讓 Period 的數字不用太大。
84MHz 直接用的話，Period 要設 41,999,999 才能得到 500ms，超過 16-bit 的上限（65535）。

### 3.4 程式碼說明

```c
/* USER CODE BEGIN 2 */
HAL_TIM_Base_Start_IT(&htim3);
// 啟動 TIM3 並開啟中斷模式（_IT 後綴 = Interrupt）
// 為什麼放 BEGIN 2：所有初始化完成後才能啟動 Timer
/* USER CODE END 2 */
```

```c
/* USER CODE BEGIN 4 */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    // 所有 Timer 溢出都會呼叫這個函數
    // 必須用 htim->Instance 判斷是哪個 Timer 觸發的
    if (htim->Instance == TIM3) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
    }
}
// 為什麼放 BEGIN 4：這是函數定義，必須在 main() 之外
/* USER CODE END 4 */
```

**HAL_TIM_PeriodElapsedCallback** 是 HAL 內建的弱函數（`__weak`），你重新定義它就會覆蓋掉原本空的版本。

---

## 四、UART 串列通訊

### 4.1 什麼是 UART？

UART（Universal Asynchronous Receiver-Transmitter）是最簡單的串列通訊協定，只需要兩條線：

- **TX**（Transmit）：傳送資料
- **RX**（Receive）：接收資料

兩端必須用同樣的速率通訊，這個速率叫做 **Baud Rate**，常用 115200 bps。

### 4.2 你的板子接線

```
STM32 PA9  (TX1) ──── UART轉USB模組 RX ──── 電腦
STM32 PA10 (RX1) ──── UART轉USB模組 TX ──── 電腦
STM32 GND        ──── UART轉USB模組 GND
```

**注意：TX 接 RX、RX 接 TX，永遠交叉接。**

### 4.3 CubeMX 配置說明

**路徑：** Connectivity → USART1

| 設定項目 | 值 | 原因 |
|---------|-----|------|
| Mode | Asynchronous | 非同步，不需要時脈訊號線，最常用 |
| Baud Rate | 115200 | 電腦終端機預設值，兩端必須一致 |
| Word Length | 8 Bits | 標準資料位元長度 |
| Parity | None | 不做奇偶校驗，簡單場景夠用 |
| Stop Bits | 1 | 標準停止位元 |

### 4.4 printf 重導向說明

C 語言的 `printf` 預設輸出到 stdout，但嵌入式系統沒有螢幕，需要把輸出重導向到 UART。

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>

int __io_putchar(int ch) {
    // __io_putchar 是 printf 底層呼叫的函數
    // 覆蓋它，讓每個字元透過 UART 送出去
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
// 為什麼放 BEGIN 0：全域函數，整個程式都需要用到
/* USER CODE END 0 */
```

**為什麼加 `#include <stdio.h>`？**
`printf` 定義在 stdio.h，沒有 include 就無法使用。

**`HAL_MAX_DELAY` 是什麼？**
等待傳送完成的超時時間，設成最大值代表「等到傳完為止，不超時」。

### 4.5 測試程式碼

```c
/* USER CODE BEGIN 2 */
HAL_TIM_Base_Start_IT(&htim3);
printf("System started! UART OK.\r\n"); // 開機確認訊息
/* USER CODE END 2 */

while (1) {
    printf("Hello from STM32!\r\n");
    HAL_Delay(1000);
    /* USER CODE END WHILE */
}
```

**為什麼用 `\r\n` 不用 `\n`？**
Windows 終端機需要 `\r\n`（Carriage Return + Line Feed）才能正確換行，只用 `\n` 有時候會排版跑掉。

### 4.6 PuTTY 設定

| 設定 | 值 |
|------|-----|
| Connection type | Serial |
| Serial line | COM5（你的板子）|
| Speed | 115200 |

---

## 五、面試常考問題

**Q: Prescaler 和 Period 怎麼計算？**
A: 觸發頻率 = Timer時脈 / (Prescaler+1) / (Period+1)。先用 Prescaler 把時脈降到容易計算的數字，再用 Period 設定計數次數。注意 STM32F407 的 Timer 最大是 16-bit（最大值 65535）。

**Q: TIM3 的時脈是幾MHz？**
A: TIM3 掛在 APB1 Bus，PCLK1 = 42MHz，Timer × 2 = **84MHz**。這是常見陷阱，不是 SYSCLK 的 168MHz。

**Q: printf 在嵌入式系統怎麼用？**
A: 覆蓋 `__io_putchar()` 函數，把輸出重導向到 UART，讓 `printf` 透過串列埠傳到電腦終端機。

**Q: `HAL_TIM_Base_Start` 和 `HAL_TIM_Base_Start_IT` 的差別？**
A: 沒有 `_IT` 只是啟動 Timer 計數，不會觸發中斷；有 `_IT` 才會在 Period 溢出時呼叫 Callback。

---

## 六、目前完成進度

- ✅ GPIO Output（LED 控制）
- ✅ GPIO Input Polling（按鈕讀取）
- ✅ EXTI 中斷（按鈕中斷）
- ✅ Debounce + Pull-up
- ✅ Timer 中斷（TIM3）
- ✅ UART printf 除錯

---

## 七、下一步：FreeRTOS

### 為什麼要學 FreeRTOS？

目前所有功能都在同一個 while loop 或中斷裡，當功能越來越多就會遇到：
- 不同功能互相干擾（例如 Delay 阻塞其他功能）
- 程式碼難以維護
- 無法處理即時性要求不同的任務

FreeRTOS 的解法是把程式分成多個獨立的 **Task**，由 RTOS 負責排程，讓每個 Task 以為自己獨占 CPU。

### Timer 中斷和 FreeRTOS 的關係

FreeRTOS 的任務切換底層就是靠 **SysTick 中斷** 驅動的，每次 SysTick 觸發，RTOS 的排程器就決定要切換到哪個 Task。

所以理解 Timer 中斷是進入 FreeRTOS 的必要基礎。

# TwinCAT 3 Structured Text (ST) 程式碼風格指南

本指南定義了本專案中所有 Beckhoff TwinCAT 3 結構化文字 (ST) 的編碼規範。AI Agent 與開發團隊在新增、修改或重構 PLC 程式碼時，必須嚴格遵守此標準。

---

## 1. 核心原則 (Core Principles)
* **確定性 (Determinism)：** 所有程式碼必須在 TwinCAT 實時循環工作 (Cyclic Task) 的掃描週期內穩定執行，嚴禁任何阻塞風險。
* **可移植性：** 遵循 IEC 61131-3 第 3 版標準，除非明確指定，否則避免使用特定廠牌的非標準私有函數。
* **高唯讀性：** 程式碼的結構必須清晰，命名必須具備自我描述能力 (Self-documenting)。

---

## 2. 命名規範 (Naming Conventions)

所有變數、常數及物件宣告必須使用**匈牙利命名法 (Hungarian Notation)** 搭配 **PascalCase (首字母大寫)**。

### 2.1 變數型別前綴 (Variable Prefixes)
* `x` : 布林值 (BOOL) —— 專案內內部邏輯優先使用（例如：`xMotorStart`）
* `b` : 位元組 / 布林值 (BYTE / BOOL) —— 依標準庫習慣使用（例如：`bEnable`）
* `i` : 整數 (INT, 16位元) —— 例如：`iCycleCount`
* `di`: 雙整數 (DINT, 32位元) —— 例如：`diEncoderValue`
* `ui`: 無號整數 (UINT, 16位元) —— 例如：`uiLimitValue`
* `r` : 實數 (REAL, 32位元浮點數) —— 例如：`rCurrentTemperature`
* `lr`: 長實數 (LREAL, 64位元浮點數) —— 精度要求高時使用，例如：`lrPosition`
* `s` : 字串 (STRING) —— 例如：`sStatusMessage`
* `t` : 時間 (TIME) —— 例如：`tDelayTime`
* `ast`: 結構陣列 (ARRAY of STRUCT) —— 例如：`astAxisData`
* `a`  : 一般陣列 (ARRAY) —— 例如：`aDataBuffer : ARRAY[0..9] OF INT;`

### 2.2 作用域與修飾詞前綴 (Scope Prefixes)
* `g_` : 全域變數 (Global Variables, GVL) —— 例如：`g_xEmergencyStop`
* `c_` : 常數 (Constants) —— 例如：`c_rMaxTemp`

### 2.3 程式組織單元與物件前綴 (POU & Object Prefixes)
* `FB_` : 功能塊 (Function Block) —— 例如：`FB_ValveControl`
* `FC_` : 函式 (Function) —— 例如：`FC_CalculateScaling`
* `I_`  : 介面 (Interface) —— 物件導向開發使用，例如：`I_Actuator`
* `E_`  : 列舉 (Enumeration) —— 狀態機專用，例如：`E_MachineState`
* `ST_` : 結構 (Structure) —— 例如：`ST_DriveStatus`
* `inst`: FB 實例化名稱 (Instance) —— 例如：`instMainValve : FB_ValveControl;`
* `ip`  : 介面指標 (Interface Pointer) —— 例如：`ipSensor : I_Sensor;`

---

## 3. 程式碼風格與格式化 (Formatting Rules)

* **關鍵字大小寫：** 所有 IEC 61131-3 關鍵字必須保持**全大寫**（例如：`IF`, `THEN`, `CASE`, `OF`, `END_IF`, `VAR_INPUT`, `TRUE`, `FALSE`）。
* **縮排規範：** 每層縮排固定使用 **4 個空格**，嚴禁使用 Tab 鍵。
* **狀態機規範：** 所有狀態機必須使用強型別列舉 (`ENUM`) 搭配 `CASE` 陳述式。定義列舉時必須加上 TwinCAT 的嚴格屬性標籤：
  ```pascal
  {attribute 'qualified_only'}
  {attribute 'strict'}
  TYPE E_MachineState :
  (
      Idle    := 0,
      Running := 10,
      Error   := 99
  ) INT;
  END_TYPE
  ```

---

## 4. TwinCAT 專屬硬體映射 (I/O Mapping)

所有與物理硬體連結的輸入/輸出變數，必須在全域變數表 (GVL) 中使用 `AT %I*` 與 `AT %Q*` 語法宣告，以便透過 TwinCAT 進行自動 I/O 連結：

```pascal
VAR_GLOBAL
    // 硬體數位輸入映射
    xDI_EmergencyStop AT %I* : BOOL; // 急停按鈕輸入
    xDI_StartButton   AT %I* : BOOL; // 啟動按鈕輸入

    // 硬體數位輸出映射
    xDO_MainContactor AT %Q* : BOOL; // 主接觸器輸出
    xDO_AlarmLamp     AT %Q* : BOOL; // 警報指示燈輸出
END_VAR
```

---

## 5. 註解與文件規範 (Comments)

根據專案語言設定，**所有邏輯說明必須使用繁體中文註解**。
* **POU 檔頭：** 每個 POU 的開頭必須包含檔頭註解，說明其功能、輸入、輸出。
* **邏輯說明：** 程式碼內部的行內註解 (`//`) 應該解釋「為什麼這麼做（邏輯目的）」，而不是複製程式碼字面意思。

---

## 6. 標準程式碼範本 (Standard Blueprint)

以下為符合本專案規範的 TwinCAT 3 ST 標準狀態機與控制邏輯範本：

```pascal
// =================================================================
// 物件名稱: FB_ConveyorControl
// 功能描述: 處理輸送帶的狀態切換、安全互鎖與運行逾時監控
// =================================================================
FUNCTION_BLOCK FB_ConveyorControl
VAR_INPUT
    xStart          : BOOL;          // 啟動命令
    xStop           : BOOL;          // 停止命令
    xInterlock      : BOOL;          // 安全互鎖訊號（必須為 TRUE 才能運轉）
    tRunTimeout     : TIME := T#5S;  // 運轉逾時設定值（預設 5 秒）
END_VAR
VAR_OUTPUT
    xMotorOn        : BOOL;          // 輸出：馬達運轉控制
    xAlarm          : BOOL;          // 輸出：警報觸發
    eCurrentState   : E_ConveyorState := E_ConveyorState.Idle; // 輸出：當前狀態
END_VAR
VAR
    instTimer       : TON;           // 逾時監控計時器實例
END_VAR

// --- 邏輯開始 ---

// 1. 安全與互鎖檢查 (最高優先權)
IF NOT xInterlock THEN
    eCurrentState := E_ConveyorState.Idle; // 強制跳回閒置/安全狀態
END_IF;

// 2. TwinCAT 最佳化狀態機邏輯
CASE eCurrentState OF
    E_ConveyorState.Idle: // 閒置狀態
        xMotorOn := FALSE;
        xAlarm   := FALSE;
        
        // 條件滿足，切換至運轉狀態
        IF xStart AND xInterlock THEN
            eCurrentState := E_ConveyorState.Running;
        END_IF;
        
    E_ConveyorState.Running: // 運轉狀態
        xMotorOn := TRUE;
        
        // 收到停止訊號，正常切換回閒置
        IF xStop THEN
            eCurrentState := E_ConveyorState.Idle;
        END_IF;
        
        // 運轉逾時監控（馬達運轉時啟動計時）
        instTimer(IN := xMotorOn, PT := tRunTimeout);
        IF instTimer.Q THEN
            eCurrentState := E_ConveyorState.Error; // 計時到達，觸發錯誤狀態
        END_IF;
        
    E_ConveyorState.Error: // 錯誤狀態
        xMotorOn := FALSE;
        xAlarm   := TRUE;
        
        // 按下停止鍵進行解除警報與重置
        IF xStop THEN
            eCurrentState := E_ConveyorState.Idle;
        END_IF;
END_CASE;
```

---

## 7. 禁忌與嚴格禁止事項 (Anti-Patterns)

* **嚴禁無限/未知次數迴圈：** 實時循環工作 (Cyclic Task) 中**絕對不可以使用** `WHILE` 或 `REPEAT` 迴圈。如果迴圈條件未被觸發，會直接導致 TwinCAT 實時看門狗 (Watchdog) 超時，進而造成電腦藍屏死機 (BSOD)。
* **實時層嚴禁動態記憶體分配：** 不可在循環執行程式碼中調用 `__NEW` 或 `__DELETE`。所有內存必須在編譯期靜態配置完成。
* **嚴禁未觸發的高頻日誌：** 使用 `ADSLOGSTR` 等系統日誌函式時，**必須**使用正緣觸發 (`R_TRIG`)。嚴禁在每個掃描週期無條件調用，這會造成 TwinCAT ADS 訊息緩衝區溢位 (Buffer Overflow)。
* **不使用 Magic Numbers：** 所有時間設定、物理極限值、特殊常數，必須使用 `VAR CONSTANT`（如 `c_rMaxTemp`）來定義，嚴禁直接在程式碼中寫死數字。
* **零警告原則：** 所有 TwinCAT 編譯器產生的警告 (Compiler Warnings) 必須視同錯誤處理，專案最終編譯必須達到 0 錯誤、0 警告。

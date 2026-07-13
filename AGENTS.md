# AGENTS.md

## 1. 專案環境與倉庫狀態 (Workspace Status)
* **當前狀態：** 全新乾淨的倉庫區 (`LINcalvin/Opencode_REP`) — 僅有初始提交，尚無任何應用程式碼。
* **遠端倉庫：** `https://github.com/LINcalvin/Opencode_REP.git`
* **工具配置規範：** 當前未配置任何建置 (Build)、測試 (Test)、程式碼檢查 (Lint) 或 CI 流程。**在新增任何應用程式碼之前或同時，必須先設定好相對應的自動化工具與環境。**

## 2. 語言與溝通設定 (Language Settings)
* **核心原則：** 所有的思考過程（Thinking Process）與對話回覆，一律必須使用**「繁體中文」**。
* **強制中文：** 即使用戶使用英文提問，AI 仍需堅持以**繁體中文**回覆。
* **程式碼註解：** 程式碼中的註解請盡量使用**中文**（變數、型別與函式命名等程式主體除外）。

## 3. TwinCAT 3 ST 開發核心規範 (Core Coding Mandate)
由於本專案為 Beckhoff TwinCAT 3 自動化專案，AI 在此倉庫中建立、修改或重構任何 **Structured Text (ST)** 程式碼或 POU 時，**必須嚴格遵守 `docs/ST_STYLE_GUIDE.md` 所定義的規範**。

為了確保開發品質，AI 必須在**新增第一行 PLC 程式碼之前**，先在專案中建立 `docs/ST_STYLE_GUIDE.md` 檔案，其核心架構需包含：
1. **命名匈牙利命名法：** 變數需帶有型別前綴（如 `x` 預留給 BOOL，`i` 給 INT，`inst` 給 FB 實例，`e` 給 ENUM）。
2. **強型別狀態機：** 狀態機必須使用帶有 `{attribute 'qualified_only'}` 與 `{attribute 'strict'}` 的明文 `ENUM` 搭配 `CASE` 語法，禁止使用 Magic Numbers。
3. **硬體映射：** I/O 宣告必須使用 `AT %I*` 與 `AT %Q*` 的自動連結語法。
4. **實時安全 (Real-time Safety)：** 嚴禁在實時循環工作（Cyclic Task）中使用 `WHILE`/`REPEAT` 等不確定次數的迴圈，亦不得在循環中執行動態記憶體分配 (`__NEW`)，避免觸發 TwinCAT Watchdog 導致藍屏 (BSOD)。
5. **邏輯註解：** 承襲語言設定，程式碼內部的邏輯說明必須使用**中文註解**。

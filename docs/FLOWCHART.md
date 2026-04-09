# 流程圖設計 (Flowchart) - 任務管理系統

根據 PRD 與系統架構文件，本文件規劃了使用者的操作流程圖、系統資料處理的序列圖，以及功能對應列表。

## 1. 使用者流程圖 (User Flow)

描述使用者進入系統後，可以進行的各項操作路徑。

```mermaid
flowchart LR
    Start([使用者開啟網頁]) --> Home[首頁 - 任務清單總覽]
    
    Home --> Action{要執行什麼操作？}
    
    %% 新增任務流程
    Action -->|新增任務| InputForm[在輸入框輸入任務名稱]
    InputForm --> SubmitBtn[點擊「新增」按鈕]
    SubmitBtn --> |系統處理中| ReloadHome1[重新載入並顯示新任務]
    ReloadHome1 --> Home
    
    %% 狀態篩選流程
    Action -->|切換視角| Filter[點擊狀態篩選 (全部/未完成/已完成)]
    Filter --> |載入特定任務| ReloadHome2[重新載入過濾後的清單]
    ReloadHome2 --> Home
    
    %% 狀態切換流程
    Action -->|更新狀態| ToggleStatus[點擊任務旁的狀態切換開關]
    ToggleStatus --> |標記為已完成/未完成| ReloadHome3[狀態更新並重新載入]
    ReloadHome3 --> Home
    
    %% 刪除任務流程
    Action -->|刪除任務| DeleteBtn[點擊「刪除」按鈕]
    DeleteBtn --> |移除資料| ReloadHome4[任務消失並重新載入]
    ReloadHome4 --> Home
```

## 2. 系統序列圖 (Sequence Diagram)

描述「使用者新增任務」時，資料如何在系統各元件間流動。

```mermaid
sequenceDiagram
    actor User as 使用者
    participant Browser as 瀏覽器 (HTML 介面)
    participant Flask as Flask Route (Controller)
    participant Model as Task Model 
    participant DB as SQLite (資料庫)

    User->>Browser: 在欄位輸入任務名稱並點擊提交表單
    Browser->>Flask: 發送 POST 請求 (/tasks/add) 帶入任務名稱
    
    %% 驗證與處理
    Flask->>Flask: 驗證輸入內容是否為空
    alt 驗證失敗 (空任務)
        Flask-->>Browser: 重導向至首頁，並顯示錯誤提示
    else 驗證成功
        Flask->>Model: 呼叫新增任務邏輯
        Model->>DB: 執行 SQL INSERT INTO Task
        DB-->>Model: 回傳成功訊息與 Task ID
        Model-->>Flask: 任務建立完成
        Flask-->>Browser: 重導向至首頁 (302 Redirect to /)
    end
    
    %% 重導向後的網頁載入
    Browser->>Flask: 發送 GET 請求 (/)
    Flask->>Model: 查詢最新任務清單
    Model->>DB: 執行 SQL SELECT * FROM Task
    DB-->>Model: 回傳所有的任務資料
    Model-->>Flask: 回傳 Task 物件清單
    Flask-->>Browser: 渲染 Jinja2 模板並回傳 HTML
    Browser-->>User: 看到畫面上顯示剛新增的任務清單
```

## 3. 功能清單對照表

以下整理了系統主要功能、其對應的 URL 路徑，以及前端發送的 HTTP 方法：

| 功能名稱 | URL 路徑 | HTTP 方法 | 說明 |
| --- | --- | --- | --- |
| 顯示清單與篩選 | `/` | GET | 顯示任務清單。可透過 Query Parameter（如 `/?filter=completed`）顯示過濾後的任務 |
| 新增任務 | `/tasks/add` | POST | 接收表單內容並新增任務，成功後重新導向至 `/` |
| 切換完成狀態 | `/tasks/<int:task_id>/toggle` | POST | 切換該任務的完成狀態（未完成變已完成，反之亦然），完成後重新導向至 `/` |
| 刪除任務 | `/tasks/<int:task_id>/delete` | POST | 刪除特定任務，成功後重新導向至 `/` |

> **說明**：由於初期架構採傳統網頁應用（SSR）、HTML 表單提交，為確保資料安全與遵循 HTTP 標準，狀態切換與刪除等會改變資料狀態的操作，皆使用 `POST` 方法發送至伺服器處理。

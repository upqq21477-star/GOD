# 消費自制力助手 — MVP 產品企劃書

版本：MVP v1.0
文件性質：工程技術企劃
產品定位：本地端數位消費自制力工具（Local-first Spending Awareness Tool）

---

## 一、產品簡介

### 背景與動機

現代人面對行動支付的普及，消費門檻大幅降低。打開一個 App、點幾下，錢就花出去了。使用者往往在月底才驚覺支出遠超預期，但此時已無法挽回。

市面上現有的記帳 App 多為被動記錄，需要使用者主動輸入，難以改變消費當下的決策行為。而支付攔截類工具則因涉及金融流程干預，面臨極高的平台審核與法律風險。

本產品填補兩者之間的空白：

> 在使用者打開支付 App 的當下，主動提供消費現況提醒，讓理性有機會介入衝動。

### 產品定位

本產品不是銀行 App、不是記帳 SaaS、不是支付攔截器。

而是「數位消費自制力助手」——一個在關鍵時刻給使用者一個暫停鍵的工具。

### 核心主張

> All financial analysis happens locally on your device.
> 所有財務分析僅在您的裝置上運行，不上傳、不追蹤、不建立帳號。

這是本產品與市場上其他金融 App 最根本的差異化。

### 目標使用者

- 有衝動消費習慣、希望建立自我控制的個人
- 對隱私敏感、不信任雲端金融服務的使用者
- 預算管理意識薄弱、需要行為提醒輔助的族群

---

## 二、核心功能

### 功能一｜高風險 App 偵測

使用 Android `UsageStatsManager` 監聽前景 App 切換事件。

預設高風險 App 清單（使用者可自訂）：

- 蝦皮購物 com.shopee.tw
- momo 購物網 com.mservice.momoshop
- PayPal com.paypal.android
- LINE Pay com.linecorp.linepay
- 街口支付 com.jkos.app
- PChome 24h tw.com.pchome.android24h

偵測到上述 App 切換至前景時，系統查詢當前超支狀態，決定是否觸發提醒視窗。

已知限制：

- 輪詢延遲約 1–3 秒，非即時
- 部分 OEM（MIUI、EMUI、ColorOS）背景殺程序策略嚴格，服務穩定性有差異
- UX 文案須標示「提醒可能延遲數秒出現」，切勿宣稱「即時攔截」

---

### 功能二｜消費通知自動記帳

透過 `NotificationListenerService` 監聽銀行與支付 App 推播，解析消費金額、商家、時間，自動寫入本地帳本。

解析流程：

```
收到通知
  ↓
過濾來源（銀行、支付平台白名單）
  ↓
Regex 解析金額與類型
  ↓
判斷類型：EXPENSE / REFUND / TRANSFER / UNKNOWN
  ↓
寫入本地 SQLite
  ↓
更新月支出統計
```

Regex 規則範例：

```regex
# 抓金額
(?:NT\$|TWD|\$)\s?([0-9,]+)

# 消費關鍵字
消費|付款|支付|刷卡|扣款

# 退款關鍵字
退款|退費|退貨|返還
```

注意：各銀行通知格式不統一，初期解析準確率約 60–70%。統計數字需於 UI 上標示為「估算值」，並提供人工修正入口。

---

### 功能三｜月支出統計

以當月為單位，統計各分類消費總額，顯示內容包含：

- 總支出估算金額
- 各分類佔比（購物、餐飲、娛樂等）
- 與上月金額比較
- 距預算剩餘金額

---

### 功能四｜預算設定與警告

使用者可依分類設定月預算上限：

- 達 70%：推播提醒通知
- 達 90%：強化提醒通知
- 超過 100%：標記為「超支狀態」，啟用彈跳提醒機制

---

### 功能五｜超支彈跳提醒視窗（核心機制）

詳見第三章。

---

### 功能六｜冷靜倒數計時

彈跳視窗中附帶 5 秒倒數，讓使用者在決策前擁有緩衝時間。

倒數結束後，使用者可選擇：

- 繼續進入 App
- 暫時不了（關閉提醒）
- 查看本月支出明細

---

### 功能七｜手動補登記帳

提供簡易輸入介面，補漏通知未能捕捉的消費：

- 現金消費
- 夜市、路邊攤、小店
- 未開啟通知的支付平台
- 跨境小額付款

---

## 三、核心重點：超支彈跳提醒視窗機制

### 設計哲學

> 不阻止，只提醒。讓理性有機會在衝動之前介入。

本機制的目標不是強制阻擋消費，而是在使用者最容易衝動消費的時刻，提供一個認知暫停點。

這符合行為經濟學的 Nudge Theory（輕推理論）：不剝奪選擇，而是在決策環境中植入有效提示，引導更理性的行為。

---

### 觸發條件

同時滿足以下兩個條件，才觸發彈跳提醒視窗：

條件 A：使用者當月某分類消費已超過預算（超支狀態）
條件 B：使用者開啟高風險購物或支付 App

---

### 視窗內容設計

彈跳視窗為半透明提醒卡，非全螢幕鎖定，使用者可隨時關閉。

```
┌─────────────────────────────────┐
│  ⚠️  本月購物支出已超出預算         │
│                                 │
│  預算上限     NT$ 3,000          │
│  本月已花     NT$ 3,847  (+28%) │
│                                 │
│  你最近 7 天已開啟蝦皮 14 次      │
│                                 │
│  ████████████████░░  128%       │
│                                 │
│  [  繼續進入  ]  [ 暫時不了 ]    │
│                                 │
│       5 秒後可繼續...            │
└─────────────────────────────────┘
```

---

### 分級提醒機制

根據超支程度，調整提醒強度：

Level 1（預算 70–89%）
推播通知，不彈視窗，低干擾。

Level 2（預算 90–99%）
輕量提醒卡 + 3 秒倒數。

Level 3（超過 100%）
完整提醒卡 + 5 秒倒數 + 當月支出統計。

Level 4（超過 150%）
完整提醒卡 + 10 秒倒數 + 本月消費明細。

---

### 技術實作方式

偵測層使用 `UsageStatsManager`：

```java
UsageStatsManager usm = (UsageStatsManager)
    context.getSystemService(Context.USAGE_STATS_SERVICE);
// 偵測到高風險 App → 查詢當月超支狀態 → 決定是否觸發
```

提醒層使用 System Alert Window（Overlay）：

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```

顯示半透明提醒卡，非全螢幕，非強制鎖定，使用者可隨時關閉或繼續。

嚴格禁止的實作方式：

- 全螢幕鎖死，無法關閉
- 攔截或阻止使用者操作
- 使用 AccessibilityService 讀取支付 App 畫面
- 偽裝成系統對話框

---

### 使用者控制權保障

使用者永遠擁有最終控制權，提醒設定選項包含：

- 可關閉特定 App 的提醒
- 可調整觸發閾值（例：改為超過 120% 才提醒）
- 可設定免打擾時段
- 可完全停用彈跳機制

---

## 四、資料架構設計

### transactions（消費記錄）

```sql
CREATE TABLE transactions (
    id              INTEGER PRIMARY KEY,
    amount          INTEGER NOT NULL,
    merchant        TEXT,
    category        TEXT,
    source          TEXT NOT NULL,
    -- AUTO 或 MANUAL
    locked          INTEGER DEFAULT 0,
    -- AUTO 記錄不可刪除
    excluded        INTEGER DEFAULT 0,
    -- 排除統計但保留原始紀錄
    tx_type         TEXT DEFAULT 'EXPENSE',
    -- EXPENSE / REFUND / TRANSFER / UNKNOWN
    correction_note TEXT,
    -- 使用者標記誤判原因
    created_at      DATETIME,
    raw_text        TEXT
    -- 原始通知文字備份
);
```

AUTO 記錄透過 `excluded` 欄位處理誤判，而非直接刪除。這防止使用者刪除紀錄作弊，同時保留通知誤判時的修正出口。

### app_events（App 開啟記錄）

```sql
CREATE TABLE app_events (
    id           INTEGER PRIMARY KEY,
    package_name TEXT,
    opened_at    DATETIME
);
```

### budgets（預算設定）

```sql
CREATE TABLE budgets (
    category      TEXT PRIMARY KEY,
    monthly_limit INTEGER
);
```

---

## 五、隱私設計原則

### 核心立場

本產品不收集、不上傳任何使用者資料。所有財務資訊僅存於使用者裝置的本地 SQLite 資料庫中。

### 正確的隱私聲明方式

不應寫：「我們不處理任何個資」
應該寫：「所有資料僅儲存於您的裝置，不會上傳至我們的伺服器」

不應寫：「本 App 沒有個資」
應該寫：「本 App 在裝置本地處理財務資訊，不建立雲端帳號」

法律說明：即使資料不離開裝置，讀取銀行通知、消費金額、商家名稱，在《個人資料保護法》下仍屬於「處理個資」，只是處理範圍限於裝置本地。務必在隱私政策中如實說明。

### MVP 隱私技術要求

- 完全不使用 Firebase Analytics
- 完全不使用 Crashlytics / Mixpanel / AppsFlyer
- Crash log 本地化儲存
- 不申請 INTERNET 權限（若 MVP 功能完全離線）
- 設定頁顯示本地模式狀態

### 設定頁隱私狀態顯示

```
✓ No cloud sync
✓ No external API upload
✓ No account required
✓ Data stored only on this device
```

---

## 六、MVP 開發範圍

### 納入 MVP v1

- NotificationListenerService 消費通知解析
- UsageStatsManager App 前景偵測
- 本地 SQLite 帳本
- 月支出統計與分類
- 超支彈跳提醒視窗（核心機制）
- 預算設定與警告
- 手動補登記帳

### MVP v1 排除

AccessibilityService：平台審核風險極高，接近惡意軟體行為。

Overlay 強制鎖定：違反 Google Play 政策風險。

銀行 App 登入 / OCR：涉及金融資安與法規風險。

雲端帳號同步：破壞 local-first 核心賣點。

AI 聊天 / 大模型：非 MVP 必要功能。

比價、投資、自動理財：超出產品定位。

Firebase Analytics：與隱私立場直接衝突。

---

## 七、已知技術風險

OEM 背景殺程序
小米、華為等嚴格限制背景服務存活。
應對：MVP 僅保證原生 Android 與 Samsung 穩定；文案標示「提醒可能延遲數秒」。

通知解析誤判
退款、分期、境外刷卡格式複雜。
應對：統計標示「估算值」；提供 excluded 欄位排除誤判紀錄。

個資聲明錯誤
Google Play Data Safety Form 填寫不實風險。
應對：上架前逐項確認資料流，不誇大「零個資」說法。

使用者棄用
提醒過於干擾導致卸載。
應對：提醒強度可由使用者自訂，保障完整控制權。

---

## 八、開發里程碑（估算）

Week 1–2
SQLite 架構建立、手動記帳 UI。

Week 2–3
NotificationListenerService 開發與 Regex 規則建立。

Week 3–4
UsageStatsManager App 前景偵測。

Week 4–5
超支彈跳提醒視窗（核心機制）開發。

Week 5–6
月統計 UI、預算設定、整合測試。

Week 6
多機型測試（Pixel、Samsung、小米）、隱私聲明審查、上架準備。

---

本企劃書內容依據工程分析文件彙整，所有技術評估基於 Android 官方文件與業界實戰經驗。

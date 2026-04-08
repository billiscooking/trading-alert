# 交易團隊緊急警報系統

> 緊急市場事件發生時，把團隊成員叫醒，確保不錯過交易機會。

## 專案背景

- **等級架構**（由低到高）：
  - **L1 - Telegram 推播**：成本零，自由度最高，首選（主動叫醒）
  - **L2 - Email**：GAS `MailApp`，免費，被動通知／留紀錄用（已完成：BTC 價格警報）
  - **L4 - 自動電話**：Twilio Voice API，約 $0.013/分；適用成員**在外面**或**在家睡覺**時，直接打到手機（主動叫醒）
  - **L5 - 物理警鈴（Tapo P105）**：智能插座接通電即響裝置；適用電話打不醒的極端情況（主動叫醒）
- **排除方案**：WhatsApp（封號風險）、Messenger（需對方 opt-in）、WeChat（無個人 API）、Signal（穿透力不足，無優勢）、SMS（電話和 Email 已覆蓋）
- **觸發條件**：市場異常事件 / 手動觸發 → 依等級升級，直到有人確認回應
- **核心設計**：確認回覆機制（有人回應即停止升級，避免過度打擾）
- **硬體**：已有 TP-Link Tapo P105 智能插座，App 支援裝置共享
- **關鍵發現**：iOS 靜音模式 + 勿擾模式下，只要把 Twilio 號碼加入勿擾白名單，來電仍有聲音

---

## 快速操作（L2 BTC 價格警報）

### 安裝
1. 開新的 Google Apps Script 專案（script.google.com）
2. 貼上 `l2_btc_price_alert.gs` 的內容
3. 執行 `setup()` → 建立每 5 分鐘觸發器
4. 首次執行需授權 `MailApp` 和 `UrlFetchApp`

### 停止 / 重啟
```
setup()   // 重啟監控（清除歷史，重新開始）
stop()    // 停止監控
```

### 測試
```
testFetchPrice()  // 確認能抓到 BTC 價格
testAlert()       // 寄一封測試警報信到信箱
```

### 警報門檻

| 時間窗口 | 漲/跌幅 | 情境 |
|----------|---------|------|
| 5 分鐘 | ±3% | Flash crash |
| 30 分鐘 | ±5% | 重大行情 |
| 1 小時 | ±8% | 黑天鵝 |

同一條件觸發後 30 分鐘冷卻，不會連續轟炸。

---

## 快速操作（L4 自動電話）

### 觸發方式

1. 開啟警報頁面（`l4_alert.html`，部署後用網址存取）
2. 輸入存取碼
3. 可選：填寫自訂訊息
4. 按下「發送緊急警報」

### 行為說明
- 從 Google Sheet 讀取啟用（C 欄打勾）的電話號碼
- 接聽後播放中文 TTS 語音警報，重複 3 次
- 第 1 通未接 → 等 40 秒 → 自動重撥第 2 通
- 撥打完成後自動寫入觸發時間到 Sheet D 欄

### 成本
- 打台灣手機：$0.1985/分鐘（國際通話費率）
- 叫醒一次（2 通 × ~1 分鐘）≈ $0.40
- Trial 帳號有免費額度（$15.50），用完再儲值

---

## 新增警報對象

### Step 1：Twilio 驗證號碼（Trial 限制）
1. 登入 Twilio Console
2. 左側選單 → Phone Numbers → Verified Caller IDs
3. 點 Add a New Caller ID → 輸入對方手機號碼（格式：`+886912345678`）
4. 對方會收到驗證電話或簡訊，完成驗證

> Trial 帳號最多 5 個 verified 號碼。升級 Pay as you go 後無此限制。

### Step 2：加入 Google Sheet
在 [L4 號碼管理 Sheet](https://docs.google.com/spreadsheets/d/1V2HaUreQjujhkOWdDmDsw7rj2QTL9mPc5lBSg2xL2hQ/) 的工作表「L4_Twillo_Trading_Alert」新增一列：
- A 欄：姓名
- B 欄：電話號碼（格式：`+886912345678`）
- C 欄：打勾啟用

### Step 3：請對方設定手機
> **重要**：iOS 靜音模式會覆蓋一切，白名單也擋不住。必須用勿擾取代靜音。

1. 把 Twilio 號碼 `+16623761837` 存為聯絡人，加入勿擾白名單
2. 設定靜音/勿擾自動聯動（勿擾 ON → 靜音自動 OFF，反之亦然）
3. 詳細步驟見 `shortcut_setup_guide.md` 的「手機端設定」

---

## 架構總覽

| 類型 | 等級 | 管道 | 成本 | 穿透勿擾 | 狀態 |
|------|------|------|------|----------|------|
| 被動通知 | **L2** | **Email（GAS）** | **免費** | **否** | **已完成** |
| 主動叫醒 | **L1** | **Telegram 推播（GAS）** | **免費** | **否** | **已完成** |
| 主動叫醒 | **L4** | **Twilio 自動電話** | **~$0.20/分（台灣手機）** | **是** | **已完成** |
| 主動叫醒 | L5 | Tapo P105 物理警鈴 | 已有硬體 | 物理聲音 | 待找水電工 |

**主動叫醒路徑**：L1（Telegram）→ L4（電話）→ L5（物理警鈴），有人回應即停止升級。

**被動通知**：L2 Email 獨立運作，偵測到異常就寄信留紀錄，不需要人回覆。

---

## 檔案結構

```
trading_alert_system/
├── .env                              # Twilio 憑證（備份用）
├── README.md                         # 本文件
├── l1_telegram_alert.gs              # L1 BTC 價格警報（GAS，Telegram 群組通知）
├── l2_btc_price_alert.gs             # L2 BTC 價格警報（GAS，被動 Email 通知）
├── l4_twilio_call_v3.gs              # L4 自動電話（GAS API 端點）
├── l4_alert.html                     # L4 前端頁面（獨立靜態 HTML）
├── shortcut_setup_guide.md           # iOS 捷徑設定教學（給團隊成員）
└── iphone_setup_guide.md             # iPhone 端設定指南（勿擾/靜音/白名單）
```

---

## Twilio 帳號資訊

- **方案**：Trial（免費，最多 5 個 verified 號碼）
- **Twilio 號碼**：+1 (662) 376-1837
- **Console**：登入 twilio.com 管理
- **憑證位置**：`.env`（Account SID + Auth Token）

---

## 待辦

### 近期（下次開工）
- [x] 團隊成員都能用的前端介面（iOS 捷徑，見 shortcut_setup_guide.md）
- [x] 檔名重構：區分 L1 / L2 / L3 / L4 / L5（如 `l4_twilio_call.py`）
- [x] 測試 iOS 勿擾模式下的接電話行為 → 結論：靜音 ON + 勿擾 ON + Twilio 號碼加白名單 = 來電有聲音
- [x] 製作手動啟用撥電話功能的操作流程（見 shortcut_setup_guide.md）

### 近期
- [ ] iOS 勿擾模式分白天/晚上設定（白天：所有來電響鈴；晚上：只有白名單響鈴）
- [x] L4 Web App 多帳號問題：改用 GAS API + 靜態 HTML 前端（v3.0.0）
- [x] L4 前端部署：https://billiscooking.github.io/trading-alert/l4_alert.html
- [ ] Twilio 驗證其他號碼（hui、Bill Sub）：短時間內驗證失敗，之後重試或升級 Pay as you go

### 後續
- [ ] 自動升級串接：L1 無回應 → 自動觸發 L4
- [ ] L5 找水電工裝 AC 110V 有線蜂鳴器 → 接 Tapo P105

---
title: "n8n｜Teams 會議逐字稿摘要自動化（Graph + Notion + Email）"
date: 2026-03-13 07:45:26
categories:
- 自動化
- 實作
tags:
- n8n
- Microsoft-Teams
- Microsoft-Graph
- OneDrive
- Transcript
- Notion
- Outlook
---


# n8n｜Teams 會議逐字稿摘要自動化（Graph + Notion + Email）

<!-- more -->

> 背景：新版 Microsoft Teams 的逐字稿常「附著在錄影檔頁面」，UI 可以手動下載，但要自動化就得**先找到錄影檔 → 對應到會議 → 再去 Graph 取 transcript**。這篇把整條 n8n pipeline 拆開講清楚，讓你可以照著重做。

## 你要達成什麼
- 週期性掃描 OneDrive/SharePoint 的 `Recordings` 錄影資料夾
- 找到最新的錄影檔，從檔案 metadata 的 `iCalUid` 回推到日曆事件
- 由日曆事件的 `onlineMeeting.joinUrl` 查到 `onlineMeetingId`
- 取得該 meeting 的 transcripts 清單 → 挑最新一筆 → 下載 `.vtt`
- 丟給 LLM 生成會議記錄（保留原順序、移除連結）
- 寫入 Notion database（避免重複寫入）
- 寄出 Email 給與會者（可額外抄送固定收件人）

---

## 先備條件（不然你會卡在「API 回空陣列」）

### A) Azure AD App（Client Credentials）
你的流程使用 `grant_type=client_credentials`，意味著：
- 你是用「應用程式權限（Application permissions）」去打 Graph
- 需要管理員同意（Admin consent）

**請把這些敏感值一律放在 n8n Credentials / Environment variables，不要硬寫在流程裡：**
- `TENANT_ID`（例：`<TENANT_ID>`）
- `CLIENT_ID`（例：`<CLIENT_ID>`）
- `CLIENT_SECRET`（例：`<CLIENT_SECRET>`）

### B) Microsoft Graph 權限（概念清單）
實際需要的 permission 取決於你用的 endpoint / tenant policy，但最低會碰到：
- 讀日曆：`Calendars.Read`（或對應的 application permission）
- 讀 online meetings：`OnlineMeetings.Read.All`
- 讀 transcripts：`OnlineMeetingTranscript.Read.All`（或等價權限）
- 讀錄影檔：`Files.Read.All` / `Sites.Read.All`（視錄影檔落點）

> 你若遇到 transcripts 清單 `value=[]`：通常是**權限不足**或是該 meeting 根本沒有 transcript。

### C) 你需要知道三個 ID（會遮蔽範例）
- `USER_ID`：你要查的使用者（流程中用 `/users/{id}`）
- `DRIVE_ID`：錄影檔所在的 drive（OneDrive 或 SharePoint drive）
- `Recordings` 路徑是否一致（你的流程用 `root:/Recordings:/children`）

---

## 全流程總覽（用人話翻譯）

### ASCII 流程圖（End-to-end）

```text
                (Schedule Trigger / every N hours)
                             |
                             v
+--------------------+   +---------------------------+
|  Get access token  |-->| Graph: OAuth2 token (AAD) |
+--------------------+   +---------------------------+
            |
            |  Bearer <ACCESS_TOKEN>
            v
      +-----+----------------------------------------------+
      |                 PARALLEL BRANCHES                  |
      +----------------------+-----------------------------+
                             |
          +------------------+------------------+
          |                                     |
          v                                     v
+---------------------------+       +-------------------------------+
| A) List Recordings videos |       | B) List calendar events (7d)  |
| GET /drives/<DRIVE_ID>/   |       | GET /users/<USER_ID>/         |
|   root:/Recordings:/...   |       |   calendarView?...            |
+---------------------------+       +-------------------------------+
          |                                     |
          v                                     v
+---------------------------+       +-------------------------------+
| Sort by createdDateTime   |       | SplitOut events[]             |
| desc + Limit(1) latest    |       | (each has iCalUId, joinUrl)   |
+---------------------------+       +-------------------------------+
          |                                     |
          +------------------+------------------+
                             v
                    +-------------------+
                    | Merge (video+events)
                    +-------------------+
                             |
                             v
                    +-------------------+
                    | Match by iCalUid  |
                    | video.source.iCalUid
                    | == event.iCalUId  |
                    +-------------------+
                             |
                             v
                +-------------------------------+
                | If meeting ended? (end < now) |
                +-------------------------------+
                             |
                             v
                 +------------------------------+
                 | Build onlineMeetings URL     |
                 | filter joinWebUrl == joinUrl |
                 +------------------------------+
                             |
                             v
+---------------------------+       +-------------------------------+
| Get onlineMeetingId       |       | Get attendees (optional path) |
| GET /users/<USER_ID>/     |       | from meeting participants     |
|   onlineMeetings?$filter= |       +-------------------------------+
+---------------------------+
             |
             v
+---------------------------+
| List transcripts           |
| GET /users/<USER_ID>/     |
|  onlineMeetings/<ID>/     |
|  transcripts              |
+---------------------------+
             |
             v
+---------------------------+
| Pick latest transcript     |
| sort createdDateTime desc  |
+---------------------------+
             |
             v
+---------------------------+
| Download VTT               |
| GET <transcriptContentUrl> |
|   ?$format=text/vtt        |
+---------------------------+
             |
             v
+---------------------------+
| LLM summary (meeting note) |
| (keep order, remove links) |
+---------------------------+
             |
             v
+-------------------------------+
| Notion dedup by meetingId      |
| Query DB: meetingId == <...>   |
+-------------------------------+
             |
     +-------+--------+
     | exists?        | not exists
     | (stop)         v
     |         +------------------+
     |         | Create Notion page|
     |         +------------------+
     |                   |
     |                   v
     |         +---------------------------+
     |         | Append blocks (chunk 1900)|
     |         +---------------------------+
     |                   |
     +-------------------+
             |
             v
+---------------------------+
| Send Email (Outlook)      |
| to attendees (+ cc fixed) |
+---------------------------+
```


1. 排程觸發（每小時/每幾小時跑一次）
2. 取 Graph access token
3. 一路並行做兩件事：
   - A 線：列出錄影檔資料夾下的影片 → 取最新一支
   - B 線：抓前 7 天行事曆事件（只抓會議）
4. 用 `iCalUid` 把「最新錄影檔」對到「對應的日曆會議事件」
5. 由該事件的 `joinUrl` 查 `onlineMeetingId`
6. 用 meetingId 拉 transcripts → 選最新一筆 → 下載 VTT
7. 交給 LLM 產摘要
8. 寫 Notion（先查是否已存在 meetingId，避免重複）
9. 寄 Email

---

## n8n 節點逐一拆解（含遮蔽參數與可重現範例）

以下「節點名稱」對應你提供的 workflow。

### 1) Schedule Trigger：定時執行
**用途**：每隔 N 小時跑一次。

建議：
- 初期 debugging：先改成手動 trigger 或拉長 interval，避免一直打 Graph 被 rate limit。

---

### 2) 取得 access token（HTTP Request / POST）
**用途**：用 Client Credentials 取得 Graph token。

**原流程重點**：
- URL：`https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`
- Body（form-urlencoded）：
  - `grant_type=client_credentials`
  - `client_id=<CLIENT_ID>`
  - `client_secret=<CLIENT_SECRET>`
  - `scope=https://graph.microsoft.com/.default`

**遮蔽範例**：
```http
POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=<CLIENT_ID>&
client_secret=<CLIENT_SECRET>&
scope=https%3A%2F%2Fgraph.microsoft.com%2F.default
```

輸出會有：
- `access_token`
- `expires_in`

後續所有 Graph 呼叫都用：
`Authorization: Bearer <access_token>`

---

### 3) driveid + folderid（Set）
**用途**：把 driveId / folderId 變成變數，讓後面好引用。

注意：你這版流程其實只用到 `driveId`（列 `Recordings` 走路徑），`folderId` 未被引用；如果你要改成用 folderId 指向任意資料夾，可以換成：
- `GET /drives/{driveId}/items/{folderId}/children`

**遮蔽**：
- `driveId = <DRIVE_ID>`
- `folderId = <FOLDER_ID>`

---

### 4) 取得所有影片資訊（HTTP Request / GET）
**用途**：列出 `Recordings` 底下所有錄影檔。

Endpoint（你目前用路徑法）：
```text
GET https://graph.microsoft.com/v1.0/drives/<DRIVE_ID>/root:/Recordings:/children
Authorization: Bearer <access_token>
```

成功時會回：
- `value: [ { id, name, createdDateTime, ... , source: { iCalUid } } ]`

> 關鍵：你後面要用的是 `source.iCalUid`。如果你拿到的檔案 metadata 沒有這個欄位，代表你抓到的不是 Teams 錄影檔（或 Graph 回應欄位不一致），需要改策略。

---

### 5) 取得影片陣列（Set）
**用途**：把上一個節點的 `value` 存到固定欄位，方便 split。

做法：
- `value = {{$json.value}}`

---

### 6) 拆分成影片檔列表（Split Out）
**用途**：把 `value[]` 拆成一筆一筆 item，讓你能排序與 limit。

---

### 7) 依新到舊排列（Sort）
**用途**：依 `createdDateTime desc` 排序。

---

### 8) 取最新影片資訊（Limit）
**用途**：只留下最新的一筆錄影檔。

建議（強化版）：
- limit 取 3～5 筆，避免最新一筆不是你要的 meeting（例如測試錄影）。

---

### 9) 找出日曆中前一周所有會議（HTTP Request / GET）
**用途**：抓過去 7 天的 calendar view，用來跟錄影檔做關聯。

Endpoint（遮蔽 USER_ID）：
```text
GET https://graph.microsoft.com/v1.0/users/<USER_ID>/calendarView?
  startDateTime=<ISO-START>&
  endDateTime=<ISO-END>
Authorization: Bearer <access_token>
```

你用的時間窗是：
- start：`now - 7 days`
- end：`now`

---

### 10) 取的日曆會議陣列（Set）→ 11) 切分日曆會議陣列（Split Out）
**用途**：同影片那段，把 `value[]` 拆成逐筆 event。

Graph event item 常見欄位：
- `iCalUId`
- `subject`
- `start.dateTime` / `end.dateTime`
- `onlineMeeting.joinUrl`

---

### 12) Merge（Merge）
**用途**：把「最新影片」與「一堆日曆 events」合流，交給下一步 code 節點做比對。

---

### 13) 比對會議資訊（Code）
**用途**：用 iCalUid 對應會議。

你的邏輯：
- events：挑有 `iCalUId` 的 item
- latestVideo：找有 `source.iCalUid` 的 item
- 若兩邊 iCalUid（忽略大小寫）相同 → 回傳 matched event

核心片段（概念）：
```js
const videoICalUid = latestVideo.json.source.iCalUid;
const matched = events.find(e => e.json.iCalUId?.toLowerCase() === videoICalUid?.toLowerCase());
```

**這一步是整條 pipeline 的靈魂**：它把「檔案系統世界（OneDrive 的錄影）」與「會議世界（Calendar/OnlineMeeting）」串起來。

---

### 14) If1（If）
**用途**：確認會議已結束才往下。

你這裡用 `end.dateTime + 8 hours` 與一個固定時間字串比較。

建議改法（更穩）：
- 用「現在時間」比較，而不是硬塞一個固定時間。
- n8n 表達式範例：
  - left：`DateTime.fromISO($json.end.dateTime).plus({ hours: 8 })`
  - right：`DateTime.now()`

避免：
- 流程跑久了，固定時間會失效。

---

### 15) 製作 URL（Code）
**用途**：用 joinUrl 反查 onlineMeeting。

Graph 這段的技巧是：
- 用 `$filter=joinWebUrl eq '<joinUrl>'`
- joinUrl 先 encode

遮蔽版輸出：
```js
const joinUrl = $json.onlineMeeting.joinUrl;
const encodedFilter = encodeURIComponent(`joinWebUrl eq '${joinUrl}'`);
const url = `https://graph.microsoft.com/v1.0/users/<USER_ID>/onlineMeetings?$filter=${encodedFilter}`;
return [{ json: { url, joinUrl } }];
```

---

### 16) 取得該會議 meetingId（HTTP Request / GET）
**用途**：打上一步產生的 URL，取得 `onlineMeeting` 物件，從 `value[0].id` 取出 meetingId。

注意：
- 如果 `value` 不是 1 筆，代表 joinUrl 可能重複或 filter 條件不夠精準。
- 如果 `value=[]`，代表你拿到的 event 沒對上 onlineMeeting（或權限不足）。

---

### 17) 取得逐字稿資訊（HTTP Request / GET）
**用途**：列出該 meeting 的 transcripts。

Endpoint（遮蔽）：
```text
GET https://graph.microsoft.com/v1.0/users/<USER_ID>/onlineMeetings/<MEETING_ID>/transcripts
Authorization: Bearer <access_token>
```

回傳：
- `value: [ { id, createdDateTime, transcriptContentUrl, meetingId, ... } ]`

---

### 18) 取得最新逐字稿資訊（Code）
**用途**：
- 確保 `value` 不為空
- 依 `createdDateTime` 排序
- 取最新 transcript 的 `id` 與 `transcriptContentUrl`
- 額外回傳 `debug_top5_dates` 幫你在 n8n UI 快速檢查

你這段已經寫得很工程化了（加 debug 非常實用）。

---

### 19) Get many database pages（Notion：查重）
**用途**：用 `meetingId` 查 Notion database，避免同一場會議寫入多次。

**遮蔽建議**：
- `databaseId = <NOTION_DATABASE_ID>`
- filter：`meetingId == <meetingId>`

---

### 20) If（Notion 結果判斷）
**用途**：如果查不到既有 page（`id` 不存在），才繼續建立新頁面。

---

### 21) 取得逐字稿全文（HTTP Request / GET）
**用途**：下載 transcript 內容（VTT）。

你用：
```text
GET <transcriptContentUrl>?$format=text/vtt
Authorization: Bearer <access_token>
```

---

### 22) 會議摘要（Anthropic / LLM）
**用途**：把逐字稿整理成「會議記錄」，且要求：
- 按順序
- 不重新編排
- 移除引用連結

安全建議：
- 逐字稿可能包含個資/商業資訊：建議加上資料保護與保存期限策略。

---

### 23) 確認語會人員資訊 → Split Out → 取得語彙人士 email → 收件人整理
**用途**：從 meeting participants 抽出 attendee emails，並手動追加固定收件人。

你目前的 code 會：
- collect `attenance`（建議拼字改成 `attendance`/`attendeeEmail` 以免未來維護踩雷）
- filter 空值
- `validEmails.push('<YOUR_COPY_EMAIL>')`

遮蔽範例：
```js
validEmails.push('<YOUR_COPY_EMAIL>');
```

---

### 24) 內容整理（Code：切 Notion blocks）
**用途**：
- 從 LLM 節點抓 `content[0].text`（針對 Claude output 結構）
- 從 transcript 節點抓全文
- 按 Notion block 長度限制（你設 chunkSize=1900）切段
- 產出 `notionBlocks`

這步的價值：
- Notion API 對 rich_text 長度有限制，你切段是必要工。

---

### 25) Create a database page（Notion：建立頁面）
**用途**：在 Notion database 建立一個會議頁。

你寫入的 properties：
- 日期（加 8 小時，Asia/Taipei）
- meetingId
- 會議名稱
- 與會者（目前是空字串 `=`，可改成 join emails）

---

### 26) HTTP Request（Notion：追加 children blocks）
**用途**：因為 Notion 建頁與寫內容是兩件事，所以你用 PATCH 把 blocks 塞進 page。

Endpoint：
```text
PATCH https://api.notion.com/v1/blocks/<PAGE_ID>/children
Notion-Version: 2022-06-28
```

Body：
- `children: <內容整理>.notionBlocks`

---

### 27) Send a message（Outlook：寄信）
**用途**：
- to：`收件人整理.emails`
- subject：`【自動發送】<會議主旨> - 會議記錄`
- body：LLM 摘要

遮蔽建議：
- 收件人 domain/個人信箱一律用 `<user@example.com>`

---

## 常見踩雷與排查（你會真的遇到）
1. **錄影檔找得到，但 `source.iCalUid` 沒有**
   - 你可能抓到的是不同來源的影片，或路徑不是 Teams Recordings。
2. **onlineMeetings filter 回空**
   - joinUrl 不同、event 沒 onlineMeeting、或權限不足。
3. **transcripts `value=[]`**
   - 沒開逐字稿、會議政策禁用、或缺 transcript 權限。
4. **Notion block 寫入失敗**
   - 多半是單段文字太長、rich_text 格式不符、或 Notion-Version 不一致。
5. **重複寫入**
   - 查重請以 `meetingId` 為唯一鍵（你已這樣做，正確）。

---

## 我會怎麼再加強這條流程（可選）
- 最新錄影檔不可靠：改成「最近 N 支」逐一比對 calendar，再挑最合理的一場。
- 產出的摘要做版本控管：把 raw VTT 也存到附件或 S3（依合規需求）。
- 收件人白名單/黑名單：避免寄給外部訪客。
- 把 If1 的固定時間改成 `now()` 比較，避免流程時間炸裂。



---

## 附錄：Graph 權限「最小集合」與 Admin Consent 檢查清單

> 你以為你卡在流程，其實你先死在權限。
>
> 這條自動化用的是 `client_credentials`（Application permissions），所以**必須做 admin consent**。另外 Teams/OnlineMeeting/Transcript 的權限在不同租戶政策下可能更嚴格，下面提供「能跑通」的最小集合與檢查步驟。

### A) 建議的 Microsoft Graph Application permissions（最小可跑通集合）

> 下面是「概念上」最小集合；實際名稱與可用性會因你租戶是否開放、Graph 版本、以及 Teams policy 而異。原則是：
> - 讀 OneDrive/SharePoint 錄影 → Files/Sites
> - 讀 Calendar → Calendars
> - 由 joinUrl 反查 onlineMeeting + transcripts → OnlineMeetings / Transcripts

1. **讀錄影檔（Recordings）**
   - `Files.Read.All`（Application）
   - 若錄影在 SharePoint site：可能需要 `Sites.Read.All`

2. **讀行事曆 events（calendarView）**
   - `Calendars.Read`（Application）或 `Calendars.Read.All`

3. **讀 Online Meetings（用 joinWebUrl filter 反查 meetingId）**
   - `OnlineMeetings.Read.All`

4. **讀 Meeting transcripts**
   - `OnlineMeetingTranscript.Read.All`（若租戶支援/已開放）

> 如果你希望「一開始就別卡」：可以先用較寬鬆的讀取權限把流程跑通，確認 endpoint 沒問題後再往回收斂。

### B) Admin consent 檢查清單（你應該逐項確認）
1. Azure Portal → App registrations → 你的 App
2. API permissions：確認上述 permissions 都已加入（Application，不是 Delegated）
3. 點 **Grant admin consent**（會顯示 Granted）
4. Credentials：client secret 未過期（建議用 `CLIENT_SECRET=<SECRET>` 放 n8n credential，不要硬寫在 workflow）

### C) 典型錯誤與「對應的權限/設定」
- `401 Unauthorized` / `InvalidAuthenticationToken`
  - token 失效、scope 不對、secret 過期
- `403 Forbidden`
  - 幾乎都是權限不足或租戶 policy 禁止
- transcripts `value=[]`
  - 逐字稿未啟用 / policy 禁用 / meeting 沒產生 transcript / transcript 權限缺失

### D) 建議的驗證順序（縮短排查時間）
1. 先驗證 token：`GET https://graph.microsoft.com/v1.0/organization`（能回資料表示 token OK）
2. 再驗證 recordings：列出 `Recordings` children
3. 再驗證 calendar：`calendarView` 是否拿到 `iCalUId` 與 `onlineMeeting.joinUrl`
4. 再驗證 onlineMeetings filter：joinUrl → meetingId 是否回 `value[0].id`
5. 最後驗證 transcripts：meetingId → transcripts list → contentUrl 下載 vtt
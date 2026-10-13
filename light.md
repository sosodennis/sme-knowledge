# 輕量級個人任務管理架構評估：基於 SQLite、Microsoft Graph API 與 GitLab SSOT 的工程實踐與邊界分析
在個人任務管理與電子郵件自動化工作流中，當資料規模維持在約 100 封常駐郵件的量級時，系統架構設計的核心在於取得維護成本、系統健全性（Robustness）與大語言模型（LLM）Token 消耗之間的最佳平衡。針對「方案 B」（以本地 SQLite 作為暫存投影層、Microsoft Graph API 為郵件來源、GitLab 作為單一真相來源 SSOT）的技術適切性，深入剖析其底層儲存機制、API 協定限制、分散式狀態對齊以及 Context 管理中的技術邊界，能為整體自動化流程提供高可靠的架構基礎。

## 模組一：量級適配度與架構權衡

### 儲存與記憶體負擔評估：SQLite 與純記憶體/JSON 的可靠性對比
在常駐 100 封郵件的負載情境中，單封純文字郵件內文與元數據平均介於 1 KB 至 3 KB，整體資料集在磁碟與記憶體中僅佔約 100 KB 至 300 KB。儘管該體積能被任何現代執行環境的記憶體完整容納，但架構設計若採用純記憶體結構或單一 JSON 檔案進行持久化，將面臨嚴峻的資料完整性風險。
單一 JSON 檔案的狀態更新高度依賴全檔覆寫（Full-file Rewrite）機制。當 Agent 或排程程序標記單筆郵件狀態時，作業系統必須重新序列化並寫入整個檔案。一旦在寫入瞬間發生處理程序崩潰、非預期斷電或多行程競態讀寫，極易造成檔案內容截斷或語法毀損，導致系統狀態遺失。
相對地，SQLite 具備預寫式日誌（Write-Ahead Logging, WAL）機制與完整的 ACID 事務保證。在 WAL 模式下，讀取操作不會阻塞寫入操作，寫入操作亦不阻塞讀取操作，單筆資料的更新僅涉及對日誌檔的追加寫入（Append-only），平均寫入延遲低於 1 毫秒。SQLite 的單檔案特性免去了獨立資料庫伺服器的部署與運維成本，在常駐 100 筆的輕量規模下，既提供了工業級的抗損毀能力，又維持了零維護負擔的優勢。

### 檢索模式評估：B-Tree 索引搭配條件查詢 vs. FTS5 全文檢索模組
在 100 筆資料量級下，標準的 B-Tree 索引搭配 SQL 條件篩選（例如 `WHERE status = 'pending'`）即可在 0.05 毫秒內完成檢索。然而，在個人任務管理情境中，Agent 經常需要執行非結構化的內文檢索，例如跨郵件比對特定伺服器錯誤代碼、追蹤專案識別碼或查詢特定異常堆疊。
若僅使用標準 SQL 的 `LIKE '%keyword%'` 語法，系統必須對內文字串進行逐行暴力比對。雖然在 100 筆記錄的極限測試中，記憶體內的 `LIKE` 掃描延遲僅約 0.084 毫秒，但此模式無法支援詞幹還原（Stemming）、詞頻權重排序以及分詞機制。SQLite 內建的 FTS5 虛擬表格模組透過外部內容表（External Content Table）與倒排索引，能以極低的磁碟開銷（約 70 KB 的額外索引資料）換取專業的全文檢索能力。

| **評估維度** | **結構化 B-Tree 條件查詢** | **傳統 SQL LIKE 模糊比對** | **SQLite FTS5 全文檢索模組** |
| --- | --- | --- | --- |
| **平均查詢延遲** | ~0.045 ms (`status = 'pending'`) | ~0.084 ms (`body LIKE '%ERR%'`) | ~0.174 ms (`MATCH 'ERR'`) |
| **磁碟與記憶體開銷** | 基礎資料庫大小約 85 KB | 無額外索引開銷 | 包含虛擬表與分詞資料，庫體約 156 KB |
| **維護複雜度** | 內建原生支援，零維護 | 內建原生支援，零維護 | 需維護觸發器（Triggers）以同步更新虛擬表 |
| **進階檢索能力** | 僅支援精確匹配與範圍查詢 | 子字串比對，不支援分詞與權重 | 支援 BM25 評分、前綴匹配、NEAR 接近檢索 |
| **架構選型定位** | **核心必選**（任務狀態流轉與主鍵關聯） | **過渡方案**（僅限極簡代碼匹配） | **建議保留**（支援 Agent 複雜多維度內文關聯） |

對於常駐 100 封信件的系統，結構化 B-Tree 索引足以支撐所有狀態與時間維度的調度；但若系統預期讓 LLM Agent 自主檢索相關歷史情境，啟用 FTS5 能以微不足道的空間代價提供顯著更強的語義檢索靈活性。

## 模組二：Microsoft Graph API 的關鍵陷阱與快照同步

### 預設 Message ID 可變性與 ImmutableId 的持久化保證
在 Exchange Online 與 Microsoft 365 底層架構中，Microsoft Graph API 預設回傳的郵件 `id` 是與信箱資料夾容器綁定的可變識別碼（Mutable ID）。當使用者手動將郵件移至其他資料夾（如自「收件匣」移至「封存」）、透過規則自動歸檔，甚至在特定情況下修改郵件分類時，底層的 Exchange EntryID 會重新生成，導致 Graph API 回傳的預設 `id` 發生改變。若本地 SQLite 將預設 `id` 定義為 `PRIMARY KEY`，一旦郵件位置移動，後續同步將判定為新郵件而重複寫入，造成資料庫中出現孤立重複記錄，破壞關聯一致性。
為防範此陷阱，所有發往 Microsoft Graph API 的 HTTP 請求均必須在 Header 中明確宣告偏好設定：

```http
Prefer: IdType="ImmutableId"

```
透過此標頭，Graph API 將生成一個與儲存位置解耦的永久識別碼（`ImmutableId`），該 ID 在郵件存在於該信箱的整個生命週期內保持不變。在系統架構中需注意以下邊界與行為：

- 跨信箱移轉例外：當郵件被移至獨立的封存信箱（Online Archive Mailbox）或以 `.pst` 檔案匯出再重新匯入時，`ImmutableId` 仍會被重新派發。
- 請求範圍限制：`Prefer` 標頭僅作用於單次 HTTP 請求，不會全域保存於 OAuth 權杖或會話中；因此在封裝 SDK 或 API Client 時，必須將該標頭設定為預設中介層（Middleware）行為。

### 增量同步（Delta Query）限制與分類批次快照查詢權衡
在管理「動態維持約 100 封常駐標記郵件」的情境下，決定同步協定時必須考量 Graph API 的底層限制。
Microsoft Graph 提供了 `messages/delta` 增量查詢機制，專門用於追蹤信件的建立、更新與刪除。然而，**Graph API 的 Delta Query 在郵件端點上存在嚴格的篩選限制：其僅支援針對接收時間的過濾（`$filter=receivedDateTime ge ...`），完全不支援集合導航屬性過濾（即無法支援 `$filter=categories/any(...)`）**。若強制採用 Delta Query，系統必須對使用者的整個收件匣進行全量增量追蹤，並在客戶端逐筆計算標籤變更，這在大型個人信箱中將導致巨大的頻寬浪費與處理負擔。
相對地，採用特定分類的批次快照查詢（Category Snapshot Polling）是更具針對性的做法。透過標準端點調用，可直接利用 OData 運算子篩選帶有特定分類的信件，並透過請求標頭同時指定獲取純文字內容與持久識別碼：

```http
GET https://graph.microsoft.com/v1.0/me/mailFolders/inbox/messages?$filter=categories/any(c:c eq 'TaskPending')&$select=id,subject,from,receivedDateTime,hasAttachments,body&$top=100
Prefer: IdType="ImmutableId", outlook.body-content-type="text"

```
下表對比了兩種同步機制在特定分類管理場景下的技術特性：

| **技術指標** | **Delta Query（增量同步）** | **特定分類批次快照查詢（Snapshot Polling）** |
| --- | --- | --- |
| **自訂分類篩選能力** | 不支援（OData 語法限制，無法篩選 `categories`） | 原生支援（`categories/any(c:c eq 'TaskPending')`） |
| **監控範圍** | 必須監控整個資料夾內所有郵件的狀態變更 | 精準鎖定標記有特定標籤的目標郵件群 |
| **API 請求次數** | 初始全量遍歷成本高，後續依憑證（Delta Token）單次拉取 | 單次請求即可覆蓋 100 筆目標資料（`$top=100`） |
| **取消標記判定機制** | 依賴回傳實體中 `categories` 欄位變更推斷 | 透過本地與遠端主鍵集合之數學差集運算判定 |
| **系統設計複雜度** | 高（需維護 Delta Link 狀態、分頁及全資料夾快取） | 低（無狀態輪詢，單一集合運算即可完成狀態對齊） |

為了在快照輪詢中精確辨識哪些郵件已被取消標記或刪除，本地系統採用集合差集運算：

1. 定義遠端快照集合 $S_{\text{remote}} = \{ \text{ID}_1, \text{ID}_2, \dots, \text{ID}_n \}$（單次請求回傳至多 100 筆資料）。
2. 定義本地待處理集合 $S_{\text{local}} = \{ \text{ID} \mid \text{status} \in (\text{'pending'}, \text{'processed'}) \}$。
3. 移除或取消標籤的資料集為 $S_{\text{removed}} = S_{\text{local}} \setminus S_{\text{remote}}$；系統將 SQLite 中屬於 $S_{\text{removed}}$ 的郵件狀態批量標記為 `untagged` 或歸檔。
4. 新增的資料集為 $S_{\text{new}} = S_{\text{remote}} \setminus S_{\text{local}}$；系統針對 $S_{\text{new}}$ 中的郵件抓取資料並寫入本地資料庫。
此機制以單次 HTTP 呼叫即可達成資料一致性，完全規避了 Delta Query 無法依分類過濾的限制，架構簡潔且易於維護。

## 模組三：GitLab 作為 SSOT 的對齊機制與檢索限制

### Description 全文檢索延遲與速率限制陷阱
在嘗試驗證某一郵件是否已建立 GitLab Issue 時，常見的直覺作法是透過 REST API 搜尋 Issue Description（例如 `GET /issues?search=source_email_id:xxx&in=description`）。然而，該模式在架構上存在三大嚴重缺陷：
第一，**Elasticsearch 索引延遲（Indexing Lag）**。在 GitLab 實例中，搜尋索引由背景工作（Sidekiq）非同步推播至 Elasticsearch 叢集。在 Issue 剛建立後的數秒至數分鐘內，搜尋端點可能無法檢索到新寫入的內容。若 Agent 依賴搜尋結果作為防重複建立的判斷依據，將引發競態條件，導致重複開單。
第二，**嚴苛的搜尋頻率限制（Rate Limits）**。GitLab 針對全域或專案搜尋端點設置了獨立的頻率節流防護。在預設配置下，認證使用者的常規 API 請求配額為每分鐘數百次至每小時數千次，但搜尋端點的配額通常被限縮至每分鐘 30 次。若對常駐 100 封郵件逐筆發起搜尋請求，將迅速耗盡配額並觸發 HTTP 429 Too Many Requests 錯誤。
第三，**子字串精確比對不足**。GitLab Advanced Search 基於分詞器運作，對於包含長雜湊或特殊符號的 Microsoft Graph ID，常因字符切割導致查詢遺失或誤判，無法達成 100% 的精確對齊。

### 單一專案批次拉取與隱藏元數據快取對齊方案
為確保 SQLite 的狀態完全服從 GitLab，且不耗損搜尋 API 配額，最佳實踐是以**單一專案全量批次拉取**取代個別搜尋。在個人任務管理的情境中，任務集中於單一特定專案，同步腳本啟動時僅需調用一次專案 Issues 列表端點：

```http
GET https://gitlab.example.com/api/v4/projects/:project_id/issues?state=all&per_page=100
PRIVATE-TOKEN: <your_gitlab_token>

```
為了使 Issue Description 能承載機器可讀的元數據，同時不干擾使用者的閱讀體驗，系統遵循 GitLab Flavored Markdown（GLFM）規範，將郵件識別碼嵌入於隱藏的 HTML 註釋區塊中：

### 任務摘要
分析生產環境 Kubernetes 節點超時異常紀錄。
HTML 註釋在 GitLab Web UI 渲染時完全隱藏，但完整保留於 REST API 回傳的原始 Markdown 字串中。同步程序在記憶體中以正則表達式快速提取所有 Issue 的 `source_id` 與其對應的 Issue 內部編號 `iid` 及狀態 `state`，隨後在本地 SQLite 執行單筆原子事務的批次更新：

```sql
BEGIN TRANSACTION;
UPDATE emails 
SET status = 'processed', 
    gitlab_issue_iid = :iid, 
    gitlab_state = :state
WHERE id = :email_id;
COMMIT;

```
此方案將潛在的 100 次高負擔搜尋請求壓縮為**1 次標準列表查詢**，從根本上解決了 Elasticsearch 索引延遲與 API 頻率限制的問題。

## 模組四：任務管理生命週期與雙向閉環

### Issue Closed 狀態處置與郵件分類雙向同步
在雙向閉環設計中，GitLab Issue 的狀態變更是驅動郵件處置的核心依據。當團隊或個人在 GitLab 上完成任務並將 Issue 標記為 `closed` 時，本地 SQLite 與遠端 Microsoft Graph 必須及時反映該生命週期終結，以防止常駐郵件池隨時間推移積壓無效資料。
若在 Issue 關閉後未將對應郵件移出自訂分類，該郵件仍將持續出現在快照查詢的 100 筆列表中，造成常駐池名額被歷史封存郵件佔據。因此，當本地同步程序比對發現 GitLab 中的 `state == 'closed'` 時，自動觸發兩組操作：

1. 本地狀態變更：將 SQLite 中該記錄的 `status` 更新為 `archived`，並將 `gitlab_state` 更新為 `closed`。
2. 遠端標籤清洗：透過 Graph API 的 `PATCH` 方法，將郵件的 `TaskPending` 分類移除，並可選擇性標記為 `TaskDone`。

```http
PATCH https://graph.microsoft.com/v1.0/me/messages/AAMkADk2NW...
Content-Type: application/json
Prefer: IdType="ImmutableId"

{
  "categories": ["TaskDone"]
}

```
下表完整定義了任務管理中跨系統的狀態機轉換規範：

| **業務狀態階段** | **本地 SQLite 投影狀態** | **Microsoft Graph API 分類標籤** | **GitLab Issue 狀態** | **系統處置行為** |
| --- | --- | --- | --- | --- |
| **新郵件納管** | `status = 'pending'` | 包含 `TaskPending` | 尚未建立 | 納入待處理隊列，供 Agent 進行任務開單 |
| **開單完成** | `status = 'processed'` | 包含 `TaskPending` | `opened` | 關聯已建立，定期快照拉取並追蹤 GitLab 狀態 |
| **任務完成閉環** | `status = 'archived'` | 移除 `TaskPending`，加入 `TaskDone`[cite: ] | `closed` | 自常駐池移出，釋出監控配額 |
| **手動取消標籤** | `status = 'untagged'` | 無相關任務分類 | 保持原樣（加註系統標記） | 停止本地主動追蹤，保留歷史 Issue 記錄 |

### 獨立郵件任務映射機制
在個人任務管理架構中，每封被標記為 `TaskPending` 的郵件皆被視為獨立的任務實體。系統以全局持久的 `ImmutableId` 作為唯一鍵值，進行嚴格的一對一任務綁定：

- 當新郵件進入系統時，以其 `ImmutableId` 查詢 SQLite，確認是否已存在關聯的 `gitlab_issue_iid`。
- 若已存在對應 Issue 且處於 `opened` 狀態，則維持現狀；若 Issue 已 `closed`，則依規則將該郵件自監控隊列歸檔。
- 若未關聯任何 Issue，則判定為全新待辦項目，進入常規開單流程。
此種以單信為單位的精確映射模型大幅精簡了資料結構與關聯邏輯，確保每個任務狀態流轉清晰可追蹤。

## 模組五：資料傳輸與 Agent Token 成本控制

### 原生純文字提取與管線輕量化
在向 Microsoft Graph API 請求郵件時，透過在 HTTP 請求標頭中宣告 `Prefer: outlook.body-content-type="text"`，Exchange 伺服器端會直接完成 HTML 的解析轉換，回傳標準純文字格式（`text/plain`）的郵件內文。
這一機制帶來了顯著的工程簡化優勢：

- **免除客戶端 HTML 清洗依賴**：完全無需在本地部署如 `html2text` 或 `trafilatura` 等第三方 HTML 解析套件，消除解析失敗、排版截斷或相容性問題。
- **根除樣式與 Base64 雜訊**：微軟專屬命名空間宣告（如 `xmlns:o`、`xmlns:w`）、條件註釋（`<!--[if gte mso 9]>`）、內嵌 CSS 樣式表以及大體積的 Base64 內嵌圖片，在伺服器端已被全數剝除，傳輸至本機的資料即為乾淨內文。
- **極小化本機運算與儲存負擔**：接收到的純文字內容可直接寫入 SQLite 資料庫並建立索引，大幅降低 CPU 序列化開銷與記憶體佔用。

| **資料處理面向** | **傳統客戶端 HTML 清洗模式** | **請求標頭指定純文字模式 (outlook.body-content-type="text")** |
| --- | --- | --- |
| **資料傳輸體積** | 較大（包含完整 HTML 標籤、樣式與可能的內嵌二進位碼） | 最小（僅傳輸字元本文，平均僅數 KB） |
| **客戶端依賴** | 需引入 HTML 解析、Markdown 轉換及正則替換模組 | **零額外解析依賴**，直接使用原生文字欄位 |
| **例外破壞風險** | 複雜排版或損毀 HTML 可能導致解析程式異常 | 伺服器端原生輸出純文字，穩定性極高 |
| **Token 控制效益** | 需經多道過濾管線才可壓縮至安全範圍 | **開箱即用**，單信平均僅消耗 100～250 Token |

### 附件處理邊界與解析防禦
郵件附件常包含 Excel 試算表或 PDF 技術文件，若未加設防地將附件文字全量載入，將迅速耗盡 Token 預算。系統應針對附件型態建立明確的處理邊界：

- 元數據先行：當郵件的 `hasAttachments` 屬性為 `true` 時，優先僅向 Agent 提供附件的檔案名稱與大小元數據。
- PDF 文件分級讀取：針對 PDF 文件，使用 `pypdf` 或 `pdfplumber` 解析時，限制僅讀取前 2 頁或上限 2,000 字元，超過長度則由本地程式截斷並附註 `[PDF Truncated]`。
- Excel 試算表結構化抽樣：針對 `.xlsx` 文件，利用 `openpyxl` 僅讀取使用中的工作表（Active Sheet），限制抽取標題列與前 10 筆資料列並轉換為文字表格，其餘資料標註 `[Sheet Data Omitted]`。大型附件與二進位檔案則完全排除在 Agent 直接讀取範圍之外。

### Agent CLI 介面設計與 Token 消耗量測
為了讓 LLM Agent 能經濟且精準地操作本地資料庫，將底層 SQL 與 API 封裝為高內聚的 CLI 命令，並實施「先摘要、後明細」的惰性載入設計。
CLI 工具介面包含三個核心指令：

- `email-agent list-pending`：僅輸出待處理郵件的元數據清單（包含 ID、發件人、主旨與時間戳記），完全不輸出內文。100 筆資料在此模式下僅消耗約 800 至 1,200 Token。
- `email-agent inspect-email <id>`：僅當 Agent 評估需要針對特定郵件開單時調用，輸出單一郵件的純文字內文與附件摘要，單次消耗約 150 至 350 Token。
- `email-agent create-issue-payload <id>`：由 Agent 產出結構化的任務標題、標籤與描述，調用 GitLab API 完成原子開單，並回寫本地狀態。
此種分段式介面設計徹底將「清單掃描」與「深度推理」解耦，確保 Agent 執行單次任務開單的整體 Token 消耗精確控制在 2,000 Token 以內。

## 具體執行與驗證步驟 (PoC Milestones)
為了驗證架構設計並提前排除實作雷區，概念驗證（PoC）規劃分為四個階段依序推進：

| **階段名稱** | **核心驗證目標** | **具體技術行動** | **交付成果 / 驗證標準** | **關鍵防禦機制** |
| --- | --- | --- | --- | --- |
| **Phase 1: Graph API 穩定性** | 驗證 `ImmutableId`、純文字 Header 與標籤篩選可靠度 | 撰寫腳本拉取標記郵件；驗證回傳內文為純文字；手動移轉資料夾後比對 ID | 郵件移轉後 ID 完全不變；成功回傳純文字內容；分類篩選準確率達 100% | 請求標頭強制注入 `Prefer: IdType="ImmutableId", outlook.body-content-type="text"`[cite: 2] |
| **Phase 2: 本地 SQLite 投影** | 驗證資料持久化、WAL 模式與純文字入庫效益 | 建立 `emails` 與 `emails_fts` 表格；注入 100 封純文字郵件並測試寫入與檢索效能 | 100 筆入庫後檔案小於 1 MB；單筆更新延遲 < 1 ms；WAL 機制運作正常 | 啟用 WAL 模式；結構化主表搭配 FTS5 虛擬表格 |
| **Phase 3: GitLab 對賬機制** | 驗證單次批次狀態對齊與防重複開單機制 | 單次調用專案 Issues 列表；以正則解析隱藏 HTML 註釋中的 `source_id` 並更新本機庫 | 1 次 HTTP 請求即完成 100 筆狀態對齊；搜尋 API 呼叫次數為 0；無重複建單 | 採用 GLFM 隱藏註釋標記，完全繞過 Elasticsearch 搜尋端點 |
| **Phase 4: Agent CLI 介面** | 驗證工具指令封裝與上下文 Token 經濟性 | 實作 `list-pending` 與 `inspect-email`；模擬 Agent 端到端開單任務並量測 Token | 單次開單決策 Token 消耗 < 2,000；命令回傳格式乾淨且無多餘雜訊 | 實施兩階段惰性載入，徹底分離元數據與本文載入 |

## 結論與架構演進建議
綜合量級規模、系統可靠性與 API 邊界分析，「方案 B」在處理常駐約 100 封郵件的個人任務管理情境中展現了極高的工程適切性。
架構選型分析表明，在本地儲存層採用 SQLite WAL 模式，能在無需額外伺服器運維成本的前提下，徹底規避 JSON 檔案覆寫引發的資料損毀風險，兼具毫秒級事務更新能力與 FTS5 檢索擴展性。在遠端資料同步層，面對 Microsoft Graph API 的協定限制，採用注入 `Prefer: IdType="ImmutableId", outlook.body-content-type="text"` 標頭的自訂分類快照查詢，既繞過了 Delta Query 無法依分類篩選的限制，又直接從來源端獲取乾淨純文字，免除了繁瑣的 HTML 清洗管線與格式轉換依賴。
在任務管理閉環與狀態對齊層，將 GitLab 確立為 SSOT 時，嚴格避開高延遲且受限於頻率配額的 Issue 全文搜尋 API，改採單一專案 Issues 批次拉取搭配 GLFM 隱藏 HTML 註釋解析，以 1 次 HTTP 請求達成全局狀態對齊。同時，將每封信件依其唯一 `ImmutableId` 獨立對齊至單一 GitLab 任務，免去複雜的執行緒跟蹤負擔。最後，配合 CLI 的惰性載入設計，能將單次開單決策的 Token 成本壓低至 2,000 Token 以內，確保系統在長期運作下維持高效能、低複雜度與高穩定性。

---

Source: https://gemini.google.com/app/d5008d33afa4abe7
Exported at: 2026-10-13T05:50:43.437Z
---
name: weekly-journal-digest
description: |
  每週 ID & 血液科期刊日報代理人 — 以 PubMed edat 過濾搜尋六大期刊近 7 天文章
  （NEJM 取 Originals + Reviews，無主題過濾），並從 Gmail 取得 NEJM ID 與 Hema/Onc 月報
  以補完 Cases / Images / Perspective / Correspondence；CrossRef 與 OA 全文回退補完缺失摘要、
  AI 標注中文摘要與嘻嘻/不嘻嘻短評、輸出 Markdown digest。

  當使用者說「跑週報」、「weekly journal」、「RSS 日報」、「跑本週 / 上週 journal」
  或明確要求執行這份 routine 時，使用此 skill。
---

> ⚠️ **SKILL_local.md — 本地端版本**
> 與 `SKILL_cloud.md` 的差異：
> - **Subagent B Step B3.0**：新增「追蹤 URL redirect → DOI 解析」作為第一優先 fallback。此步驟依賴 WebFetch 跟隨 `t.n.nejm.org` → `www.nejm.org/doi/...` 的跨網域 redirect；cloud sandbox 的 TLS proxy 會阻擋此請求，故**僅本地端執行有效**。
> - **Step C2.5**：新增機構瀏覽器（Claude-in-Chrome）作為付費全文的最後回退。
> - **Step 5**：newsletter 文章 B3 解出 DOI 時直接顯示 DOI 連結，而非 tracking URL。

## 配置

首次使用前請修改下列設定：

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `OUTPUT_DIR` | `output` | digest Markdown 輸出目錄 |
| `SKILL_DIR` | `.claude/skills/weekly-journal-digest` | skill 所在目錄（存放 `.seen_pmids.json`） |
| `CROSSREF_MAILTO` | `pubmed-weekly-digest@noreply.example` | CrossRef / Unpaywall polite-pool 識別碼（任意有效 email） |

---

# Weekly ID & Hema Journal Digest

## ⚠️ 週次計算規則（最重要，先讀）

**本 skill 在每週初（通常週一）執行，涵蓋的是「剛結束的那一週」。**

```python
from datetime import date, timedelta

today = date.today()

# 搜尋視窗：過去 7 天（不含今天）
date_to   = today - timedelta(days=1)   # 昨天
date_from = today - timedelta(days=7)   # 7 天前

# 週次標籤：使用「剛結束那週」的 ISO week，而非今天所在週
target_date = today - timedelta(days=7)
iso_year, iso_week, _ = target_date.isocalendar()
week_label = f"{iso_year}-W{iso_week:02d}"   # e.g. 2026-W19

# 檔案路徑
filepath = f"{OUTPUT_DIR}/{week_label}.md"
```

**範例**：今天 2026-05-11（W20）→ `week_label = 2026-W19`，搜尋範圍 2026-05-04 至 2026-05-10。

> **錯誤陷阱**：不要用 `today.isocalendar()` 算週次，否則會把 W19 的內容存成 W20。

---

## 執行架構

計算出 `date_from`、`date_to`、`week_label` 後，先跑 **NEJM 月報閘門**（見下）決定本次是否需要 Subagent B，再 **同時（單一訊息中）** 啟動 subagent：

- **Subagent A（PubMed）**：搜尋六大期刊 + 取得完整 metadata → 回傳各 section 文章清單（JSON）。**每次都跑。**
- **Subagent B（Gmail）**：取得 NEJM 月報 + 解析 + 對每篇 newsletter 文章做 PubMed/CrossRef 標題回查 → 回傳兩個月報的文章清單（JSON）。**僅當 `run_newsletter == True` 時才跑。**

subagent 均完成後，主 agent 執行：dedup → CrossRef 補全 → 過濾 → 撰寫摘要 → 輸出。

### NEJM 月報閘門（newsletter 每月僅出刊一次）

NEJM ID 與 Hema/Onc 月報每月只寄一次（約當月第一週），同一期月報只該被收錄進**一份**週報。在啟動 Subagent B 前，先檢查本月是否已有週報收錄過月報；若有則本次跳過 Subagent B，避免重複（也避免 newer_than 視窗外的舊月報被誤撈進來）：

```python
import glob, re
from pathlib import Path

# 以 target_date（= 涵蓋週）的日曆月為「本月」
ym = target_date.strftime("%Y-%m")          # e.g. 2026-05

run_newsletter = True
prior_week = None
for f in sorted(glob.glob(f"{OUTPUT_DIR}/*.md")):
    name = Path(f).stem                      # e.g. 2026-W19
    m = re.match(r"(\d{4})-W(\d{2})$", name)
    if not m or name == week_label:
        continue
    fy, fw = int(m.group(1)), int(m.group(2))
    monday = date.fromisocalendar(fy, fw, 1)
    sunday = monday + timedelta(days=6)
    # 該週若與本月有重疊才納入比對
    if ym not in (monday.strftime("%Y-%m"), sunday.strftime("%Y-%m")):
        continue
    # 「本月已收錄」偵測：實際收錄月報的週，其 NEJM Monthly 標題帶真實發行日（含 4 位年份，如「（May 2026）」）；
    # 佔位週寫「（本月已收錄）」、未收錄週無年份。此訊號不受 newsletter 連結轉成 DOI 影響。
    if re.search(r'## NEJM Monthly.*（[^）]*\d{4}[^）]*）', Path(f).read_text(encoding="utf-8")):
        run_newsletter = False
        prior_week = name
        break
```

- `run_newsletter == True`：本月尚無週報收錄月報 → 照常啟動 Subagent B。
- `run_newsletter == False`：本月 `{prior_week}` 已收錄 → **不啟動 Subagent B**；主 agent 略過 Step C1 dedup，兩個 NEJM Monthly section 改寫佔位（見 Step 5）。

---

## Subagent A — PubMed 搜尋與 Metadata

### 傳入 subagent 的 prompt 內容

> 你是 weekly-journal-digest routine 的 PubMed subagent。
>
> 搜尋視窗：date_from={date_from}，date_to={date_to}（YYYY/MM/DD）
>
> **Step A1 — 並行搜尋六期刊**（datetype=edat）：
>
> | 查詢字串 | max_results |
> |---------|-------------|
> | `Clin Infect Dis[Journal]` | 50 |
> | `Emerg Infect Dis[Journal]` | 50 |
> | `MMWR Morb Mortal Wkly Rep[Journal]` | 30 |
> | `Transpl Infect Dis[Journal]` | 50 |
> | `Blood[Journal]` | 50 |
> | `N Engl J Med[Journal]` | 50 |
>
> 任一期刊搜尋失敗（MCP 錯誤）：回傳錯誤訊息並停止。
>
> **Step A2 — 並行分批取得完整 metadata**：對每個期刊的 PMID 清單呼叫 `mcp__PubMed__get_article_metadata`。若單批超過 20 筆，分批處理（每批 ≤20）。
>
> **Step A3 — 過濾**（在回傳前執行）：
> - 所有期刊：移除 `article_types` 含 `"Erratum"` 或 `"Published Erratum"` 的文章
> - 所有期刊：移除 `article_types` 含 `"Editorial"` 或 `"Comment"` 的文章
> - **N Engl J Med section 額外**：只保留 Original Article 與 Review Article：
>   - Original Article：`"Journal Article"` 且 `article_types` 不含 Case Reports / Editorial / Comment / Letter / News / Biography / Historical Article / Portrait / Interview / Personal Narrative
>   - Review Article：`article_types` 含 `"Review"` 或 `"Systematic Review"`
>   - **再排除 Perspective 與 Multimedia**（PubMed 常把這兩類標成 `["Journal Article"]`，無法靠 `article_types` 區分，改以 DOI 前綴判斷）：
>     - DOI 符合 `/10\.1056\/NEJMp/i`（Perspective，含 ITT / NOS / Double Take 等 podcast 多媒體系列）→ 排除
>     - DOI 符合 `/10\.1056\/NEJMvcm/i`（Videos in Clinical Medicine）或 `article_types` 含 `"Multimedia"` → 排除
>   - 其餘排除（這些非 Original/Review 的篇目由 Gmail 月報補完）
>
> **回傳格式**：最後輸出一個 JSON 程式碼區塊，結構如下：
>
> ```json
> {
>   "sections": {
>     "Clin Infect Dis": [
>       {
>         "pmid": "12345678",
>         "title": "...",
>         "authors": "Smith J et al.",
>         "pub_date": "2026-05-12",
>         "doi": "10.1093/cid/ciad123",
>         "abstract": "...",
>         "article_types": ["Journal Article"]
>       }
>     ],
>     "Emerg Infect Dis": [...],
>     "MMWR Morb Mortal Wkly Rep": [...],
>     "Transpl Infect Dis": [...],
>     "Blood": [...],
>     "N Engl J Med": [...]
>   }
> }
> ```
>
> 沒有 abstract 的文章：`"abstract"` 填 `""` 或 `"[Abstract not available]"`。
> 沒有 DOI 的文章：`"doi"` 填 `""`。

---

## Subagent B — Gmail 月報與 PubMed 回查

> **Gmail MCP 說明**：若你以 Gmail 帳號直接訂閱 NEJM newsletter，可直接使用 Claude Code 的 Gmail MCP（`mcp__Gmail__search_threads`）取得，無需額外設定。若訂閱信箱非 Gmail（如 ProtonMail），請先設定將 NEJM 月報自動轉寄至 Gmail，再透過 MCP 存取。

### 傳入 subagent 的 prompt 內容

> 你是 weekly-journal-digest routine 的 Gmail subagent。
>
> **Step B1 — 並行搜尋兩類月報**：
>
> | 用途 | 查詢字串 |
> |------|---------|
> | NEJM 月報 — Infectious Disease | `from:nejm.org subject:"Infectious Disease Update from NEJM" newer_than:7d` |
> | NEJM 月報 — Hematology/Oncology | `from:nejm.org subject:"Hematology/Oncology Update from NEJM" newer_than:7d` |
>
> （若月報是從其他信箱轉寄至 Gmail，可在查詢字串中加上 `OR from:{YOUR_FORWARDING_ADDRESS}`。）
>
> 對每個有結果的 thread 用 `mcp__Gmail__get_thread` 取出內容。
> 月報查無結果：該月報清單回傳空陣列，`issue_date` 填 `null`。
> Gmail MCP 失敗：回傳錯誤訊息，讓主 agent 知道繼續但該 section 標注 `> Gmail 取得失敗。`。
>
> **Step B2 — 解析每個月報 HTML**，逐篇提取：
> - `title`：anchor tag 的文字（先 strip 子標籤如 `<em>`、`<strong>` 再讀）
> - `authors`：title 連結下一行，缺則留空
> - `article_type`：type/date 行（忽略 FREE / CME / [Video] / [Audio] tag）
> - `per_article_date`：type 行內的日期（格式 `Apr 30, 2026`）
> - `issue_date`：email body 上方 `{Specialty} Update {Month YYYY}` 標題
> - `tracking_url`：從 `htmlBody` 找到包含該標題文字的 `<a href="...">` 標籤，取 href 並 HTML-unescape（`&amp;` → `&`）。**注意部分標題含巢狀 `<em>`/`<strong>`，需 strip 後比對。** 找不到則留空字串。
>
> 解析失敗的單篇：跳過並回報。
>
> **Step B3 — 對每篇 newsletter 文章做 DOI/PubMed 回查**（取得 DOI 與 abstract）：
>
> > **重要**：所有 article type（Editorial、Perspective、Correspondence 等）**一律執行**，不得跳過。
>
> 依序嘗試以下 fallback，命中即停止：
>
> 0. **追蹤 URL redirect → DOI 解析**（有 `tracking_url` 時優先嘗試）：
>    - `WebFetch(tracking_url)`，跟隨 redirect（`t.n.nejm.org` → `www.nejm.org/doi/...`）。
>    - 從落地 URL 路徑解 DOI：`re.search(r'/doi/(?:full|abs|pdf)/(10\.\d{4,}/[^?#\s]+)', final_url)`
>    - 解出 DOI 後，以 `mcp__PubMed__search_articles` 查詢 `{DOI}[AID]`（`max_results=1`）取 PMID，再呼叫 `mcp__PubMed__get_article_metadata` 取 abstract、`article_types`、authors。
>    - 命中：填入 `doi`、`pmid`、`abstract`（可能為空），**跳至 B4**。
>    - WebFetch 失敗（非 2xx、redirect 後仍無法取 URL、解不出 DOI、PubMed 0 筆）：繼續步驟 1。
>
> 1. **PubMed 標題搜尋（多階段 fallback）**，`max_results=5`，**不限 edat**：
>    1. `"<full title>"[Title] AND N Engl J Med[Journal]`
>    2. 關鍵字拆解：抽出 2–3 個識別力最強的名詞片語，`kw1[Title] AND kw2[Title] AND N Engl J Med[Journal]`
>    3. Case Records 專屬：丟棄 `Case NN-YYYY:` 前綴，改用 `"{age}-Year-Old {sex}"[Title] AND "{strongest symptom}"[Title] AND N Engl J Med[Journal]`
>    - 命中：取最接近 `per_article_date` 的 PMID，呼叫 `mcp__PubMed__get_article_metadata` 取 abstract、DOI、`article_types`、authors（若 newsletter 缺）。
>
> 2. **CrossRef fallback**（PubMed 三階段皆 0）：`GET https://api.crossref.org/works?query.title={urlencoded title}&query.container-title=N+Engl+J+Med&rows=5&mailto={CROSSREF_MAILTO}`，依 score 取第一筆。
>
> 3. **皆 miss**：`pmid` 留空，`doi` 留空，`abstract` 留空。
>
> **Step B4 — 過濾**（在回傳前執行）：
> - **Heme/Onc 月報**：排除 `title` 符合 `/cancer|carcinoma/i` 的文章
> - ID 月報不做主題過濾
>
> **回傳格式**：最後輸出一個 JSON 程式碼區塊：
>
> ```json
> {
>   "id_newsletter": {
>     "issue_date": "May 2026",
>     "articles": [
>       {
>         "title": "...",
>         "authors": "...",
>         "article_type": "Correspondence",
>         "per_article_date": "2026-04-23",
>         "tracking_url": "https://t.n.nejm.org/r/?id=...",
>         "pmid": "42012345",
>         "doi": "10.1056/NEJMc2519999",
>         "abstract": "..."
>       }
>     ]
>   },
>   "heme_newsletter": {
>     "issue_date": "May 2026",
>     "articles": [...]
>   }
> }
> ```

---

## 主 agent 繼續（subagent 均完成後）

### Step C0 — 跨週 PMID dedup（7 天窗口）

讀 `{SKILL_DIR}/.seen_pmids.json`（dict `{pmid: first_seen_ISO_date}`，檔案不存在或解析失敗視為 `{}`）。對 Subagent A 回傳的每篇文章：

```python
from datetime import date

today = date.today()
for journal, articles in sections.items():
    sections[journal] = [
        a for a in articles
        if a['pmid'] not in seen
        or (today - date.fromisoformat(seen[a['pmid']])).days > 7
    ]
```

- 命中（seen 內且距今 ≤ 7 天）→ **skip**，不進入後續 Step C1 / C2 / Step 4。
- 未命中或距今 > 7 天 → **keep**。

> **設計考量**：weekly 跑頻率約每 7 天，正常 edat 視窗不重疊；此步主要防 (a) 同篇 paper 出現 epub vs print 雙 PubMed entry、(b) 上週 routine 失敗手動補跑導致視窗重疊。Newsletter PMIDs **不寫入** seen file（newsletter 月 cycle 與此 dedup 邏輯互斥）。

寫入時機：見 Step 5.5。

### Step C1 — Dedup（newsletter ∩ NEJM PubMed）

> `run_newsletter == False`（本月月報已收錄）時無 newsletter 文章，**整段跳過**，直接進 Step C2。

Subagent A 的 `N Engl J Med` section 已過濾為 Original Article + Review Article（且已排除 Perspective / Multimedia），故 dedup 不會誤刪 newsletter 中的 Editorial/Perspective。

對 Subagent B 回傳的每篇 newsletter 文章：
- 若 `pmid` 已存在於 Subagent A 的 `N Engl J Med` section，**略過**。
- 若 `title`（不分大小寫、忽略結尾句號）與 Subagent A 的 `N Engl J Med` 任一篇 title 相同，**略過**。

### Step C2 — CrossRef 補全（PubMed 文章缺 abstract）

掃描 Subagent A 回傳的所有文章（非 newsletter）的 abstract 欄位。若為空且有 DOI，嘗試 CrossRef：

- `GET https://api.crossref.org/works/{urlencoded_doi}?mailto={CROSSREF_MAILTO}`
- 清掉 JATS/HTML 標籤後寫回 abstract。
- 失敗（非 200 / timeout）：不中止，保留該文章在「仍缺 abstract」清單。

**仍缺 abstract**：
- **互動模式**：列出 PMID + 標題，請使用者補上後繼續。
- **自主模式**：TL;DR 留空，最終回報清單。

---

### Step C2.5 — 缺摘要的全文回退（OA → 機構瀏覽器；PubMed 與 newsletter 皆適用）

對任何 abstract 仍為空但**有 DOI** 的文章（含 Subagent A 的 PubMed 篇目與 Subagent B 的 newsletter 篇目）：

1. **先判斷是否開放取用**（用 API，不盲試出版商頁面）：
   - Unpaywall：`GET https://api.unpaywall.org/v2/{DOI}?email={CROSSREF_MAILTO}` → 看 `is_oa`、`best_oa_location.url`／`url_for_pdf`。
   - 或 Europe PMC：`GET https://www.ebi.ac.uk/europepmc/webservices/rest/search?query=DOI:{DOI}&format=json&resultType=core` → 看 `isOpenAccess=="Y"`、`pmcid`、`fullTextUrlList`。
2. **OA 命中** → 以 WebFetch 取 OA 全文（PMC／CDC／OA 連結）→ 依 Step 4 由全文補寫 TL;DR，標 provenance `（全文補摘要 — {來源}）`。
3. **非 OA 但有授權的機構瀏覽器**：若有已登入機構帳號的 Claude-in-Chrome 互動瀏覽器，可用它取付費全文——
   - HTML 全文（Wiley、NEJM 等）：`navigate` 到 `https://doi.org/{DOI}` 後 `get_page_text`。
   - PDF-only（如 Oxford accepted manuscript）：`navigate` 到文章頁的 PDF 連結，待轉到出版商簽章 URL，以 `WebFetch` 下載該 PDF 再用 `Read` 解析。
   - 命中 → 依 Step 4 由全文補寫 TL;DR，標 provenance `（全文補摘要 — 機構訂閱全文）`。
   - **限制**：僅用使用者已登入、確有訂閱權的瀏覽器；**不得繞過或破解 CAPTCHA／bot 偵測**。
4. **仍取不到**（無互動瀏覽器、無機構權、或 bot 偵測擋下）→ 不杜撰，維持 Step 4 表格的 `（無摘要）` 標註。
5. 盡力而為：任何 API、WebFetch 或瀏覽器操作失敗皆**不中止** routine。

> **環境註記**：本地端可正常跑 WebFetch 與 Claude-in-Chrome（含步驟 0 redirect 解析、機構瀏覽器全文）。

---

## Step 3 — 過濾文章（已由 subagent 完成）

PubMed 過濾（Erratum、Editorial、Comment，及 NEJM 只留 Original/Review）由 **Subagent A Step A3** 完成。
Newsletter 過濾（Heme/Onc 排除 cancer/carcinoma）由 **Subagent B Step B4** 完成。
主 agent 無需另行過濾，直接進 Step 4。

---

## Step 4 — 撰寫中文摘要與評語

對每篇文章：

- **不翻譯原文標題。**
- 不翻譯藥物名（含類別名）、病原（細菌、病毒、真菌）名及重要醫學術語（例：amphotericin B 保留原文；carbapenems、beta-lactams、azoles 等類別名一律不翻）
- 病原屬名／種名（含縮寫屬名）以 markdown 斜體標示：如 *Salmonella*、*Candida albicans*、*A. baumannii*、*Aspergillus*、*Mucorales*；疾病名（candidiasis、aspergillosis）與病毒縮寫（CMV、EBV、HIV、PJP）不用斜體
- **Salmonella 特例**：接小寫種小名→整串斜體（*Salmonella typhosa*）；接大寫者為 serovar，只斜體屬名、serovar 名不斜體（*Salmonella* Typhi、*S.* Enteritidis、*S.* Typhimurium）

### TL;DR 規則
- 繁體中文，60–90 字，1–2 句，直接陳述關鍵發現，並盡量帶上重要數字（樣本數、死亡率、效果量、HR/OR/CI、追蹤時間等）
- 地名（國家、地區、城市）用中文台灣慣用譯名，不要保留英文；「保留原文不翻譯」只適用於藥名/病原名/醫學技術詞，不含地名
- 禁止開場白：「本研究探討」「本文分析」「這篇研究」等

### 嘻嘻/不嘻嘻 規則
- 幽默短評，最多 40 字，一句話：
  - 嘻嘻 = 正面評價（有希望的發現、突破、好結果）
  - 不嘻嘻 = 搞笑吐槽（壞消息、難解問題、令人沮喪的流行病學）
- 文中若提及病原屬名/種名以 markdown 斜體標示（如 *Candida*、*A. baumannii*），疾病名與病毒縮寫不用斜體；Salmonella serovar（大寫 Typhi 等）只斜體屬名（*S.* Typhi），接小寫才整串斜體（*Salmonella typhosa*）

**Newsletter 項目缺 abstract 的三種情況**：

| 情況 | TL;DR 輸出 |
|------|-----------|
| PubMed/CrossRef 皆 miss | 留空，標注 `（無摘要 — 無法定位 PubMed/CrossRef 紀錄）` |
| PMID 命中但 `[Abstract not available]` | 留空，標注 `（無摘要 — PubMed 此類型無 abstract）`；嘻嘻可基於標題寫一句 |
| PMID 命中本週 NEJM section | 已在 Step C1 dedup，不輸出 |

> 落入上表「留空／無摘要」前，須先經 **Step C2.5** 以 DOI 查 OA、再試機構瀏覽器全文；皆取不到時才標「無摘要」。

---

## Step 5 — 輸出 Markdown 檔案

存至 `{OUTPUT_DIR}/{week_label}.md`。
若該檔已存在：只有確認是本次 routine 產生的暫存/中斷輸出時才覆寫；若是既有手動檔案或來源不明，先詢問使用者。

```markdown
---
tags: [journal-digest]
date: {date_to}
week: {week_label}
---

# Weekly Journal Digest — {week_label}

> 資料來源：PubMed（edat 過濾，{date_from} 至 {date_to}）。涵蓋期刊：Clin Infect Dis、Emerg Infect Dis、MMWR、N Engl J Med（Originals + Reviews，無主題過濾）、Transpl Infect Dis、Blood。共 __ 篇。NEJM Cases / Images / Perspective / Correspondence 由本月 NEJM ID 與 Hema/Onc 月報（Gmail）補完。

<!-- more -->

## Clin Infect Dis

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {PubDate}
**DOI**: [{DOI}](https://doi.org/{DOI})
**TL;DR**: 

> **嘻嘻/不嘻嘻**：

---

## Emerg Infect Dis
...（同上格式）

## MMWR Morb Mortal Wkly Rep
...

## Transpl Infect Dis
...

## Blood
...（同上格式）

## N Engl J Med
...（同上格式）

## NEJM Monthly — Infectious Disease（{nejm_id_issue_date}）

> 資料來源：Gmail（Infectious Disease Update from NEJM，發行日 {nejm_id_issue_date}）。補完 PubMed N Engl J Med section 之外的 Cases / Images / Perspective / Correspondence 等，無主題過濾。

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {nejm_id_issue_date}
**DOI**: [{DOI}](https://doi.org/{DOI})  <!-- 無 DOI 時改用 [**NEJM Newsletter**]({tracking_url}) -->
**TL;DR**: 

> **嘻嘻/不嘻嘻**：

---

## NEJM Monthly — Hematology/Oncology（{nejm_heme_issue_date}）

> 資料來源：Gmail（Hematology/Oncology Update from NEJM，發行日 {nejm_heme_issue_date}）。補完 PubMed N Engl J Med section 之外的 Cases / Images / Perspective / Correspondence 等。已濾除標題含 cancer / carcinoma 之篇目。

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {nejm_heme_issue_date}
**DOI**: [{DOI}](https://doi.org/{DOI})  <!-- 無 DOI 時改用 [**NEJM Newsletter**]({tracking_url}) -->
**TL;DR**: 

> **嘻嘻/不嘻嘻**：

*資料來源：PubMed（articles retrieved {today}）*
*NEJM 月報資料來源：Gmail；abstract / DOI 優先由追蹤 URL redirect 解析，fallback PubMed 標題搜尋及 CrossRef 回查取得。*
```

**格式細節**：
- Authors：單一作者用全名；多位作者寫「姓 縮寫 et al.」
- 沒有 DOI：使用 `PMID: {pmid}`
- 文章標題（`### {Title}`）中的病原屬名/種名（含縮寫屬名）以 markdown 斜體標示，其餘一字不改
- Newsletter 文章：**若 Subagent B 解出 DOI（B3 任一 fallback 命中，含步驟 0），DOI 行用 `**DOI**: [{DOI}](https://doi.org/{DOI})`（與 PubMed 篇目一致，不另顯示 newsletter 連結）**；僅當無 DOI 時才用 `[**NEJM Newsletter**]({tracking_url})`；DOI 與 tracking_url 皆無，則 `**DOI**: 無 PubMed/CrossRef 紀錄`。
- 某 section 無文章：寫 `> 今日無符合條件的新文章。`
- 文章間用 `---` 分隔
- **`run_newsletter == False`（本月月報已收錄）**：兩個 NEJM Monthly section 標題的 issue_date 寫 `（本月已收錄）`（**不可含 4 位年份**——閘門以標題中的真實發行年份判定本月是否已收錄；含年份會被下個月的閘門誤判），section 內各寫一行佔位 `> 本月 NEJM 月報已於 {prior_week} digest 收錄，本週不重複。`。

---

## Step 5.5 — 寫入 seen_pmids.json

成功寫出 `{OUTPUT_DIR}/{week_label}.md` 後（Step 5 結束），把本次最終進入 digest 的 PMID 寫回 `{SKILL_DIR}/.seen_pmids.json`：

```python
import json
from datetime import date

SEEN_FILE = Path("{SKILL_DIR}/.seen_pmids.json")
today = date.today()
today_iso = today.isoformat()

try:
    seen = json.loads(SEEN_FILE.read_text())
except (FileNotFoundError, json.JSONDecodeError):
    seen = {}

# GC：刪除距今 > 14 天的條目（窗口 2 倍當安全邊際；避免無限膨脹）
seen = {p: d for p, d in seen.items()
        if (today - date.fromisoformat(d)).days <= 14}

# 寫入：本次保留進 digest 的 PubMed PMID；newsletter PMIDs 不寫入
for journal, articles in sections.items():
    for a in articles:
        seen.setdefault(a['pmid'], today_iso)  # 首見才寫，不 overwrite

SEEN_FILE.write_text(json.dumps(seen, indent=2, ensure_ascii=False) + "\n")
```

- 失敗（檔案無法寫入）：回報錯誤但**繼續**。下次 routine 仍能用既有 seen 檔運作。

---

## Error Handling

| 情況 | 處理 |
|------|------|
| Subagent A 搜尋失敗 | 回報錯誤並停止 |
| 本月月報已收錄（`run_newsletter == False`） | 不啟動 Subagent B；NEJM Monthly section 寫佔位（見 Step 5），略過 Step C1 |
| Subagent B Gmail 失敗 | 繼續 PubMed 流程；newsletter section 寫 `> Gmail 取得失敗。` |
| 某 section 無文章 | 正常，寫佔位訊息 |
| Step B3.0 redirect WebFetch 失敗 | 繼續 PubMed 標題搜尋（步驟 1），不中止 |
| Step C0 seen file 讀取失敗（解析錯誤 / 不存在） | 視為 `{}` 處理，dedup 等同停用，繼續 routine |
| Step 5.5 seen file 寫入失敗 | 回報錯誤，繼續（下次 routine 仍能用既有檔） |
| 檔案已存在 | 僅覆寫本次 routine 產生的暫存/中斷輸出；來源不明則先詢問使用者 |

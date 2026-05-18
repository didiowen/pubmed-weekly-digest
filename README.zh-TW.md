# pubmed-weekly-digest

> [English version](./README.md)

一個 Claude Code skill，針對你預先設定的一份 PubMed 期刊清單，彙整過去 7 天內刊出的文章，為每篇文章寫一段 TL;DR 與一句話的 **Hot Take**（捧或酸都可以），最後輸出成一份本機的 Markdown 檔案。輸出預設是英文，想換語言只要改一處 prompt 就好。

## 資料夾結構

```
pubmed-weekly-digest/
├── SKILL.md          # skill 本體
├── README.md         # 英文版
├── README.zh-TW.md   # 本檔案
└── LICENSE
```

沒有附帶任何腳本，整個 skill 完全由 prompt 驅動。

## 相依套件

- **Claude Code**（含 Skill 系統）—— `https://claude.com/claude-code`
- 在 Claude Code MCP 設定中啟用的 **PubMed MCP server**
- 就這些。不需要 Python 套件，最小執行流程也不需要 `.env`。

## 安裝設定

1. 把整個資料夾放到任一專案的 skills 目錄底下：`<project>/.claude/skills/pubmed-weekly-digest/`。
2. 確認 PubMed MCP server 已經設好。
3. 打開 `SKILL.md`，編輯檔案頂端的 **Configuration** 區塊：
   - `OUTPUT_DIR`：摘要寫到哪裡（預設是 skill 資料夾下的 `output/`）
   - `JOURNALS`：Step 1 中的期刊清單
   - `CROSSREF_MAILTO`：任一個可用的電子郵件，給 CrossRef polite-pool 識別用
4. 從 Claude Code 觸發：直接跟它說「跑週報期刊摘要」（run the weekly journal digest）、「跑週報」，或以 skill 名稱呼叫。

## 自訂期刊清單

這通常是你最想動的部分。

### 清單放在哪裡

`SKILL.md` → **Step 1 — PubMed search** 有一張表格：

```markdown
| Query string                          | max_results |
|---------------------------------------|-------------|
| `Clin Infect Dis[Journal]`            | 50          |
| `Emerg Infect Dis[Journal]`           | 50          |
| `MMWR Morb Mortal Wkly Rep[Journal]`  | 30          |
| `Transpl Infect Dis[Journal]`         | 50          |
| `Blood[Journal]`                      | 50          |
| `N Engl J Med[Journal]`               | 50          |
```

每一列會展開成一次平行的 `mcp__PubMed__search_articles` 呼叫。要換不同期刊就替換列、要追更多期刊就新增列、不需要的就刪掉。**同時記得更新 Step 5 的 section 清單**，因為每個期刊在 Markdown 模板裡都有自己的 `## {期刊名稱}` 區塊。

### 找到正確的期刊縮寫

PubMed 的 `[Journal]` 欄位要的是 **NLM Title Abbreviation**，不是全名。有三種找法：

1. 在 PubMed 上搜該期刊 → 打開任一篇近期文章 → citation 字串會顯示縮寫，例如 `Clin Infect Dis. 2026 Apr 30;82(8):...`。
2. 瀏覽 https://www.ncbi.nlm.nih.gov/nlmcatalog/journals → 以全名查 → "NLM Title Abbreviation" 欄位就是要貼進 `[Journal]` 的內容。
3. 先用 `mcp__PubMed__search_articles` 以全名做一次測試查詢，PubMed 會自動 normalise，回傳結果會顯示它實際使用的縮寫。

幾個常見縮寫：

| 期刊全名                                | `[Journal]` 用法          |
|-----------------------------------------|---------------------------|
| Clinical Infectious Diseases            | `Clin Infect Dis`         |
| Emerging Infectious Diseases            | `Emerg Infect Dis`        |
| The New England Journal of Medicine     | `N Engl J Med`            |
| The Lancet                              | `Lancet`                  |
| The Lancet Infectious Diseases          | `Lancet Infect Dis`       |
| JAMA                                    | `JAMA`                    |
| Journal of Clinical Oncology            | `J Clin Oncol`            |
| Nature Medicine                         | `Nat Med`                 |
| The Journal of Infectious Diseases      | `J Infect Dis`            |
| Blood                                   | `Blood`                   |
| BMJ                                     | `BMJ`                     |
| Annals of Internal Medicine             | `Ann Intern Med`          |

### 如何決定 `max_results`

MCP wrapper 每條查詢的上限是 **200 筆**。7 天視窗的粗估：

- 高刊量專科期刊（CID、EID、Blood）：`50` 通常夠用。
- 週報（MMWR、Lancet）：`30` 可涵蓋一整期。
- 高影響力綜合期刊（NEJM、JAMA）：`50` 是安全上限，單期通常不到 30 篇。
- 月刊（多數次專科期刊）：`30` 綽綽有餘。

如果經常看到結果被截斷，就把數字拉高。但不要無腦設成 200，metadata 抓取階段是以每批 20 筆 fan out，結果集太大會明顯變慢。

### 為高刊量期刊加上主題篩選

像 NEJM、JAMA 這類綜合期刊，你大概只想留下涉及特定主題的文章，用 `[tiab]`（title 加 abstract）做布林 AND 就好：

```
N Engl J Med[Journal] AND (infection[tiab] OR antimicrobial[tiab] OR sepsis[tiab] OR HIV[tiab] OR transplant[tiab])
```

這樣可以把範圍收斂到 ID 相關的 NEJM 文章，同時仍以 `edat` 控制時間視窗。

### PubMed MCP 的兩個坑

都是踩過的：

- **不支援萬用字元 `*`**。請改用 `mycobacterium`，或明確展開成 `mycobacterium OR tuberculosis OR NTM`。
- **布林運算子上限是 20**。一條查詢若超過 20 個 `OR`／`AND` 會以 `INVALID_PARAMETERS` 失敗，遇到時拆成兩條平行查詢即可，skill 的其餘部分會自動合併結果。

### 寫進 SKILL.md 之前先試查詢

建議先在 PubMed 網頁版跑一次：

```
N Engl J Med[Journal] AND (infection OR sepsis) AND 2026/05/04:2026/05/10[edat]
```

網頁查得對，經由 MCP 跑同樣查詢（把日期範圍改用 `date_from`／`date_to` 參數傳入）就會回傳相同結果。

## 調整輸出風格

Step 4 的標註預設是 **英文**，一句話評論用 **Hot Take** 框架，可正面也可酸。兩個選擇都純粹是 prompt 設定，想改就改：

- **語言**：在 Step 4 的指令裡指定你想要的語言（例如繁體中文、Spanish、Japanese、French）。PubMed 與 CrossRef 回傳的摘要是原始語言（通常是英文），Claude 會在標註時即時翻譯，skill 其餘部分都不用動。
- **語氣標籤**：把 "Hot Take" 換成任何你喜歡的詞，例如 "key takeaway"、"clinical implication"、"bottom line"，或中文的「臨床意義」、「重點」。Step 5 每篇文章兩行（TL;DR 加引述）的格式不變，只換用詞。
- **長度與深度**：預設 1–2 句，想看得更深就放寬成一段，要更速覽就縮成一句。

## License

MIT，詳見 [`LICENSE`](./LICENSE)。

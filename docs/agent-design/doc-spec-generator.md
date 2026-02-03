# Cursor Agent Skill 設計：doc-spec-generator

> 目標：  
> 將原本 n8n 的 `drive_ingest_normalize` + `llm_generate_spec`  
> 重構為 **Cursor IDE 可執行的 Agent Skill**  
> 以「檔案導向、規格導向、可重試」為核心設計。

---

## 一、設計總覽

### 原本（n8n）

- Workflow A：檔案接收、轉檔、Normalize
- Workflow B：LLM 產生規格文件

### 改造後（Cursor Agent）

- 單一 Agent Skill：`doc-spec-generator`
- 任務導向、非事件導向
- 以資料夾作為輸入 / 輸出邊界

---

## 二、資料夾規約（Contract）

### 輸入資料夾

```text
AI_Input/
└─ incoming/
   ├─ xxx.pdf
   ├─ yyy.docx
   ├─ zzz.pptx
   ├─ aaa.png
   └─ bbb.jpg
```

**支援檔案類型**

- PDF
- DOCX
- PPTX
- PNG
- JPG

---

### 輸出資料夾

```text
output/
└─ {fileId}/
├─ normalized.md
├─ system_plan.md
├─ flow.mmd
├─ tech_spec.md
└─ error.log（僅失敗時產生）

```

---

## 三、Agent Skill：doc-spec-generator

### 職責說明

> 將一個輸入文件，轉換為一組「可實作的工程規格文件」

---

## 四、Skill 流程拆解

### Skill 1：scan_incoming_files

**用途**

- 取代 Google Drive Trigger + Filter

**行為**

- 掃描 `AI_Input/incoming`
- 僅接受指定副檔名
- 為每個檔案建立：
  - fileId
  - mimeType
  - absolutePath

---

### Skill 2：convert_to_raw_markdown

**用途**

- 取代 Download + HTTP /convert + IF error

**轉換策略**

- PDF → pdf-to-markdown
- DOCX → pandoc
- PPTX → libreoffice
- PNG / JPG → OCR

**輸出**

- rawMarkdown（string）

**錯誤處理**

- 轉換失敗時：
  - 建立 `output/{fileId}/error.log`
  - 記錄 converter 名稱與錯誤訊息
  - 中止該檔案後續流程（不影響其他檔案）

---

### Skill 3：normalize_markdown

**用途**

- 取代 n8n 的 Code：Normalize Markdown

**Normalize 規則**

- 統一 heading 層級
- 移除多餘空行
- 修正表格結構
- 圖片轉為 placeholder（不保留 base64）
- 章節順序固定為：
  1. 摘要
  2. 背景
  3. 需求
  4. 流程
  5. 限制

**輸出**

`normalized.md`

---

### Skill 4：generate_spec_documents

**用途**

- 取代 Workflow B（LLM Generate + Validate）

**輸入**

- normalizedMarkdown
- fileId

**LLM 輸出（JSON Schema）**

```json
{
  "system_plan_md": "...",
  "flow_mmd": "...",
  "tech_spec_md": "..."
}
```

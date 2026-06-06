# NLR-workflow

**Narrative Literature Review workflow — AI-Assisted Workflow with Claude Code**

---

很多AI科研项目趋向于去做一个“全能战士”，但是实际上，我们在科研过程中需要解决的是 **十分具体的问题**。据此，我产生了一个想法，每次只用AI做好一件具体的事。首次给大家带来的便是此项目—**Narrative Literature Review workflow (NLR-workflow)**，专为解决叙述性文献综述而生。

这个仓库包含一套完整的、由 AI 辅助的**叙述性文献综述（Narrative Literature Review, NLR）**工作流，适用于生物医学领域的综述型文章写作。工作流以 Claude Code 为核心，结合 Python 自动化脚本、Zotero 文献管理和 Pandoc 排版，覆盖从文献检索到 Word 稿件导出的全部流程。

> 本仓库以癌症多模态 AI 综述项目为模板：
> **Multimodal Digital Twins for Cancer: Integrating Pathology, Radiology, and Multi-Omics AI**
> 内容可按需替换为任意生物医学选题。

---

## 目录

- [工作流概览](#工作流概览)
- [目录结构](#目录结构)
- [环境配置](#环境配置)
- [Claude Code 自定义技能](#claude-code-自定义技能)
- [阶段一：文献检索](#阶段一文献检索)
- [阶段二：文献筛选](#阶段二文献筛选)
- [阶段三：资料提取](#阶段三资料提取)
- [阶段四：综合与大纲](#阶段四综合与大纲)
- [阶段五：写作](#阶段五写作)
- [阶段六：润色（五个串行 Pass）](#阶段六润色五个串行-pass)
- [阶段七：参考文献与 Word 导出](#阶段七参考文献与-word-导出)
- [写作规范要点](#写作规范要点)
- [工具技能速查](#工具技能速查)

---

## 工作流概览

```
Phase 1  文献检索    PubMed API → pubmed_results.csv
Phase 2  文献筛选    AI 批量打分 → 人工复查 → final_screened.csv
Phase 3  资料提取    PDF/摘要 → AI 结构化提取 → extractions/*.md
Phase 4  综合分析    聚类 → 差距分析 → 大纲草稿
Phase 5  写作        /lit-draft 逐节起草 → manuscript_v1.md
Phase 6  润色        6a 事实审查 → 6b 去AI痕迹 → 6c 句式多样 →
                     6d 框架密度 → 6e 连贯性 → 6e-gate 投稿门控
Phase 7  参考文献    /lit-cite → pandoc + zotero.lua → manuscript.docx
```

**叙述性综述 vs 系统性综述**：本工作流专为叙述性综述设计。叙述性综述以论点驱动文献，而非枚举文献；行文中应避免使用精确文献计数（如"本综述收录了89篇研究"）或系统综述惯用语，这是工作流中多个 pass 重点检查的内容。

---

## 目录结构

```
literature-review/
├── CLAUDE.md               ← Claude Code 项目配置（每次会话自动读取）
├── README.md               ← 本文件
├── pyproject.toml          ← uv 依赖管理
├── uv.lock                 ← 锁定版本（提交到 git）
├── LOG.md                  ← 会话日志
├── search/                 ← PubMed 原始检索结果（CSV）
├── screening/              ← 筛选表格（ai_screened / borderline / final）
├── pdfs/                   ← 全文 PDF + pdf_map.csv（DOI → 文件名映射）
├── extractions/            ← 每篇文献一个 .md 提取笔记
├── synthesis/              ← 聚类、差距分析、审查报告
├── outline/                ← 大纲草稿（v1, v2, …）
├── draft/                  ← 各节草稿（每节一个文件）
├── manuscript/             ← 完整稿件（v1 → v2 + 最终 .docx）
├── references/             ← .bib 文件（Better BibTeX 导出）
└── scripts/                ← Python 自动化脚本
```

**规则**：所有文件必须写在上述结构内，不得在结构外新建文件（除非与用户确认）。

---

## 环境配置

### 1. 安装 uv（Python 包管理器）

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### 2. 同步项目依赖

```bash
uv sync
```

> **注意**：始终使用 `uv run python scripts/xxx.py` 运行脚本，`uv add <包名>` 添加新依赖，**不要**直接使用 `pip install`。

### 3. 配置环境变量

在项目根目录创建 `.env` 文件（已加入 `.gitignore`，不会提交到仓库）：

> 需先在Deepseek官网申请Deepseek API key

```dotenv
# AI 批量提取（DeepSeek API，OpenAI 兼容格式）
DEEPSEEK_API_KEY=your_deepseek_key
SCREEN_MODEL=deepseek-v4-flash          # 默认值，可按需修改
```

### 4. 安装 Claude Code

> 详见 [Claude Code 官方文档](https://docs.anthropic.com/claude-code)。


### 5. Zotero 集成

本工作流使用 **NBIB 批量导入**（使用 Zotero 本地API），可获取完整元数据（卷号、页码、MeSH 词）。

> Zotero 本地API开启方式：Zotero Edit→settings→Advanced→勾选 Allow other applications on this computer to communicate with Zotero Available at http://localhost:23119/api/

**导入流程：**

1. 运行 `uv run python scripts/export_nbib.py` 生成 `search/included_papers.nbib`
2. 在 Zotero 中：文件 → 导入 → 选择上述 `.nbib` 文件
3. 导入完成后，将自动生成的集合重命名为 `DigitalTwins-Cancer-Review`（或你的项目名）

**Better BibTeX 插件**（引文导出必需）：

- 安装后，Zotero 会为每条文献分配稳定的 citekey（格式如 `Chen2023`）
- 通过"文件 → 导出文献库"，选择 Better BibTeX 格式，导出到 `references/library.bib`

**Word 引文嵌入（Phase 7）**：

- 下载 [zotero.lua](https://retorque.re/zotero-better-bibtex/exporting/zotero.lua) 并放置于 `manuscript/` 目录
- 运行 pandoc 时使用 `--lua-filter=zotero.lua`，会自动调用 Zotero JSON-RPC API 嵌入活引用字段
- **不要**使用 `--citeproc` 或 `--bibliography`（会生成静态文本，无法通过 Word Zotero 插件编辑）

---

## Claude Code skill

本工作流包含以下 Claude Code 自定义技能（位于 `.claude/skills/` 目录），在 Claude Code 会话中直接使用斜杠命令调用：

| 技能命令 | 功能 |
|---|---|
| `/lit-status` | 会话启动状态面板：读取 LOG.md、统计各目录文件数、推断当前阶段、建议下一步 |
| `/lit-audit` | 事实准确性审查：核查稿件中每一个数字（AUC、样本量、数据集名称）是否与提取笔记一致 |
| `/lit-deregister` | 去 AI 写作痕迹：检测并修改对立构式、评价性形容词、重复句式开头，以及系统综述惯用语 |
| `/lit-draft [节号]` | 按大纲逐节起草：读取大纲字数预算、从提取笔记核实数据、执行引文密度和行文规范检查 |
| `/lit-gate` | 投稿门控检查：测量字数、破折号频率、引文密度、摘要完整性、占位符残留等指标，返回 PASS/WARN/FAIL |
| `/lit-cite` | 引文占位符替换：解析 `library.bib`，将 `[Chen2023]` 格式替换为 pandoc `[@citekey]` 格式 |

---

## 阶段一：文献检索

### 构建 PubMed 检索脚本

> 需先在NCBI官网申请 Entrez API key

将以下提示词复制到 Claude Code 执行，生成 `scripts/pubmed_search.py`：

```
Read CLAUDE.md. Then write a Python script at scripts/pubmed_search.py that:
1. Queries PubMed using the Entrez API (use Biopython's Entrez module)
2. Runs the following search strings (one at a time, with a 1-second delay between):
   - ("digital twin" OR "digital twins") AND (cancer OR tumor OR oncology)
   - (computational pathology OR whole slide image) AND (radiology OR imaging) AND (multimodal OR multi-modal)
   - (multi-omics OR multiomics) AND (cancer OR tumor) AND (deep learning OR machine learning OR AI) AND (patholog OR radiol OR imaging OR mri OR ct scan OR histolog OR whole slide OR wsi)
   - (foundation model) AND (pathology OR radiology) AND (oncology OR cancer)
3. Deduplicates results by PMID
4. Outputs to search/pubmed_results.csv with columns: PMID, Title, Authors, Journal, Year, Abstract, DOI
5. Prints a summary: total hits per query, total after deduplication
Use email = "xxx@xxx.com" for the Entrez API xxxxx（填入你的API key）.
```

> 根据你的研究主题修改检索式，其余部分不变。脚本生成后运行：`uv run python scripts/pubmed_search.py`

---

## 阶段二：文献筛选

### 步骤 2a — AI 批量打分

生成 `scripts/batch_screen.py`（复制到 Claude Code 执行）：

```
Read CLAUDE.md.
Write scripts/batch_screen.py that processes search/pubmed_results.csv as follows:

SETUP:
- Drop rows where Abstract is NaN before processing
- Process in batches of 25 rows
- After each batch, APPEND results to screening/ai_screened.csv
  (write CSV header only when creating the file for the first time)
- On startup, check if screening/ai_screened.csv already exists;
  if so, read already-processed PMIDs and skip them — this allows safe resume after interruption

SCORING RUBRIC (apply to Title + Abstract only):

  Score 2 — Include. Paper meets ANY of:
    • Proposes or applies a cancer digital twin framework
    • Integrates ≥2 modalities from {pathology/WSI, radiology/CT/MRI/PET/ultrasound,
      omics/genomics/transcriptomics/proteomics} for a cancer task
    • Multimodal foundation model for oncology involving pathology + imaging or + omics

  Score 1 — Borderline. Paper meets ANY of:
    • Single-modality but methodologically landmark (e.g. top-tier WSI foundation model,
      pan-cancer imaging atlas)
    • Radiomics + omics integration without a distinct imaging modality
    • Digital twin for surgical/radiotherapy planning without multimodal AI component
    • Review or perspective broadly relevant to the topic scope

  Score 0 — Exclude:
    • Pure omics study (no imaging or pathology component)
    • Pure imaging or pathology (no omics integration, not multimodal)
    • Clinical trial design, statistical methods, pharmacokinetics
    • Non-cancer disease context
    • Only tangentially mentions AI/cancer with no in-scope method

OUTPUT: for each paper write one row to screening/ai_screened.csv with columns:
  PMID, Title, Year, Journal, DOI, Score, Reason
  Keep Reason to ≤15 words.

After all batches complete, print:
  Total processed | Score=2 (Include) | Score=1 (Borderline) | Score=0 (Exclude)

Run with: uv run python scripts/batch_screen.py
```

> **修改说明**：将评分标准中的研究主题替换为你的综述范围，其余框架结构不变。

### 步骤 2b — 生成人工复查表

打分完成后，在 Claude Code 中执行：

```
Read screening/ai_screened.csv and search/pubmed_results.csv.
Generate screening/borderline_review.csv containing ONLY Score=1 papers.

Steps:
1. Filter ai_screened.csv to rows where Score = 1.
2. Join with pubmed_results.csv (encoding="latin-1") on PMID to bring in Abstract.
3. Sort by Year descending, then PMID descending within the same year.
4. Output screening/borderline_review.csv with exactly these 8 columns in order:
     PMID, Title, Year, Journal, Abstract, DOI, Reason, Include
   - Include column: leave blank (empty string) for every row.
   - Use encoding utf-8-sig so Excel opens it correctly.
5. Print: "Written: screening/borderline_review.csv — N rows"
```

### 步骤 2c — 人工判断（手动操作）

用 Excel 打开 `screening/borderline_review.csv`，在 `Include` 列填写：
- `1` = 纳入（升为 Score=2）
- `0` = 排除（降为 Score=0）
- 留空 = 维持边界（会触发警告）

保存后对 Claude Code 说：**"I've filled in borderline_review.csv, please sync decisions back to ai_screened.csv"**

### 步骤 2c — 同步决策到 CSV

```
Read screening/borderline_review.csv and screening/ai_screened.csv.
Parse the Include column: for each PMID, read the value.
Update the Score column in ai_screened.csv:
  - If Include = 1: set Score = 2
  - If Include = 0: set Score = 0
  - If blank / missing: leave Score = 1 and print a warning with the PMID
Save the updated data to screening/final_screened.csv (do not overwrite ai_screened.csv).
Columns of final_screened.csv: PMID, Title, Year, Journal, DOI, Score, Reason.
Print: final Include count (Score=2) / Exclude count (Score=0) / still-borderline (Score=1).
```

### 步骤 2d — 导出 NBIB 以供 Zotero 导入

```
Read screening/final_screened.csv.
Write scripts/export_nbib.py that:
1. Reads all PMIDs where Score = 2 from final_screened.csv
2. Uses Biopython Entrez.efetch (db="pubmed", rettype="medline", retmode="text")
   to fetch complete PubMed records in batches of 100
3. Writes all records to search/included_papers.nbib
4. Prints: PMIDs requested / records fetched / output path

Why NBIB (not Zotero API):
  - PubMed returns complete metadata: volume, issue, pages, MeSH, language, etc.
  - Zotero API-created items lack these fields and require a second metadata fetch.
  - One-click bulk import; Zotero auto-deduplicates on DOI.

Run with: uv run python scripts/export_nbib.py

Manual step after running:
  Zotero → File → Import → select search/included_papers.nbib
  (Zotero auto-creates a collection named "included_papers" — rename it to your project name after import)
```

---

## 阶段三：资料提取

### 构建批量提取脚本

此步骤为每篇纳入文献生成一个结构化 `.md` 提取笔记，包含架构名称、数据集、样本量、关键指标，供后续写作引用核验。

```
Read CLAUDE.md.
Write scripts/batch_extract.py for all Score=2 papers in screening/final_screened.csv.

SETUP: Load final_screened.csv, pdfs/pdf_map.csv, and search/pubmed_results.csv (latin-1);
join on PMID/DOI. Resume-safe: skip papers whose DOI already appears in any extractions/*.md.

PER-PAPER:
- PDF found in pdfs/: extract with pymupdf; parse major sections via regex on section
  headers; fall back to first/last N chars if a section is missing. Source = "Full text PDF".
- No PDF: use Abstract from pubmed_results.csv. Source = "Abstract only".

EXTRACTION INSTRUCTION (pass verbatim to model):
"Capture exact metric values (AUC, C-index, accuracy, etc.), specific dataset names, and
patient counts. Name the specific model/framework rather than a generic category.
Record whether validation was internal, external, or prospective."

Return JSON with fields:
  paper_type, subject_tags, in_scope_score,
  framework, architecture_name, key_techniques,
  datasets, datasets_detail [{name, n_patients, institution}],
  validation, validation_type, key_metrics [{metric, value, dataset, n}],
  key_findings, limitations, sections_fed, key_claim, conflicts

Render into extractions/{FirstAuthorYear}.md using the Extraction Note Format in CLAUDE.md,
plus this section appended after Methods:
  ## Quantitative Evidence
  - **Architecture name:** {architecture_name}
  - **Validation type:** {validation_type}
  - **Datasets (name · n patients · institution):** {datasets_detail}
  - **Key metrics (metric · value · dataset · n):** {key_metrics}

API: DeepSeek, OpenAI-compatible, same pattern as batch_screen.py.
.env: DEEPSEEK_API_KEY (required), SCREEN_MODEL (default: deepseek-v4-flash).

Print per-paper progress and final summary: from PDF / from abstract / errors.
Run with: uv run python scripts/batch_extract.py
```

### 提取进度检查

随时可在 Claude Code 中运行以下提示词检查提取完成情况：

```
Read CLAUDE.md. Read screening/final_screened.csv (Score = 2 rows) and list files in extractions/.
Report papers missing an extraction, split into: has PDF in pdfs/ vs no PDF.
Sort each list by Year descending.
```

---

## 阶段四：综合与大纲

### 主题聚类

```
Read CLAUDE.md. Read all files in extractions/.
Group the papers into thematic clusters. For each cluster, suggest a descriptive label and list the papers (by FirstAuthorYear) that belong to it.
Consider clustering by: modality combination, technical approach, cancer type, clinical application.
Output the clustering to synthesis/clusters.md in a readable Markdown table.
```

### Gap 分析

```
Read CLAUDE.md. Read synthesis/clusters.md and all extractions/.
Identify:
1. Modality combinations that are underrepresented
2. Cancer types rarely studied in multimodal digital twin work
3. Technical approaches mentioned as future directions but not yet implemented
4. Conflicting findings between papers
Output to synthesis/gap_analysis.md with each gap labeled as: Research Gap / Methodological Gap / Clinical Translation Gap.
```

### 生成大纲草稿

以下约束针对 npj Digital Medicine，可根据目标期刊调整字数限制和章节数量：

```
TARGET JOURNAL: npj Digital Medicine
HARD CONSTRAINTS:
- Total main text: 10,000–12,000 words (enforce strictly)
- Major sections (## level): 6–8 maximum
- Subsections (### level): 2–4 per major section maximum
- Required figures: plan 3–5 figures; describe each in one sentence
- Required tables: plan 1–2 tables; describe columns
- Introduction: no subsections — flowing paragraphs only
- Conclusion: no subsections — flowing paragraphs only

Read CLAUDE.md, synthesis/clusters.md, and synthesis/gap_analysis.md.
Generate a detailed section outline for the review. For each section, include:
- Section title
- 2–3 sentence description of what it covers
- Which thematic clusters/papers feed into it
- Approximate target word count

After the outline, output:
FIGURE PLAN:
- Figure 1: [title] — [one sentence description, what it depicts]
- Figure 2: ...
- Figure 3: ...

TABLE PLAN:
- Table 1: [title] — columns: [list them]

WORD BUDGET:
- Section 1 (Introduction): X words
- Section 2: X words
[... total must sum to 10,000-12,000]

Save both the outline AND the figure/table plan to outline/outline_v[x].md.
```

---

## 阶段五：写作

### 步骤 5a：逐节起草

```
/lit-draft [节号或节名]
```

**示例**：`/lit-draft 3`、`/lit-draft 3.1`、`/lit-draft "Pathomic Fusion"`

`/lit-draft` 技能会从 `outline/outline_current.md` 读取字数预算，自动发现并读取相关提取笔记，核验每一个引用数字，执行引文密度（目标每 60–80 词至少 1 条引文）、破折号限制和行文规范检查，最终保存到 `draft/[section_slug].md`。

**注意事项**：
- 每次调用只起草一节
- 按大纲顺序依次起草
- 技能结束时会提示下一节标识符

### 步骤 5b：组装完整稿件

```
Read CLAUDE.md and outline/outline_current.md.
Following the section order in outline_current.md, read all corresponding section
.md files under draft/ and assemble them into a complete review at
manuscript/manuscript_v1.md.

Requirements:
1. Order sections strictly as in the outline — do not reorder.
2. Add YAML front matter at the top of the file (for later pandoc use):
   ---
   title: "Your Review Title Here"
   ---
3. Normalize heading levels across sections (# = article title, ## = top-level
   section, ### = subsection); fix any level inconsistencies caused by drafting
   sections in separate files.
4. During assembly, perform a word count per section. If any section exceeds its outline budget
   by more than 20%, flag it with <!-- OVER BUDGET: ~X words, target Y --> and trim the most
   redundant paragraph to bring it within range. Removing a clearly redundant paragraph is
   permitted; rewriting for style is not.
5. After assembly, report the total word count. If total exceeds 12,000 words, identify the
   3 longest sections and flag the most cuttable paragraphs in each with
   <!-- TRIM CANDIDATE: [reason] -->.
6. Handle inter-section seams: if adjacent passages contain duplicate transition
   sentences or duplicate definitions, flag them with <!-- DUPLICATE: ... --> comments
   but do NOT auto-delete — leave them for me to confirm.
7. Keep all [FirstAuthorYear] placeholders exactly as-is.
8. At the end, print a checklist: which sections were assembled, and which are missing
   (present in the outline but not found under draft/).
```

### 步骤 5c：撰写摘要

```
Read manuscript/manuscript_v1.md.
Write a 150–180 word unstructured abstract for npj Digital Medicine as a SINGLE paragraph.

Content to cover (in this narrative order):
1. The clinical problem — why single-modality data is insufficient (1-2 sentences)
2. The integrative concept being reviewed, introduced naturally in the flow — NOT as a
   scope statement. Do not write "This review synthesizes/examines/covers X studies".
   Instead, name the concept and define it parenthetically in the same sentence that
   advances the argument.
3. The 2-3 main synthesis findings — what the evidence establishes (2-3 sentences)
4. The key barriers to clinical translation (1 sentence)
5. A forward-looking statement naming specific enabling technologies (1 sentence)

Hard prohibitions:
- NO "This review synthesizes / examines / covers / presents" meta-sentences
- NO specific paper counts (not "280 studies", not "89 pathomic papers", not "N=X")
- NO numbered subsections or bullet points
- NO abbreviations without definition on first use

Style requirements:
- Single paragraph only
- No citations
- Third person, present tense for established findings, future tense for directions
- Last sentence must name a specific technology or mechanism, not a vague "will enable"

Output the abstract immediately after the YAML front matter and before "## 1. Introduction".
Format: ## Abstract\n\n[abstract text]
```

---

## 阶段六：润色（五个串行 Pass）

**执行顺序**：6a → 6b → 6c → 6d → 6e → 6e-gate，不可跳过或合并。每个 pass 针对不同的错误模式。

```
manuscript_v1.md  →[6a audit]→  修复  →[6b /lit-deregister]→  manuscript_v2.md
  →[6c sentence-shape]→  manuscript_v3.md
  →[6d framing]→  manuscript_v4.md
  →[6e seams]→  manuscript_v5.md
  →[6e-gate /lit-gate]→  PASS 后进入 Phase 7
```

### 步骤 6a — 事实准确性审查（最高优先级）

```
/lit-audit
```

该skill自动发现稿件和 `extractions/`，核查每一个引用数字是否与提取笔记一致，报告保存到 `synthesis/audit_YYYYMMDD.md`。

**所有 MISMATCH 和 NOT_IN_EXTRACTION 问题必须在进入 6b 前修复。含未核实数字的稿件不得进入下一阶段。**

### 步骤 6b — 去 AI 写作痕迹

```
/lit-deregister
```

技能统计对立构式（"not X but Y"）、评价性形容词（striking、remarkable、unprecedented 等）、重复句式开头，以及系统综述惯用语（精确文献计数如"89 studies in this corpus"、穷举声明如"none of the 280 reviewed studies"），展示计数表后等待确认，修订后保存为 `manuscript_v2.md`。

### 步骤 6c — 句式多样性 pass

```
Read manuscript/manuscript_v2.md.

For each paragraph, inspect the opening word and syntactic structure of each sentence.
Flag any paragraph where 2 or more consecutive sentences share the same opening structure
(e.g., all begin with "The", all begin with a subject-verb, all are long compound sentences).

Rewrite flagged paragraphs for variety. The target rhythm, drawn from published npj Digital
Medicine reviews, is: an opening claim sentence (~15–20 words) that states what the paragraph
argues; two to four evidence sentences each naming a specific study with its key finding or
metric (~15–25 words each); then a closing sentence that either interprets the pattern for
the review's argument (~25–35 words) or acknowledges a translational gap or limitation
(e.g., "However, most evidence derives from retrospective cohorts, and prospective validation
remains absent from the published literature" — this closing caveat is a characteristic
pattern in both benchmark papers). Body paragraphs in the benchmarks typically run 5–8
sentences. The paragraph must NOT open with a study name — that is annotated-bibliography
structure, not narrative synthesis.
Do not change meaning or citations.

Do not restructure paragraphs that already have variety. Only fix the flagged ones.
Save the revised text as manuscript/manuscript_v3.md.
```

### 步骤 6d — 引言与结论密度 pass

```
Read manuscript/manuscript_v3.md. Work on the Introduction and Conclusion ONLY.

INTRODUCTION:
- By the end of the second paragraph, at least one named framework, dataset, or study
  (with a patient count or metric) must appear. Move concrete evidence forward.
- Cut abstract scene-setting sentences that do not name anything specific.
- Maximum 2 sentences per paragraph that contain no citation.

CONCLUSION:
- The benchmark conclusion style allows general synthesis sentences without citation —
  whole paragraphs can be conceptual and field-level. The test is not "does this sentence
  cite a study?" but "could this sentence be anchored to a specific named study or
  technology from the review body, and is it being left vague out of laziness?" For each
  uncited general claim, check whether a specific exemplar exists in the reviewed literature;
  if one does, name it. If the claim is genuinely field-level synthesis with no single
  study to cite, it may stand. Limit uncited general claims to no more than 2–3 per
  conclusion paragraph.
- Keep the final forward-looking sentence; make it name the specific mechanism or
  technology that will unlock the next step, not a vague call to action.
Save the revised text as manuscript/manuscript_v4.md.
```

### 步骤 6e — 连贯性与一致性 pass

```
Read CLAUDE.md and manuscript/manuscript_v4.md.

1. Seams: check that transitions between sections read naturally; remove duplicate
   concept definitions and duplicate transition sentences (resolve <!-- DUPLICATE -->
   comments left by the assembly step).
2. Global consistency: terminology, abbreviations (defined on first use, used
   consistently thereafter), tense, and person throughout.
3. Narrative arc: ensure the whole text reads as a narrative synthesis, not a
   patchwork of paragraphs.
4. Concision: tighten overlong sentences by cutting redundant clauses. Do NOT delete
   whole paragraphs — tighten within sentences to preserve flow.
5. Citation coverage: flag any factual claim without a [FirstAuthorYear] placeholder.
6. Style: no bullets in main text; no "Recently, …" or "In recent years, …" openers.

7. NLR language: scan for any remaining systematic-review language — paper counts
   about the review's own collection ("89 studies in this corpus", "only 12 of the
   280 reviewed papers"), exhaustive-corpus assertions ("none of the 280 reviewed
   studies", "absent from all papers in the corpus"), meta-sentences about the
   review's scope ("this review synthesizes 280 studies"), and "in this corpus" /
   "the corpus" used as SLR boundary terms. Do NOT rewrite them here — this pass is
   a consistency check, not a rewrite pass. Insert a <!-- NLR-WARN: [brief reason] -->
   comment at each instance and list all flagged lines in the output summary. If any
   flags are present, the user must run /lit-deregister before proceeding to Phase 7.
   Numbers from *cited studies* (patient counts, AUC values) are not SLR language —
   do not flag them.

Do not summarize what changed. Output the complete revised text to manuscript/manuscript_v5.md.
```

### 步骤 6e-gate — 投稿门控检查

```
/lit-gate
```

技能检查字数、破折号频率、引文密度、摘要完整性、未填充占位符等指标，返回 PASS / WARN / FAIL 及具体修正建议。**没有 CRITICAL 级别问题时方可进入 Phase 7。**

---

## 阶段七：参考文献与 Word 导出

> **前提条件**：
>
> - Zotero 已打开，Better BibTeX 插件已安装
> - `references/library.bib` 已从 Better BibTeX 导出（含 citekey）
> - `manuscript/zotero.lua` 文件已就位
> - `/lit-gate` 返回无 CRITICAL 问题

### 步骤 7a — 替换引文占位符

```
/lit-cite
```

技能解析 `references/library.bib`，将所有 `[FirstAuthorYear]` 占位符匹配到 citekey，自动替换无歧义项，歧义项批量呈现供用户确认。保存 `manuscript_cited.md` 和 `cite_report_YYYYMMDD.md`。

### 步骤 7b — pandoc + zotero.lua 生成 Word 文档

在 `manuscript/` 目录下运行（**Zotero 必须保持打开状态**）：

```bash
pandoc -s --lua-filter=zotero.lua manuscript_cited.md -o manuscript.docx
```

> `zotero.lua` 过滤器会联系 Zotero 的 Better BibTeX JSON-RPC API，将live cittation 字段直接嵌入 docx 文件。

### 步骤 7c — 在 Word 中设置引文格式（手动操作）

1. 在 Word 中打开 `manuscript.docx`（需安装 Zotero Word 插件）
2. Zotero 工具栏 → 文档偏好设置 → 选择目标引文格式（如 Vancouver）→ 确定
3. 点击"添加/编辑参考文献列表"→ Zotero 自动插入格式化参考文献

---

## 写作规范要点

本工作流的行文规范由 CLAUDE.md 和各技能共同执行，核心要点：

**叙述性综述语言规范（NLR）：**
- 以论点驱动文献，不以文献驱动论点——段落以论断句开头，文献作为证据，结尾为解释或差距
- 禁止精确计数本综述文献库（"89 studies in this corpus"）
- 禁止穷举声明（"none of the 280 reviewed studies"）
- 禁止关于综述本身的元句子（"This review synthesizes 280 studies"）
- 用定性表述代替计数（"the dominant evidence base"、"relatively few published studies"）

**数字准确性：**
- 所有具体数字（AUC、C-index、样本量、数据集名称）必须可追溯至提取笔记
- 不得从记忆或模型推断生成数字

**格式规范：**
- 正文无项目符号列表
- 破折号（em-dash）全文不超过 8–10 个
- 缩写首次出现必须定义
- 不以研究名称开头段落（"Smith et al. found X. Jones et al. found Y."是注释书目结构，不是叙述综合）

---

## 工具技能速查

| 使用场景 | 命令 |
|---|---|
| 每次会话开始时查看项目状态 | `/lit-status` |
| 稿件数字准确性审查 | `/lit-audit` |
| 去除 AI 写作痕迹 | `/lit-deregister` |
| 逐节起草 | `/lit-draft [节号]` |
| 投稿前门控检查 | `/lit-gate` |
| 替换引文占位符 | `/lit-cite` |
| 记录本次会话 | 对 Claude Code 说："Append an entry to LOG.md for today's session. Include: phase, what was completed, what's next." |

---

## 许可证

MIT License — 欢迎在遵守许可证条款的前提下自由使用和改造本工作流。

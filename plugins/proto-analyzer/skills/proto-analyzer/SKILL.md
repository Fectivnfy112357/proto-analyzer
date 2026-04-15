---
name: proto-analyzer
description: >
  Analyzes prototype page URLs to generate backend development documentation:
  PRD (business requirements), System Design (architecture), and API Specification
  (integration contracts). Use whenever the user provides a prototype page link
  (Figma, Axure, online demo, deployed HTML) and wants backend documentation,
  API specs, data models, or any phrase like "analyze this prototype", "generate
  backend docs from this page", "extract requirements from this URL".
---

# Proto Analyzer

Transform a prototype URL into three backend development documents:
1. **PRD.md** — Business requirements, field rules, workflows
2. **SYSTEM-DESIGN.md** — Architecture, schema, module boundaries
3. **API-SPEC.md** — RESTful endpoints, request/response contracts

## Workflow

### Step 1: Prototype Exploration (Blocking)

Explore the prototype to understand its content.

**Multiple URLs**: The user may provide multiple prototype URLs (e.g., separate pages for list, form, detail). If multiple URLs are provided, process each URL through Steps 1.1-1.5 individually, then aggregate all results into a single `page-analysis.json`. For each URL, limit navigation depth to **2 levels from that URL**.

#### 1.1 Browser Tool Selection

Use whatever browser automation tool is available:
- **Playwright MCP** (`browser_navigate`, `browser_snapshot`, `browser_take_screenshot`, `browser_click`, `browser_type`)
- **Chrome Puppeteer MCP** (`puppeteer_navigate`, `puppeteer_screenshot`, `puppeteer_evaluate`, `puppeteer_click`, `puppeteer_fill`)
- **Other browser MCP tools** — adapt accordingly

**Strategy**: Start with the most capable tool. If one fails (e.g., page doesn't render, auth blocked), try another or ask the user for access.

**Fallback chain**: If browser automation fails entirely (page won't load, iframe blocked, anti-scraping prevents rendering):
1. Ask the user to upload screenshots of the prototype pages
2. If screenshots aren't available, ask the user to provide a field list directly (label, type, required flag, validation rules)
3. Proceed with Steps 2-6 using whatever information is available, flagging low-confidence areas

#### 1.2 Navigation & Authentication

- Navigate to the provided URL
- If a password/verification/login page appears, **ask the user** for credentials
- Take a screenshot for visual reference
- Identify the prototype platform type:
  - **Figma** — typically renders as a canvas with static layers
  - **Axure** — may have simulated interactions, notes panels, dynamic content
  - **CoDesign/蓝湖/墨刀** — Chinese prototyping platforms with their own viewers
  - **Deployed HTML** — real HTML/CSS/JS, may have real form elements
  - **Other** — describe what you see

#### 1.3 Prototype Platform Awareness

Different prototype platforms behave differently in a browser. Adjust your extraction approach accordingly:

| Platform | DOM Reality | Extraction Approach |
|----------|------------|--------------------|
| **Figma embed** | Static SVG/canvas layers; no real HTML form elements | Read visual labels, infer field structure from layout; look for design annotations |
| **Axure viewer** | Simulated interactions via JavaScript; notes in sidebar panels | Use both visual inspection AND notes panel (highest authority); interact with dynamic panels to reveal hidden states |
| **CoDesign/墨刀** | May have real HTML inputs mixed with static previews | Check if elements are actual `<input>` tags or just styled divs; prefer actual inputs |
| **Deployed HTML demo** | Real HTML/CSS/JS; actual form elements | Standard DOM inspection; extract from `<form>`, `<input>`, `<select>`, etc. |
| **无法渲染** (iframe blocked / auth wall / anti-scraping) | Empty page, login redirect, or blank iframe | Do NOT proceed with browser tools. Fall back to: (1) ask user for screenshots, (2) ask user for field list directly, (3) use provided URL metadata (page title, URL path segments) to make educated guesses with low confidence |

**Key principle**: When in doubt about whether a visual element represents a real field or mock data, **check the prototype's annotation/notes system first**, then ask the user for clarification.

#### 1.4 Page Type Classification

Before extracting any fields, classify each page:

| Page Type | Characteristics | Extraction Strategy |
|-----------|----------------|--------------------|
| **表单页** (Form) | Input boxes, textareas, selects, file uploads, editors with labels | Extract label-input pairs as backend fields |
| **列表页** (List) | Table/grid with column headers, filter bar, pagination | Extract column headers as list fields; extract filter items; do NOT extract individual row data values |
| **详情页** (Detail) | Read-only display of a record | Extract displayed fields; note whether there's an associated edit/create form |
| **导航/面板** (Dashboard) | Cards/links to other pages | Extract navigation structure only; typically no backend fields |
| **登录/注册** (Auth) | Auth forms | Extract auth fields; note auth method |

#### 1.5 Field Extraction: Core Decision Framework

The central challenge: **distinguishing field definitions from data instances and mock content**. Use this decision tree:

```
Is this element part of a form control or data display?
├── Form control (input, select, textarea, upload zone, editor)
│   └── Has a visible label nearby? → YES = real field, NO = investigate further
│
├── Data display (static text, table cell, card content)
│   ├── Is it inside a "list" or "table" row? → YES = data instance (extract structure, not values)
│   ├── Is it inside a "form" with other input fields? → YES = could be pre-filled value (check if editable)
│   └── Is it standalone display content? → Evaluate context (see 1.6)
│
└── Navigation/UI chrome (menus, breadcrumbs, buttons, logos)
    └── NOT a field
```

**Apply these filtering rules:**

**Rule 1: Label-Input Pairing (primary signal for form pages)**
A real form field has:
- A label (noun phrase, typically 2-8 characters in Chinese) adjacent to a control
- Often marked with `*` if required
- The control is interactive (or designed to appear interactive in the prototype)
- Examples: "课程名称" + text box, "考试时长" + number input, "课程图片" + upload zone

**Rule 2: Data Instance vs. Field Definition**
- **Field definition** describes a category of data: "考试名称", "手机号", "总课时"
- **Data instance** is a specific value: "2026年中级经济法VIP高效通关班", "182******83", "60"
- Extract the **field** (with its constraints), not the **value**
- When you see a list of repeated items (e.g., multiple questions, multiple products), extract the **item structure** (what fields each item has) but not the specific content of each item

**Rule 3: Context-Based Mock Data Detection**
A text block is likely mock/sample data (not a backend field) when:
- It contains complete, domain-specific business content (a full exam question, a complete user bio, a full product description)
- It is significantly longer than a typical form input (paragraphs of text inside what should be a single field)
- It appears inside a "sample" or "example" container, card, or panel
- It is repeated multiple times with similar structure (indicates data instances, not field definitions)
- It contains placeholder patterns (Lorem ipsum style, "正确答案内容", "示例文本")
- The label explicitly says "示例", "样例", "mock", "demo"
- It appears in a "preview" or "展示" zone that is separate from the input form

**Rule 4: Aggregated/Computed Displays**
Text that summarizes or aggregates other data is typically NOT a field:
- "共 X 条记录", "总计 X 分", "第 X/Y 页" — pagination/summary info
- "已选择 3 项" — selection counter
- These are UI metadata, not backend fields

**Rule 5: Designer Notes Authority**
If the prototype has a notes/annotations/specification system:
- Notes describing field rules (length limits, format requirements, data sources) are **authoritative**
- Field definitions in notes take priority over visual interpretation
- Notes may reveal fields that aren't visually obvious (server-side validations, hidden fields)

**Rule 6: When Ambiguous — Ask or Flag**
If you cannot confidently determine whether content is a field definition or mock data:
- Flag it with `confidence: "low"` in the output
- Include it in `excluded_items` with reason "uncertain — requires clarification"
- During Step 2 (Context Collection), present ambiguous items to the user

#### 1.6 Navigation Graph

- Identify all pages reachable from the current page (navigation trees, sidebars, menus, clickable cards)
- For each reachable page: classify its type, extract fields using the rules above
- Limit depth: do NOT follow links more than **2 levels deep** from the current URL being analyzed
- Deduplicate: skip pages with visually identical content (e.g., "编辑" vs "创建" may have the same fields)

#### 1.7 Output: page-analysis.json Schema

Produce a `page-analysis.json` with this EXACT structure:

```json
{
  "project_name": "string",
  "platform_type": "string — Figma/Axure/CoDesign/墨刀/deployed HTML/etc.",
  "prototype_url": "string",
  "total_pages": number,
  "pages": [
    {
      "page_name": "string",
      "page_type": "form | list | detail | dashboard | auth | other",
      "navigation_path": "string — e.g., 后台 > 课程管理 > 创建课程",
      "confidence": number,
      "sections": [
        {
          "section_name": "string",
          "fields": [
            {
              "label": "string",
              "field_key": "string — suggested snake_case key",
              "control_type": "text | number | textarea | select | radio | checkbox | date | file | rich_text | switch | custom",
              "required": boolean,
              "max_length": "number | null",
              "placeholder": "string | null",
              "validation_rules": ["string array"],
              "confidence": "high | medium | low",
              "notes": "string | null — designer annotations if available"
            }
          ],
          "actions": [
            {
              "label": "string",
              "type": "submit | reset | navigate | dialog_trigger | delete | other",
              "requires_confirmation": boolean,
              "target_page": "string | null"
            }
          ]
        }
      ],
      "excluded_items": [
        {
          "text": "string — brief description of excluded content",
          "reason": "string — why excluded (data_instance, mock_content, aggregated_display, uncertain, navigation_chrome, etc.)",
          "rule_applied": "string — which extraction rule triggered the exclusion"
        }
      ]
    }
  ],
  "navigation_tree": ["array of reachable page paths"],
  "confirmed_workflow": [
    {
      "page_name": "string",
      "entry_path": "string — navigation path to reach this page",
      "core_flow": ["string array — step-by-step flow: A → B → C"],
      "available_actions": ["string array — operations user can perform"],
      "post_action_behavior": [
        {
          "action": "string",
          "behavior": "string — navigate/dialog/refresh/etc."
        }
      ],
      "exception_branches": [
        {
          "scenario": "string — e.g., permission denied, network timeout",
          "handling": "string — system response"
        }
      ]
    }
  ],
  "auth_required": boolean,
  "analysis_notes": "string — observations about prototype completeness, ambiguities, or limitations"
}
```

**Key principles for the output:**
- `excluded_items` documents what was filtered and why — this is auditable and debuggable
- `confidence` on each field and page lets downstream agents know what to trust
- `excluded_items[].rule_applied` links each exclusion to a specific rule, making the reasoning transparent
- Never include raw mock data content in the `fields` arrays

### Step 2: Business Context Collection (Interactive)

Based on `page-analysis.json`, **ask the user to provide business context**. Do NOT infer business background from page content alone — the UI only shows what fields exist, not why they are needed.

Ask the user three questions:

```
请提供以下业务背景信息（如果某些项暂不确定，可以跳过或说"用你的推断"）：

**1. 业务背景与痛点：** 当前业务面临什么问题？为什么需要开发这个功能？
**2. 目标用户角色：** 谁会使用这个页面/功能？（如运营人员、C端用户、管理员等）
**3. 预期目标：** 这个功能上线后希望达到什么效果？

请逐一回答，或回复"先跳过"让我基于原型内容做初步推断。
```

If the user says "先跳过" or asks you to infer, draft the three items with a clear disclaimer that they are preliminary and need verification:

```
（以下为用户未提供时的初步推断，需要后续确认）

**业务背景与痛点：** [基于原型字段的推断]
**目标用户：** [基于页面结构的推断]
**预期目标：** [基于功能集的推断]
```

After the user provides or confirms the business context, present any **low-confidence fields** or **ambiguous items** from `excluded_items` that need user clarification:

```
以下字段/内容我拿不准，请确认：
- [模糊项 1] — 原因：...
- [模糊项 2] — 原因：...
```

Wait for user confirmation or edits before proceeding.

### Step 3: Business Workflow Confirmation (Interactive, Blocking)

基于 `page-analysis.json` 中的 `navigation_graph` 和 `actions`，**梳理出完整的业务流程图**，向用户展示并要求确认。这是生成文档前的最后一道闸门。

**呈现格式**：

```
以下是我基于原型梳理的业务流程，请逐项确认：

**页面：[页面名称]**
- 入口路径：[导航路径]
- 核心流程：[步骤A] → [步骤B] → [步骤C]
- 可执行操作：[操作1]、[操作2]、[操作3]
- 操作后的行为：[操作1]→跳转到XX页 / 弹出确认对话框 / 刷新当前列表
- 异常分支：[如权限不足时跳转 / 网络超时时提示]

---
（多个页面依次列出）
---

请确认以上流程是否准确。如有遗漏或错误，请指出需要调整的部分。
回复"确认"继续，或告诉我具体要修改的地方。
```

**关键规则**：
- **禁止跳过** — 此步骤不允许用户说"跳过"。如果用户不确认，不得进入文档生成阶段
- **基于实际提取内容** — 流程描述必须严格来自 `page-analysis.json` 中的 `navigation_tree`、`sections[].actions`、`pages[].page_type`，不可凭空捏造
- **标注推断项** — 如果某些操作的行为（如"弹出确认对话框"）无法从原型直接判断，标注为 `[推断]` 并提醒用户确认
- **异常流程必须列出** — 即使原型未展示异常场景（如网络超时、权限不足），也要基于常识列出可能的异常分支，标注为 `[推断—需确认]`

**用户确认后**：
- 将确认后的流程写入 `page-analysis.json` 的 `confirmed_workflow` 字段
- 如果用户提出修改，更新 `page-analysis.json` 后再次展示，直到用户确认

### Step 4: Tech Stack Selection (Interactive, Blocking)

Ask the user to choose a backend tech stack. If they haven't specified one, present options:

```
请选择后端技术栈：
1. Java + Spring Boot + MyBatis Plus
2. Python + FastAPI + SQLAlchemy
3. Go + Gin + GORM
4. Node.js + NestJS + TypeORM
5. 其他（请说明）
6. 框架无关的描述即可
```

**Do NOT proceed to document generation until tech stack is confirmed.**

### Step 5: Tech Stack Convention Discovery

Once tech stack is confirmed:

1. Check installed skills for matching tech stack domain knowledge (grep skill directories and descriptions for keywords like "spring", "fastapi", "gin", "nestjs", "django", "mybatis", "jpa", "gorm").
2. **If a matching skill exists:** read its relevant reference files to extract framework conventions, best practices, and patterns.
3. **If no matching skill:** use `mcp__context7__*` tools to query the framework's official documentation for latest conventions and best practices. Then use `WebSearch` to supplement with recent community guidance (e.g., GitHub issues, Stack Overflow, blog posts from the past 12 months).
4. **If user chose framework-agnostic:** skip this step; generate generic descriptions in SYSTEM-DESIGN.md.

### Step 6: Sequential Document Generation

**Only proceed after Step 1 is complete (`page-analysis.json` available) AND Steps 2-4 confirmed by user.**

Document generation follows a **sequential dependency** to ensure consistency:

```
Step 6a (Agent A: PRD)  ──completed──▶  Step 6b (Agent B: System Design + Agent C: API Spec, in parallel)
```

Each agent receives:
- `page-analysis.json` (complete field/interaction data)
- `confirmed_workflow` (user-confirmed business workflows from Step 3)
- User-confirmed context (business background, roles, goal)
- Tech stack conventions (from Step 5)

#### Step 6a: PRD Writer (Blocking — must complete first)

Spawn Agent A to generate `PRD.md`. This agent **must complete before** Step 6b begins, as both Agent B and Agent C will read the generated PRD.

Agent A must include:

- **Business background & pain point** — from user-provided context; never fabricate
- **Roles & usage scenarios** — who uses this page and in what situation
- **Core workflow diagram** — text-based flow (A → B → C) derived from `confirmed_workflow`; if not confirmed, derive from the navigation graph
- **Page module breakdown** — filter zone, data grid, form sections, action bar
- **Field & interaction rules** — every field from page-analysis with: field name, data type (in business terms: text/number/date), required flag, length limit, validation rules (format, range, business constraints)
- **Operation feedback** — what happens on each button click (confirm dialog, navigation, loading state, duplicate submission prevention)
- **Exception scenarios** — network timeout, permission denied, empty data placeholder behavior

**REMINDER:** No database terminology, no API paths, no JSON structures.

**CRITICAL:** Only include fields from `page-analysis.json` that are in the `fields` arrays. Cross-check `excluded_items` to ensure excluded content was not inadvertently included.

#### Step 6b: System Design + API Spec (Parallel — after PRD.md is written)

Once `PRD.md` is written and available, spawn Agent B and Agent C **in parallel**. Both agents receive the same inputs plus the completed `PRD.md`.

**Agent B: System Design Writer**

Generate `SYSTEM-DESIGN.md` using the template at `templates/system-design-template.md`. Must include:

- **Tech stack declaration** — framework, database, cache (from Step 5 conventions)
- **Module boundary planning** — routers → services → models layering
- **Schema design** — entities derived from PRD fields; relationships (one-to-many, etc.); core table structures with field name, SQL type, constraints, index suggestions (no raw SQL)
- **Directory structure** — recommended project layout following the tech stack conventions from Step 5

**REMINDER:** No API endpoint definitions, no request/response JSON examples. Reference PRD for business rules.

**Agent C: API Spec Writer**

Generate `API-SPEC.md` applying the tech stack conventions from Step 5. Must include:

- **Endpoint inventory** — all endpoints needed to support the prototype. Include both standard CRUD endpoints (list, detail, create, update, delete) AND **operation-type endpoints** identified from the prototype's action buttons (e.g., batch approval, status transition, import/export, file upload, custom business actions).
- **Per-endpoint definition**:
  - Path, HTTP method, brief description
  - Request: headers, query params, body JSON (mark required vs optional)
  - Response: success JSON structure, status codes, business error codes with messages
- **Framework-specific conventions**: apply rules from Step 5 (e.g. MyBatis Plus single-table CRUD inherits BaseMapper — no separate XML; FastAPI uses Pydantic schemas with dependency injection; etc.)
- **Naming conventions**: follow the tech stack's RESTful or framework-specific naming patterns

**Non-CRUD operations**: For actions that are not simple CRUD (batch operations, state transitions, file uploads, exports), define them as custom endpoints with clear request/response contracts. Example: `POST /api/v1/orders/{id}/approve` for order approval, `POST /api/v1/users/import` for file upload with `multipart/form-data`.

**REMINDER:** No database table definitions, no business background. Reference SYSTEM-DESIGN for entity types, PRD for validation rules.

### Step 7: Verification (Three-Pass)

After all three documents are generated, run verification in three sequential passes:

#### Step 7a: Prototype Backtrack Checker (NEW — Ground Truth Verification)

Spawn a verification sub-agent that performs **independent verification against the original prototype**, not just `page-analysis.json`:

1. **Re-visit the prototype URL** — take a fresh screenshot/snapshot of each analyzed page
2. **Visual field sweep** — scan the rendered prototype for any visible fields, labels, buttons, or form controls that exist on the page
3. **Three-way cross-reference**:
   - For each visual element on the prototype: is it captured in `page-analysis.json`? Is it in the generated documents?
   - For each field in `page-analysis.json`: does it correspond to a real visual element on the prototype?
   - For each field in the documents: does it trace back to both `page-analysis.json` AND a visible prototype element?
4. **False positive detection** — identify fields in `page-analysis.json` that don't actually exist on the prototype (over-extraction, mock data misclassified as fields)
5. **False negative detection** — identify fields visible on the prototype that were missed by `page-analysis.json` (under-extraction, real fields misclassified as mock data)
6. **Excluded items audit** — re-examine items in `excluded_items` arrays:
   - Was the exclusion rule correctly applied? Could any excluded item actually be a real field?
   - Flag items where the exclusion seems overly aggressive
7. Produces a backtrack report with:
   - **Over-extraction list**: fields in `page-analysis.json` that shouldn't be there
   - **Under-extraction list**: real prototype fields missing from `page-analysis.json`
   - **Exclusion review**: excluded items that may have been wrongly filtered
   - **Confidence adjustment**: revised page-level confidence scores based on findings

**Key principle**: The prototype is the ground truth. Both `page-analysis.json` and the generated documents are derived artifacts that may contain errors. Always verify against the source.

#### Step 7b: Coverage Checker

Spawn a verification sub-agent that:

1. Reads all three generated documents, `page-analysis.json`, AND the Step 7a backtrack report
2. **Apply corrections from Step 7a**: if the backtrack checker found over-extraction or under-extraction, use the corrected field list (not the original `page-analysis.json`) as the baseline for coverage checks
3. Cross-references every field mentioned in PRD/API-SPEC against the corrected field list
4. Checks that no field from the prototype is missing in the API contract
5. Validates that API endpoints cover all interactions identified in the navigation graph
6. Ensures SYSTEM-DESIGN entities map to the corrected field list
7. **Checks for excluded-item leakage** — verifies nothing from `excluded_items` arrays (that were correctly excluded per Step 7a) appears as a field in any document
8. Produces a coverage report with:
   - Coverage matrix: corrected prototype fields vs document mentions (covered / missing)
   - Gap list: any field, interaction, or page zone that was not captured
   - **Exclusion integrity check**: confirms no excluded content leaked into documents

#### Step 7c: Boundary Checker

Spawn a second verification sub-agent (or continue with the same agent) that:

1. Reads all three generated documents
2. **Checks document boundary compliance** — verifies no overlap between the three documents:
   - PRD must NOT contain database table names, SQL types, API paths, JSON structures
   - SYSTEM-DESIGN must NOT contain API endpoint definitions, request/response JSON examples, business rule descriptions
   - API-SPEC must NOT contain database table definitions, ORM models, business background descriptions
3. Produces a boundary compliance report with:
   - Any detected overlaps or boundary violations
   - Confidence score per document

#### Combined Output

Merge all three reports into a single `VERIFICATION.md` with:
- **Prototype backtrack findings** (over-extraction, under-extraction, exclusion review)
- Coverage matrix
- Gap list
- Exclusion integrity check
- Document boundary compliance check
- Confidence score per document (adjusted based on backtrack findings)

**If Step 7a finds significant errors in `page-analysis.json`** (more than 2 false positives or false negatives):
1. **Stop document generation** — do NOT proceed with retry on existing documents
2. **Update `page-analysis.json`** with the corrected field list
3. **Re-spawn all three document agents** (PRD, System Design, API Spec) with the corrected analysis
4. **Retry at most once.** If significant errors remain after one retry, output the remaining discrepancy list for manual review.

**If Step 7a finds minor errors** (1-2 discrepancies):
1. Patch the affected documents directly via Edit
2. Proceed to 7b and 7c

If 7b or 7c finds gaps or boundary violations (after 7a has stabilized the field list), re-spawn the relevant document agent with the gap report and regenerate only the affected sections. **Retry at most once per document.** If gaps remain after one retry, output the remaining gap list for manual review.

## Output Location

All documents are written to a `docs/` directory at the project root. If `docs/` does not exist, create it. If a file with the same name already exists, append a timestamp suffix (e.g., `PRD_20260415_1430.md`).

```
docs/
├── PRD.md
├── SYSTEM-DESIGN.md
├── API-SPEC.md
└── VERIFICATION.md
```

## Templates

- PRD structure: see `templates/prd-template.md`
- System Design structure: see `templates/system-design-template.md`
- API Spec structure: see `templates/api-spec-template.md`

**Agent Lab v1 交互稿：生图提示词记录**

日期：2026-10-07。使用内置 image_gen 工具；没有使用 CLI/API fallback。三张最终图片采用演示数据，均为静态交互概念稿。

主工作台先生成，再修订运行中配置的只读状态。后两页以修订主工作台为视觉参考。下列为实际提示词。

**主工作台初稿**

```text
Use case: ui-mockup
Asset type: first high-fidelity interaction concept for a serious desktop Agent research and evaluation workbench; one straight-on full application screenshot, landscape 16:10, very high resolution. This is a proposed interface, not a real deployed app.
Primary request: Design the main workspace of "Agent Lab", a Chinese-language application where a human asks a research question, either Codex or a built-in agent hosts the research, experiments run through interchangeable model API profiles, and every conclusion links to inspectable evidence. A CLI is connected to the same workspace. Show a realistic IN-PROGRESS experiment with clear controls and a polished coherent interface.
Visual style: calm professional research software, warm very light gray background, white panels, graphite typography, thin cool-gray separators, restrained teal accent and a few amber status indicators, generous but efficient spacing, small sharp monochrome icons, legible Chinese sans-serif typography. No neon, no glassmorphism, no gradient hero, no decorative illustrations, no laptop device frame. Use a full-width flat application canvas, crisp readable text, careful grid alignment.
Layout:
- Top bar: small wordmark "Agent Lab"; breadcrumb "记忆研究 / 更新与遗忘"; on right "CLI 已连接" with small green dot, "主持者：Codex" as a dropdown, a subtle "切换主持者" control. Small badge "演示数据".
- Narrow left navigation around 14% width. Items "研究", "实验", "数据集", "策略库", "API 连接". Research selected. Below a section "最近研究" with "更新与遗忘" selected, "上下文压缩", "工具失败恢复". Bottom small "本地工作区".
- Middle research conversation column around 32% width. Header "研究对话" with an understated new-message icon. Human message: "比较普通摘要和结构化记忆，哪个更少使用过期信息？". Agent message labelled "Codex · 研究主持": "先固定模型和场景，只改变记忆策略。". Show one elegant structured proposal card titled "实验方案 v3" with three short rows "3 种策略", "20 个场景 × 3 次重复", "固定模型 · 固定判分". Below a subdued small status row "实验运行中 · 已完成 126 / 180". One clickable evidence-style message "已发现 4 个过期记忆案例" with tag "查看证据 →". Bottom pinned conversation composer placeholder "追问结果，或提出下一轮实验…" and compact send button.
- Large right work area around 54% width. Header "实验 EXP-024" with controls "暂停", outlined "复制实验". Tab bar "方案" "运行" (selected) "比较" "证据".
- At top of this area a compact two-column settings summary: "实验 API：OpenAI / 模型 A" dropdown and "判分：规则校验"; a small button "编辑实验矩阵". Budget line "已用 ¥18.40 / ¥50.00" and thin progress track.
- Main substantial visual is a well-designed experiment matrix table, not KPI cards. Column headers "策略" "完成" "成功" "过期引用" "状态". Three rows: "最近消息" "42 / 60" "29" "8" "运行中"; "普通摘要" "42 / 60" "33" "5" "运行中"; "结构化记忆" "42 / 60" "37" "1" "运行中". Use teal for completed amount but no misleading winner declaration before completion. Clear subtle line below: "阶段结果 · 尚未完成全部重复".
- Under table, a split region: left 60% a compact chronological activity list titled "运行记录", with rows "14:32 场景 S-017 已完成", "14:32 记忆已更新", "14:31 场景 S-008 重试 1 次"; right 40% a small outlined card titled "待复核" with item "S-017 · 旧禁令被引用", item "S-008 · 工具超时". Clearly clickable cases with arrows.
- Bottom status bar across workspace "同一实验可由对话、界面或 CLI 继续操作".
Interaction priorities: host selector distinct from experiment API selector; conversation remains visible beside experiment artifacts; experiment run state is clear; case-level evidence navigation prominent. No code in the main user flow, no secret API keys. Treat all numbers as demo examples; keep the "演示数据" badge legible.
Text constraints: render the quoted Chinese labels exactly and clearly wherever possible, not lorem ipsum. Keep labels concise, large enough to read. Output only the polished UI mockup image.
```

**主工作台修订**

```text
Use case: ui-mockup edit. Edit this Agent Lab application screenshot with ONLY two small changes, preserving the exact layout, all panels, typography, palette, dimensions, and all other text and values.
1) In the right experiment configuration strip, change the button text "编辑实验矩阵" to "查看实验矩阵".
2) The active running experiment's "OpenAI / 模型 A" field and "规则校验" field must look read-only: replace their dropdown chevrons with subtle small lock icons. Keep the separate top bar "主持者：Codex" host dropdown interactive, unchanged.
These changes communicate that a running experiment's configuration is immutable. Do not alter any other area. Keep the "演示数据" badge. Output the revised high resolution image.
```

**证据对照页**

```text
Use case: ui-mockup
Asset type: second screen in the same first interaction-design version for Agent Lab.
Input image role: visual style and app chrome reference. Preserve the reference's exact calm light theme, wordmark, left navigation, teal accent, Chinese typography, header proportions and icon style. Replace the main content with the case evidence comparison screen described below. One high-resolution straight-on desktop application screenshot, same landscape aspect ratio. No device frame.
Primary request: Show the screen reached when clicking a failed scenario in the research workspace. It must let a human inspect why ordinary summary memory used an outdated policy, while structured memory correctly applied an update. All content is demonstration data.
Top bar unchanged: "Agent Lab", breadcrumb "记忆研究 / 更新与遗忘", "CLI 已连接", "主持者：Codex" dropdown, "切换主持者", "演示数据".
Left app sidebar unchanged: "研究", "实验", "数据集", "策略库", "API 连接", recent studies with "更新与遗忘" selected; bottom "本地工作区".
Content layout after sidebar:
- Top spanning header: a modest back link "← 返回研究"; prominent "S-017 · 禁令更新"; metadata "EXP-024 · 第 2 次重复 · 同场景 / 同模型"; top-right buttons "导出证据" and teal "在研究中讨论".
- Under header a slim navigation row "原始记录" "记忆变化" "检索" "上下文" selected "动作与结果"; and a right-aligned toggle "只看差异" turned on.
- A narrow scenario list at left of content, about 18% total canvas width: header "场景" and a search field. Rows: "S-017 禁令更新" selected, "S-008 工具超时", "S-012 条约到期", "S-019 信息不足". S017 has small amber dot.
- The central area, about 43% canvas width, contains two equal-width side-by-side comparison panels. Heading over both "模型实际看到的内容". Small chip "OpenAI / 模型 A".
Left panel: header "普通摘要" and amber status pill "失败". Card text "城邦 B 禁止粮食出口。" with restrained amber highlight, source chip "E-003". Beneath a clear issue line "缺失更新 E-017". At the bottom a section "实际动作" and text "拒绝交易", with muted note "使用了已失效的禁令".
Right panel: header "结构化记忆" and teal status pill "通过". Show a tidy structured fact card, plain-language labels instead of code: "当前政策：允许出口", "生效回合：17", "替代旧记录：E-003", "依据：E-017". Teal highlight on current fact. At bottom "实际动作" and "发起交易", note "遵循当前有效政策".
- A right evidence column about 23% canvas width. Header "判分依据", small badge "规则校验 v2". A concise condition "应使用回合 20 的有效政策". Below two compact result rows "普通摘要  未满足" with amber x and "结构化记忆  满足" with teal check.
- Below it a clean timeline card titled "事件时间线": "回合 03" then "E-003 · 禁止出口"; "回合 17" then "E-017 · 禁令解除"; "回合 20" then "收到粮食交易请求". Lines and event chips link the timeline to comparison cards.
- Below central comparison, a horizontal soft-gray note "差异定位：更新已存在于原始记录，但未进入普通摘要的最终上下文。"
- Pinned bottom composer across content: small quoted-context chip "已引用 S-017", placeholder "基于这个案例，继续追问或设计下一轮实验…", teal send arrow.
Important: This is an evidence review interface, not a generic dashboard. Make the two context panels and source event identifiers prominent, show provenance and chronological validity. Do not add charts, fake console logs, API secrets, or unsupported statistical claims. Keep clear alignment and ample readability. Render the quoted Chinese text accurately. Keep the "演示数据" badge visible.
```

**实验矩阵配置页**

```text
Use case: ui-mockup
Asset type: third screen in the SAME Agent Lab interaction concept, showing experiment matrix configuration BEFORE launch.
Input image role: style and app chrome reference. Match the same wordmark, pale background, teal accent, graphite Chinese typography, sidebar, header and precise UI spacing. One straight-on flat desktop software screenshot, landscape 16:10, high resolution. No device frame or presentation captions.
Primary request: A Chinese-language research workbench with a conversation alongside an editable experiment matrix. Clearly show that a research host can be Codex or a built-in Agent, independently of the three model API profiles used for experiments. All content is demo data.
Header same as reference: "Agent Lab", "记忆研究 / 更新与遗忘", "CLI 已连接", "主持者：Codex", "演示数据". Show the HOST dropdown OPEN under its trigger but only occupying a small corner at upper right, not obscuring the main matrix. Dropdown options "Codex" selected with small check, "内置 Agent", "手动主持"; footer hint "切换后保留研究与实验记录". No other dropdown is open.
Left sidebar same: "研究" selected, "实验", "数据集", "策略库", "API 连接"; recent studies selected "更新与遗忘"; "本地工作区" at bottom.
Middle 30% conversation panel headed "研究对话": human message "把三种记忆策略分别跑在三个 API 上。"; agent labelled "Codex · 研究主持" responds "已建立 9 组组合。每组 60 次运行，共 540 次。"; compact linked proposal card "新实验草案" then "主持者与被测模型独立配置"; lower evidence note "沿用数据集 v2 · 固定规则判分". Bottom composer "补充假设，或调整实验设置…".
Large right content 55% width with heading "实验矩阵" and a small pill "草案". Tabs "方案" selected, "运行", "比较", "证据". Content start with segmented comparison mode "比较策略", "比较模型", "交叉比较" selected.
Row titled "实验 API" with three checked profile chips, wrapped if necessary: "OpenAI / 模型 A", "Anthropic / 模型 B", "自定义接口 / 模型 C"; small link "+ 添加连接". These are descriptive configured profiles, no real secret keys and no invented model release names.
Main large matrix table: top-left header "记忆策略"; three column headers "模型 A" "模型 B" "模型 C", smaller provider label under each. Rows "最近消息", "普通摘要", "结构化记忆". Every one of the nine cells has a subtle teal enabled check and small text "60 次". Well aligned, beautiful soft dividers.
Below table two rows of compact fields: "数据集" value "更新与遗忘 v2 · 20 场景"; "重复次数" value "3"; "判分方式" value "固定规则 v2"; "预算上限" value "¥100.00"; "最大并发" value "3". Enough white space and legible field labels.
Bottom summary bar: "9 组组合 · 540 次运行". Small neutral status "启动前检查 API 能力与预算". Prominent teal action "校验并启动", secondary "保存草案". Fine note "启动后冻结配置，修改将创建新实验。"
Interaction priorities: the host dropdown with Codex and built-in Agent is physically and conceptually distinct from the three selected experiment API profiles; matrix dimensions are correct (3 strategies × 3 models × 20 scenarios × 3 repeats = 540); this draft is editable; the shared conversation and CLI connection remain visible. No irrelevant metrics, no fabricated real performance results. Keep "演示数据" readable.
Text: Render the Chinese labels accurately, concise and readable. A coherent polished research tool, not a crowded marketing dashboard.
```

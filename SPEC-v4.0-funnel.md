     1|# SPEC-v4.0-funnel — Maple Hollow Home 多类目科学选品漏斗系统
     2|
     3|> 本文档是 SPEC-v3.1（评分/利润/状态机骨架）的**上层补充规范**。
     4|> v3.1 定义"一个关键词怎么打分、一个产品怎么算利润/走状态机"；
     5|> 本 v4.0 定义"怎么从三大类目里，用科学漏斗逐层收窄，找出值得做的长尾产品"。
     6|> 两者互补：v4.0 的 L4/L5 直接复用 v3.1 的利润计算与决策/状态机，不重复定义。
     7|
     8|---
     9|
    10|## 修订记录
    11|
    12|| 版本 | 日期 | 核心变更 |
    13||------|------|---------|
    14|| v4.0 | 2026-05-31 | ①引入"先导/现状/滞后"三时相数据模型 ②6 层漏斗（L0-L5）③多类目×多子分类隔离 ④候选词池与 rising 裂变回灌 ⑤每个节点写成可独立验收的"模块规格卡" |
    15|
    16|**冻结声明**：本版本可直接交付任意能力的 LLM 逐卡实现。每张模块规格卡自给自足（输入 schema + 清洗伪代码 + 输出 schema + API 契约 + 真实数据验收示例 + 人工验收步骤）。后续仅做 bug 修正，不在本版无限加功能。
    17|
    18|【实现优先级】：
    19|- P0（最小可用闭环 — 必须一步到位）：M1 类目体系 → M2 候选池 → N0 L0先导(Google rising + N0.1信号一致性) → N1 L1赛道(滞后 A/B) → M3 时相引擎 → N2 推荐①一级词
    20|  ⚠ N0 不是"增强"，是 M3+N2 正常工作的前提。拆到 P1 = P0 输出垃圾(mug 排第一)。
    21|- P1（长尾+能力匹配）：N3 L2 长尾分析 + L3.1 能力画像匹配(能力匹配)
    22|- P2（落地闭环+反馈，复用 v3.1）：N4 L3找货 → N5 L4利润 → N6 L5决策(N6.1 RESEARCH诊断) + feedback fields
    23|- P3（横向复制）：把 home_living 验证过的漏斗复制到 wedding / jewelry（含 滞后 策略不串）
    24|
    25|---
    26|
    27|## 0. 给实现 LLM 的总规则（必读，违反即跑偏）
    28|
    29|R1. **类目隔离铁律**：所有信号计算、归一化（min-max）、推荐排序，**必须**在同一 `(erank_category, subcategory)` 分组内进行。**禁止**跨大类目把关键词放进同一个池子归一化或排序。跨类目只能并列展示，不可同池比较。违反此条 = 整个系统输出无意义。
    30|
    31|R2. **时相标签强制**：每个关键词候选必须带三类信号分（leading / current / lagging），缺失的用明确空值（null / 0.0 + confidence=0），**禁止**用某一类信号的值假装另一类。
    32|
    33|R3. **AI 边界**（继承 v3.1 §2.2）：除非某张卡显式标注"本节点用 LLM"，否则该节点 100% 规则驱动。LLM 不做评分、不做最终筛选、不做决策。标注"用 LLM"的节点，LLM 只产候选+理由，规则层做硬门槛过滤。
    34|
    35|R4. **不静默降级**：外部依赖（Google Trends / Pinterest / Etsy API / LLM）不可用时，对应端点返回 HTTP 503 并在 body 说明缺哪个依赖。**禁止**返回降级/猜测结果冒充真实数据。
    36|
    37|R5. **数据真实性分层**（继承 v3.1 §2.1）：L1 真实(confidence=1.0) / L2 估算(0.5，UI 灰色+~前缀) / L3 缺失(0.0，UI 显示 "--")。
    38|
    39|R6. **金额单位后缀**：所有金额字段必须带 `_rmb` 或 `_usd` 后缀。所有时间字段 ISO 8601 字符串。所有数据库里的 JSON 字段在应用层 序列化/反序列化。
    40|
    41|R7. **统一 API 响应格式**（继承 AGENTS.md）：
    42|```json
    43|{"success": true, "data": {...}, "error": null, "meta": {"timestamp":"...","version":"v4.0"}}
    44|```
    45|错误：`{"success": false, "data": null, "error": {"code":"...","message":"...","field":"..."}}`
    46|
    47|R8. **每张卡 = 一个可独立验收单元**。实现一张卡后，跑该卡的"人工验收步骤"，对拍"验收示例"的期望输出，通过才进下一张。
    48|
    49|R9. **文件命名小写**（继承 AGENTS.md）：`phase_engine.py` 不是 `PhaseEngine.py`。所有 public 函数有中文 docstring + 类型标注。
    50|
    51|R10. **复用优先**：L4 利润 = 直接调 v3.1 `/api/profit/calculate`；L5 决策/状态机 = 直接调 v3.1 `/api/decisions` 和 `/api/products/{id}/transition`。本 SPEC 不重新定义这些，只定义它们的输入怎么从漏斗上游来。
    52|
    53|---
    54|
    55|## 1. 系统全景：6 层漏斗
    56|
    57|```
    58|候选词池 (candidate pool, 按 类目×子分类 隔离)
    59|   │  注入①: eRank 4表现状热词(冷启动唯一种子)
    60|   │  注入②: L0 rising 裂变出的新词(回灌)
    61|   │  注入③: 人工录入(Pinterest/逛Etsy发现)
    62|   ▼
    63|[N0 / L0 先导层]  → 每个候选词附 先导 分(Google rising + Pinterest)
    64|   ▼
    65|[N1 / L1 赛道层]  → 每个候选词附 现状 4维分(eRank) + 滞后 分(listing销售集中度+标题同质化)
    66|   ▼
    67|[M3 时相引擎]     → 综合 leading/current/lagging → 生命周期阶段标签
    68|   ▼
    69|[N2 推荐①]        → 输出 3-5 个一级关键词(按时相阶段排序) ──┐ 人在环:
    70|   ▼                                                    │ 你拿一级词去
    71|[N3 / L2 长尾层]  ← 你上传 eRank Keyword Tool 的 Top    │ eRank 搜词下载
    72|   ▼                                                    │ Top Listings CSV
    73|[N3 推荐②]        → 输出 5-10 组长尾关键词(机会分排序) ──┐ 人在环:
    74|   ▼                                                    │ 你拿长尾词
    75|[N4 / L3 找货层]  ← 你录入 1688 成本(suppliers)          │ 去1688找货
    76|   ▼
    77|[N5 / L4 利润层]  → 调 v3.1 /api/profit/calculate → 毛利率，回填 keywords.margin
    78|   ▼
    79|[N6 / L5 决策层]  → 汇总全信号信号卡 → 你拍板 BUY/RESEARCH/IGNORE (调 v3.1)
    80|```
    81|
    82|每层的"职责唯一性"自检（科学漏斗判据：每层必须能淘汰上层放进来的候选）：
    83|
    84|| 层 | 回答的唯一问题 | 收窄维度 | 主数据源 | 时相 |
    85||----|---------------|---------|---------|------|
    86|| L0 | 这方向未来会不会火 | 时间(先导) | Google rising, Pinterest | 先导 |
    87|| L1 | 这方向现在值不值得进 | 类目热度 | eRank 4表 | 现状 + 滞后 |
    88|| L2 | 具体哪个长尾词能切进去 | 竞争可入性 | eRank Keyword Tool CSV | 现状(竞争) |
    89|| L3 | 这词对应啥货、成本多少 | 供给可得性 | 1688(手工/OpenCLI) | — |
    90|| L4 | 这货能不能赚钱 | 经济性 | v3.1 利润引擎 | — |
    91|| L5 | 买不买 | 风险/人判断 | 全信号汇总 | — |
    92|
    93|---
    94|
    95|## 2. 数据模型总览（与现有表的关系）
    96|
    97|现有表（v3.1 已建，本 SPEC 复用/扩展）：
    98|- `keywords` — 候选词主表。**v4 扩展**：加 `subcategory`、`source_origin`、`phase_stage` 字段（见 M1/M3）。
    99|- `trend_signals` — 先导/趋势信号。**v4 复用**：source ∈ {google_rising, google_momentum, pinterest}，已有字段够用。
   100|- `erank_listings` — Top Listings。**v4 复用**：L2 长尾分析 + 滞后 数据源。
   101|- `most_clicked_shops` — 头部店铺。**v4不再作为滞后源**(CSV无百分比)，保留供参考。
   102|- `top_subcategories` — 子分类热度。**v4 复用**：子分类种子。
   103|- `suppliers` — 1688 成本。**v4 复用**：L3 输入。
   104|
   105|v4 新增表：
   106|- `candidate_pool_log` — 候选词来源审计（记录每个词从哪注入、何时裂变产生）。
   107|- `phase_snapshot` — 每月每词的时相快照（leading/current/lagging 三分 + 阶段标签），供历史回看。
   108|- **反馈闭环字段**（扩展现有 products / erank_listings 表，不建新表）：
   109|
   110|v4 对 products 表的反馈闭环扩展 🟡P2：
   111|> 业务盲区：没有"系统预测 vs 实际表现"的结构化记录，阈值校准是玄学。
   112|```
   113|ALTER TABLE products ADD COLUMN predicted_opportunity_score REAL;  -- 上架前系统给的机会分
   114|ALTER TABLE products ADD COLUMN actual_views_30d   INTEGER;        -- 上架30天真实浏览
   115|ALTER TABLE products ADD COLUMN actual_sales_30d   INTEGER;        -- 上架30天真实销量
   ALTER TABLE products ADD COLUMN prediction_delta   REAL;           -- actual_sales_30d - predicted (自动算, 负=高估, 正=低估)
   ALTER TABLE products ADD COLUMN deviation_reason   TEXT;           -- 人工标注枚举: pricing/tagging/季节性/quality/luck/能力不匹配/unknown
   118|```
   119|校准回路跑法（有了这组数据才能科学）：
   120|```
   121|1. 每月拉上架上满30天的产品: GROUP BY 偏差原因 统计
   122|2. "偏差原因=能力不匹配 占比高" → 能力画像匹配阈值需调
   123|3. "偏差原因=季节性 占比高" → 伪信号 过滤规则需扩
   124|4. "埋伏期 词 预测偏差 均值 -15" → HI 太低(过松), 上调
   125|5. 把偏差原因分布反馈给 SPEC 设计者（你），而不是自动改阈值
   126|```
   127|
   128|详细字段见对应模块规格卡。
   129|
   130|---
   131|
   132|## 模块规格卡 · 基础设施层
   133|
   134|> 每张卡格式固定：【目的】【输入 schema】【清洗/加工伪代码】【输出 schema】【API 契约】【验收示例(真实数据)】【人工验收步骤】【依赖】。
   135|> 弱 LLM：严格按卡实现，不要自由发挥。实现完一张，跑"人工验收步骤"，对拍"验收示例"，通过再下一张。
   136|
   137|---
   138|
   139|### M1 — 类目体系（多类目 × 子分类隔离）
   140|
   141|【目的】
   142|建立"大类目 → 子分类"两级体系，让所有下游计算能按 (erank_category, subcategory) 隔离。这是 R1 铁律的数据基础。
   143|
   144|【类目枚举（硬编码常量，禁止 LLM 自由扩展）】
   145|```python
   146|# app/constants/categories.py
   147|CATEGORY_TREE = {
   148|    "home_living": ["wall_decor", "desk_decor"],          # 用户当前主战场
   149|    "wedding":     ["wedding_signs", "wedding_favors", "table_decor"],
   150|    "jewelry":     ["necklaces", "earrings", "rings"],
   151|}
   152|# erank_category 取 CATEGORY_TREE 的 key
   153|# subcategory   取对应 value 列表之一，或 "unassigned"(未归类)
   154|```
   155|
   156|【输入 schema】无（这是常量 + 表结构扩展）
   157|
   158|【表结构扩展 — keywords 表加 3 字段】
   159|```sql
   160|ALTER TABLE keywords ADD COLUMN subcategory   TEXT DEFAULT 'unassigned';
   161|ALTER TABLE keywords ADD COLUMN source_origin TEXT DEFAULT 'erank';   -- erank / rising_fission / manual
   162|ALTER TABLE keywords ADD COLUMN phase_stage   TEXT DEFAULT NULL;       -- 由 M3 写入
   163|```
   164|
   165|【子分类归类规则（清洗伪代码）】
   166|```
   167|def assign_subcategory(keyword_text, erank_category):
   168|    """把一个关键词归到子分类。规则优先，匹配不上则 unassigned。"""
   169|    rules = SUBCATEGORY_KEYWORD_RULES[erank_category]   # 词典见下
   170|    for subcat, patterns in rules.items():
   171|        if any(re.search(p, keyword_text.lower()) for p in patterns):
   172|            return subcat
   173|    return "unassigned"
   174|
   175|SUBCATEGORY_KEYWORD_RULES = {
   176|  "home_living": {
   177|     "wall_decor": [r"\bwall\b", r"\bposter\b", r"\bprint\b", r"\bmirror\b", r"\bart\b", r"\btapestry\b"],
   178|     "desk_decor": [r"\bdesk\b", r"\borganizer\b", r"\btray\b", r"\bstand\b", r"\bpen\b"],
   179|  },
   180|  # wedding / jewelry 同构，P4 阶段补
   181|}
   182|```
   183|
   184|【输出 schema】keywords 行新增三字段被正确填充。
   185|
   186|【API 契约】
   187|```
   188|GET /api/categories
   189|→ data: {"tree": CATEGORY_TREE}
   190|
   191|POST /api/categories/reassign?month=2026-05&erank_category=home_living
   192|→ 对该类目该月所有 keywords 重跑 assign_subcategory，回填 subcategory
   193|→ data: {"updated": <int>, "by_subcategory": {"wall_decor": 8, "desk_decor": 2, "unassigned": 5}}
   194|```
   195|
   196|【验收示例（真实数据 2026-05 home_living）】
   197|```
   198|输入：keywords 表里 "wall art"(home_living), "home decor", "furniture", "halloween"
   199|调用：POST /api/categories/reassign?month=2026-05&erank_category=home_living
   200|期望输出（assign_subcategory 结果）：
   201|  "wall art"   → wall_decor   (命中 \bart\b 和 \bwall\b)
   202|  "home decor" → unassigned   (无 desk/wall 具体词)
   203|  "furniture"  → unassigned
   204|  "halloween"  → unassigned
   205|  by_subcategory 至少 wall_decor>=1
   206|```
   207|
   208|【人工验收步骤】
   209|1. `GET /api/categories` 返回三大类目树。
   210|2. `POST /api/categories/reassign?...home_living`，检查 "wall art" 的 subcategory 变成 "wall_decor"。
   211|3. 前端 keywords 表新增 subcategory 列，"wall art" 行显示 wall_decor。
   212|
   213|【依赖】无。这是地基，最先实现。
   214|
   215|---
   216|
   217|### M2 — 候选词池（candidate pool + 三来源注入 + rising 裂变回灌）
   218|
   219|【目的】
   220|管理"候选一级词"的来源与生命周期。解决先有鸡先有蛋问题：冷启动种子来自 eRank，运转后由 rising 裂变和人工扩充。
   221|
   222|【三个注入口（source_origin 字段区分）】
   223|```
   224|注入① erank          : eRank Category Report 导入时自动入池(冷启动唯一种子)
   225|注入② rising_fission : N0 L0 的 Google rising 吐出的、池中不存在的新词，回灌入池
   226|注入③ manual         : 用户在前端手工添加(Pinterest/逛Etsy发现)
   227|```
   228|
   229|【输入 schema（注入②裂变回灌）】
   230|```json
   231|{
   232|  "parent_keyword": "wall art",        // 裂变来源词
   233|  "new_keyword": "boho arch mirror",   // rising 吐出的新词
   234|  "erank_category": "home_living",     // 继承父词
   235|  "rising_value": "Breakout",          // 或数值如 450
   236|  "month": "2026-05"
   237|}
   238|```
   239|
   240|【裂变回灌伪代码】
   241|```
   242|def fission_inject(parent_keyword, new_keyword, erank_category, rising_value, month):
   243|    """rising 新词回灌候选池。已存在则跳过(不重复)。"""
   244|    if exists(keywords, keyword=new_keyword, month=month):
   245|        return {"injected": False, "reason": "already_in_pool"}
   246|    subcat = assign_subcategory(new_keyword, erank_category)   # 复用 M1
   247|    insert(keywords, keyword=new_keyword, erank_category=erank_category,
   248|           subcategory=subcat, source_origin="rising_fission",
   249|           month=month, created_at=now_iso())
   250|    insert(candidate_pool_log, keyword=new_keyword, origin="rising_fission",
   251|           parent=parent_keyword, rising_value=str(rising_value), month=month, ts=now_iso())
   252|    return {"injected": True, "subcategory": subcat}
   253|```
   254|
   255|【新增表 candidate_pool_log】
   256|```sql
   257|CREATE TABLE candidate_pool_log (
   258|  id INTEGER PRIMARY KEY,
   259|  keyword TEXT NOT NULL,
   260|  origin TEXT NOT NULL,            -- erank / rising_fission / manual
   261|  parent TEXT,                     -- 裂变来源词(注入②才有)
   262|  rising_value TEXT,               -- "Breakout" 或数值字符串
   263|  month TEXT NOT NULL,
   264|  ts TEXT NOT NULL
   265|);
   266|```
   267|
   268|【输出 schema】
   269|```json
   270|{"injected": true, "subcategory": "wall_decor"}
   271|```
   272|
   273|【API 契约】
   274|```
   275|POST /api/pool/manual-add     body:{keyword, erank_category, month}  → 注入③
   276|POST /api/pool/fission        body:{parent_keyword,new_keyword,erank_category,rising_value,month} → 注入②(N0内部调用)
   277|GET  /api/pool?month=&erank_category=
   278|→ data:{"total":20,"by_origin":{"erank":15,"rising_fission":4,"manual":1}, "keywords":[...]}
   279|```
   280|
   281|【验收示例（真实数据）】
   282|```
   283|前置：2026-05 home_living 已有 15 个 erank 来源词(含 wall art)
   284|输入：POST /api/pool/fission {parent:"wall art", new:"boho arch mirror", erank_category:"home_living", rising_value:"Breakout", month:"2026-05"}
   285|期望：
   286|  {"injected": true, "subcategory": "wall_decor"}   // mirror 命中 wall_decor 规则
   287|  再查 GET /api/pool?month=2026-05&erank_category=home_living
   288|  → total 变 16，by_origin.rising_fission == 1
   289|重复同一调用第二次 → {"injected": false, "reason":"already_in_pool"}
   290|```
   291|
   292|【人工验收步骤】
   293|1. 裂变注入 "boho arch mirror"，确认 total 15→16。
   294|2. 重复注入同词，确认不重复(injected:false)。
   295|3. candidate_pool_log 有一条 origin=rising_fission, parent="wall art"。
   296|
   297|【依赖】M1（assign_subcategory）。
   298|
   299|---
   300|
   301|### M3 — 时相引擎（先导/现状/滞后 → 生命周期阶段）⭐系统灵魂
   302|
   303|【目的】
   304|把三类信号（leading/current/lagging）综合成"生命周期阶段"标签。这是把系统从"排序工具"升级成"时机判断引擎"的核心。100% 规则驱动（R3）。
   305|
   306|【输入 schema（每个关键词的三类信号，由 N0/N1 产出）】
   307|```json
   308|{
   309|  "keyword": "boho arch mirror",
   310|  "leading_score": 88.0,     // N0 产出: Google rising + Pinterest, 0-100, null=无数据
   311|  "current_score": 45.0,     // N1 产出: eRank 4维 total_score, 0-100
   312|  "lagging_score": 20.0,     // N1 产出: listing销售集中度+标题同质化, 0-100, 越高越红海. 无数据时 null
   313|  "seasonal_is_noise": false // N0 季节伪信号标记
   314|}
   315|```
   316|
   317|【阶段判定伪代码（阈值为 config 可调常量，不写死）】
   318|```
   319|# 阈值从 config 表读取，不硬编码。初始值与依据见下方【阈值科学定值】
   320|HI = config.get("phase_threshold_hi", 65)   # 高档分界
   321|LO = config.get("phase_threshold_lo", 35)   # 低档分界
   322|
   def classify_phase(leading, current, lagging, seasonal_is_noise):
       """综合三时相 → 生命周期阶段。规则优先级从上到下，命中即返回。"""
       L = leading if leading is not None else 0
       C = current
       G = lagging

       # 伪信号优先过滤
       if seasonal_is_noise and L >= HI:
   331|        return "伪信号"                    # 季节性暴涨伪装成先导
   332|
   333|    if L >= HI and C < LO and G < LO:
   334|        return "埋伏期"        # ⭐埋伏机会: 先导热、现状冷、不红海 = 最佳进场
   335|    if L >= HI and C >= LO and C < HI and G < LO:
   336|        return "新兴期"      # ✨新兴: 先导起来、现状爬坡中、竞争未形成 = 黄金窗口
   337|    if L >= HI and C >= LO and C < HI and G >= HI:
   338|        return "震荡期"      # 矛盾信号: 先导热但竞争已固化 = 假阳性风险
   339|    if L >= HI and C >= HI and G < LO:
   340|        return "起飞期"       # 起飞中: 先导现状都热但还没红海，要快
   341|    if C >= HI and G >= HI:
   342|        return "红海期"     # 红海: 现状热且已固化，别进
   343|    if L < LO and C >= HI and G >= HI:
   344|        return "衰退期"     # 见顶/衰退
   345|    if L < LO and C < LO:
   346|        return "冷门期"          # 冷门
   347|    return "观望期"           # 其它
   348|```
   349|
   350|【阶段 → 前端标签映射（守 AGENTS.md 禁 emoji 当 UI）】
   351|```
   352|埋伏期       → [ 埋伏期 ]      浅绿底+黑字  排序最优先
   353|新兴期     → [ 新兴期 ]    浅蓝底+黑字  排序第二 ← v4.0干跑发现
   354|起飞期      → [ 起飞期 ]     浅黄底+黑字
   355|红海期    → [ 红海期 ]   浅红底+黑字  排序降权
   356|衰退期    → [ 衰退期 ]   浅红底+黑字
   357|震荡期     → [ 震荡期 ]    浅橙底+黑字  伪信号风险
   358|冷门期         → [ 冷门期 ]        浅灰底+黑字
   359|伪信号 → [ 伪信号 ] 浅灰底+黑字 过滤
   360|观望期      → [ 观望期 ]     浅灰底+黑字
   361|```
   362|
   363|【输出 schema（写入 keywords.phase_stage + phase_snapshot 表）】
   364|```json
   365|// boho arch mirror 的 (88,45,20) → 新兴期 (C=45在[35,65), 非埋伏期)
   366|{"keyword":"boho arch mirror", "phase_stage":"新兴期",
   367| "leading_score":88.0, "current_score":45.0, "lagging_score":20.0}
   368|```
   369|
   370|【新增表 phase_snapshot】
   371|```sql
   372|CREATE TABLE phase_snapshot (
   373|  id INTEGER PRIMARY KEY,
   374|  keyword TEXT NOT NULL, erank_category TEXT, subcategory TEXT,
   375|  leading_score REAL, current_score REAL, lagging_score REAL,
   376|  phase_stage TEXT NOT NULL, month TEXT NOT NULL, ts TEXT NOT NULL
   377|);
   378|```
   379|
   380|【API 契约】
   381|```
   382|POST /api/phase/classify?month=2026-05&erank_category=home_living
   383|→ 对该类目该月所有 keywords 跑 classify_phase，回填 phase_stage + 写 snapshot
   384|→ data:{"classified":16, "by_stage":{"埋伏期":2,"新兴期":3,"红海期":3,"冷门期":5,"观望期":3}}
   385|```
   386|
   387|【验收示例（真实数据 2026-05 home_living）】
   388|```
   389|真实输入(current_score = v3.1 total_score):
   390|  "wall art":   current=83.0   假设(P1前)leading=0, lagging=高(头部店铺集中)
   391|                → 暂判 红海期 或 观望期(取决 lagging)
   392|  裂变词 "boho arch mirror": leading=88(rising Breakout), current=45, lagging=20
   393|                → classify_phase(88,45,20,false) == "新兴期"  ✅核心验收点
   394|                (C=45在[35,65), L>=65, G<35 → 新兴期, 不是埋伏期. 埋伏期需要C<35)
   395|  "halloween":  seasonal_is_noise=true, 若 leading>=60 → "伪信号"
   396|```
   397|
   398|【人工验收步骤】
   399|1. 用上面三组数手工调 classify_phase，确认 boho arch mirror → 新兴期。
   400|2. `POST /api/phase/classify?...`，前端 keywords 表出现 phase_stage 标签列。
   401|3. 确认 新兴期 词排在 红海期 词前面（M3 只打标签，排序在 N2 做）。
   402|
   403|【依赖】N0(leading_score) + N1(current/lagging_score)。可先用桩数据验收 classify_phase 纯函数，再接真实上游。
   404|
   405|【阈值科学定值（HI / LO / age_median）】
   406|> 诚实前提：数据积累前的任何阈值都是"有依据的初始猜测"，不是科学真值。科学性体现在"可被数据校准"，而非"一次定对"。
   407|
   408|初始值与依据：
   409|```
   410|phase_threshold_hi = 65   # 不用 50。归一化分上半区拥挤(很多词 demand=100)，
   411|                          #   HI=65 避免过多词挤进"高"档，"高"才是真头部
   412|phase_threshold_lo = 35   # 留出明确"冷门"尾部，35 以下直接不看
   413|                          #   中间 35-65 = 观望期 = "数据不足以判断，需人工看"的诚实区间
   414|longtail_age_median_max = 180   # 可切入性上限(天)。业务依据：
   415|                          #   Etsy 新 listing 平均 2-3 月才进搜索靠前；中位数>180天=老链接霸榜，
   416|                          #   新店半年内挤不上去=不可切入。<90天=强可切入(算法在洗牌，机会窗)
   417|```
   418|全部写入 config 表，不硬编码（v3.1 已有 config 机制）。
   419|
   420|阈值校准回路（攒够数据后，这才是真科学）：
   421|```
   422|1. 每月跑完，记录每词的阶段判定 + 你的实际决策(BUY/IGNORE) + 后续是否盈利
   423|2. 三个月后回看命中率：
   424|   - "判 埋伏期 但你都 IGNORE" → HI 太低，上调
   425|   - "观望期 里藏着你 BUY 的好词" → HI 太高，下调
   426|3. 复用 v3.1 决策质量反馈闭环(Type A/B/C/D)做这个校准
   427|4. 未来样本充足时，可把固定 HI/LO 切换成"类目内三分位"动态阈值
   428|   (LO=33分位, HI=67分位)，比固定值更贴合各类目分布
   429|```
   430|
   431|【依赖】N0(leading_score) + N1(current/lagging_score)。
   432|
   433|---
   434|
   435|## 模块规格卡 · 漏斗节点层
   436|
   437|---
   438|
   439|### N0 — L0 先导层（Google Rising + Pinterest）【P0】
   440|
   441|【目的】
   442|给候选池每个词附 先导 分，并通过 rising 裂变发现全新候选词。这是"预判而非追热"的核心。
   443|
   444|【输入 schema】
   445|```
   446|候选池里某词 keyword(str) + erank_category + month
   447|```
   448|
   449|【数据获取方法】
   450|```
   451|源A Google Rising(pytrends related_queries):
   452|    pytrends.build_payload([keyword], timeframe='today 12-m', geo='US')
   453|    rq = pytrends.related_queries()[keyword]['rising']   # DataFrame: query, value
   454|    # value 为整数(增长%)或字符串 "Breakout"(>5000%)
   455|源C Google Momentum(已有 /trends/momentum, 复用)
   456|源B Pinterest: P1先做手工录入端点, 后续可接API
   457|```
   458|
   459|【清洗伪代码】
   460|```
   461|# 前置常量定义（实现者必读）
   462|SEASONAL_RE = re.compile(r"\b(christmas|halloween|valentine|easter|thanksgiving|mothers?.day|fathers?.day|st\.?patricks?)\b", re.I)
   463|# in_category_lexicon: 简单规则词典，check query 是否至少包含一个该类目的特征词
   464|# 实现方式: 复用 M1 的 SUBCATEGORY_KEYWORD_RULES[erank_category] 所有 patterns 平铺成一组,
   465|#           若 query 匹配任意一个 pattern → 属于该类目；全不匹配 → 不属于
   466|
   467|def compute_leading(keyword, erank_category, month):
   468|    rising = fetch_google_rising(keyword)        # list[{query, value}]
   469|    if rising is None: raise DependencyError(503, "google_trends_unavailable")  # R4
   470|
   471|    breakout_count = 0; max_growth = 0; new_words = []
   472|    for r in rising:
   473|        v = 9999 if r["value"]=="Breakout" else int(r["value"])
   474|        if v < 200: continue                     # 去噪: <200%不算先导突破
   475|        if SEASONAL_RE.search(r["query"]):       # 季节伪信号
   476|            mark_seasonal_noise(keyword); continue
   477|        if not in_category_lexicon(r["query"], erank_category): continue  # 相关性过滤
   478|        if v >= 9999: breakout_count += 1
   479|        max_growth = max(max_growth, v)
   480|        if not exists_in_pool(r["query"], month): new_words.append((r["query"], r["value"]))
   481|
   482|    pin = fetch_pinterest_flag(keyword)          # 手工录入表查, 命中=True; pin 值: rising/stable/declining/null
   483|    pin_trending = (pin == "rising")
   484|
   485|    # ── 信号一致性校验 (N0.1) ──
   486|    # 四种情况全是显式分支，不用默认值兜底（防止 pin 有值但非 trending 且 breakout=0 → CONFIRMED 穿透）
   487|    if pin is None:
   488|        consistency = "UNVERIFIED"  # 无 Pinterest 数据 → 证据不足, 降权
   489|    elif pin_trending:             # pin == "rising"
   490|        consistency = "CONFIRMED"   # 双源一致 → 真先导
   491|    elif breakout_count > 0:       # pin 非 rising 但 Google 有 breakout
   492|        consistency = "DIVERGENT"   # Google 涨但 Pinterest 跌/平 → 假阳性风险
   493|    else:                          # pin 非 rising, Google 也无 breakout
   494|        consistency = "UNVERIFIED"  # 双方都无先导信号 → 信息不足
   495|
   496|    # 震荡期/DIVERGENT 标记: 不裂变新词(防止假阳性词污染候选池)
   497|    if consistency == "DIVERGENT":
   498|        for w, _ in new_words:
   499|            insert(trend_signals, keyword=w, source="google_rising", trend_direction="unstable",
   500|                   trend_strength=0, month=month)
   501|
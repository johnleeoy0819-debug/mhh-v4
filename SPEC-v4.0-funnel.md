# SPEC-v4.0-funnel — Maple Hollow Home 多类目科学选品漏斗系统

> 本文档是 SPEC-v3.1（评分/利润/状态机骨架）的**上层补充规范**。
> v3.1 定义"一个关键词怎么打分、一个产品怎么算利润/走状态机"；
> 本 v4.0 定义"怎么从三大类目里，用科学漏斗逐层收窄，找出值得做的长尾产品"。
> 两者互补：v4.0 的 L4/L5 直接复用 v3.1 的利润计算与决策/状态机，不重复定义。

---

## 修订记录

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v4.0 | 2026-05-31 | ①引入"先导/现状/滞后"三时相数据模型 ②6 层漏斗（L0-L5）③多类目×多子分类隔离 ④候选词池与 rising 裂变回灌 ⑤每个节点写成可独立验收的"模块规格卡" |

**冻结声明**：本版本可直接交付任意能力的 LLM 逐卡实现。每张模块规格卡自给自足（输入 schema + 清洗伪代码 + 输出 schema + API 契约 + 真实数据验收示例 + 人工验收步骤）。后续仅做 bug 修正，不在本版无限加功能。

【实现优先级】：
- P0（最小可用闭环 — 必须一步到位）：M1 类目体系 → M2 候选池 → N0 L0先导(Google rising + N0.1信号一致性) → N1 L1赛道(滞后 A/B) → M3 时相引擎 → N2 推荐①一级词
  ⚠ N0 不是"增强"，是 M3+N2 正常工作的前提。拆到 P1 = P0 输出垃圾(mug 排第一)。
- P1（长尾+能力匹配）：N3 L2 长尾分析 + L3.1 能力画像匹配(能力匹配)
- P2（落地闭环+反馈，复用 v3.1）：N4 L3找货 → N5 L4利润 → N6 L5决策(N6.1 RESEARCH诊断) + feedback fields
- P3（横向复制）：把 home_living 验证过的漏斗复制到 wedding / jewelry（含 滞后 策略不串）

---

## 0. 给实现 LLM 的总规则（必读，违反即跑偏）

R1. **类目隔离铁律**：所有信号计算、归一化（min-max）、推荐排序，**必须**在同一 `(erank_category, subcategory)` 分组内进行。**禁止**跨大类目把关键词放进同一个池子归一化或排序。跨类目只能并列展示，不可同池比较。违反此条 = 整个系统输出无意义。

R2. **时相标签强制**：每个关键词候选必须带三类信号分（leading / current / lagging），缺失的用明确空值（null / 0.0 + confidence=0），**禁止**用某一类信号的值假装另一类。

R3. **AI 边界**（继承 v3.1 §2.2）：除非某张卡显式标注"本节点用 LLM"，否则该节点 100% 规则驱动。LLM 不做评分、不做最终筛选、不做决策。标注"用 LLM"的节点，LLM 只产候选+理由，规则层做硬门槛过滤。

R4. **不静默降级**：外部依赖（Google Trends / Pinterest / Etsy API / LLM）不可用时，对应端点返回 HTTP 503 并在 body 说明缺哪个依赖。**禁止**返回降级/猜测结果冒充真实数据。

R5. **数据真实性分层**（继承 v3.1 §2.1）：L1 真实(confidence=1.0) / L2 估算(0.5，UI 灰色+~前缀) / L3 缺失(0.0，UI 显示 "--")。

R6. **金额单位后缀**：所有金额字段必须带 `_rmb` 或 `_usd` 后缀。所有时间字段 ISO 8601 字符串。所有数据库里的 JSON 字段在应用层 序列化/反序列化。

R7. **统一 API 响应格式**（继承 AGENTS.md）：
```json
{"success": true, "data": {...}, "error": null, "meta": {"timestamp":"...","version":"v4.0"}}
```
错误：`{"success": false, "data": null, "error": {"code":"...","message":"...","field":"..."}}`

R8. **每张卡 = 一个可独立验收单元**。实现一张卡后，跑该卡的"人工验收步骤"，对拍"验收示例"的期望输出，通过才进下一张。

R9. **文件命名小写**（继承 AGENTS.md）：`phase_engine.py` 不是 `PhaseEngine.py`。所有 public 函数有中文 docstring + 类型标注。

R10. **复用优先**：L4 利润 = 直接调 v3.1 `/api/profit/calculate`；L5 决策/状态机 = 直接调 v3.1 `/api/decisions` 和 `/api/products/{id}/transition`。本 SPEC 不重新定义这些，只定义它们的输入怎么从漏斗上游来。

---

## 1. 系统全景：6 层漏斗

```
候选词池 (candidate pool, 按 类目×子分类 隔离)
   │  注入①: eRank 4表现状热词(冷启动唯一种子)
   │  注入②: L0 rising 裂变出的新词(回灌)
   │  注入③: 人工录入(Pinterest/逛Etsy发现)
   ▼
[N0 / L0 先导层]  → 每个候选词附 先导 分(Google rising + Pinterest)
   ▼
[N1 / L1 赛道层]  → 每个候选词附 现状 4维分(eRank) + 滞后 分(listing销售集中度+标题同质化)
   ▼
[M3 时相引擎]     → 综合 leading/current/lagging → 生命周期阶段标签
   ▼
[N2 推荐①]        → 输出 3-5 个一级关键词(按时相阶段排序) ──┐ 人在环:
   ▼                                                    │ 你拿一级词去
[N3 / L2 长尾层]  ← 你上传 eRank Keyword Tool 的 Top    │ eRank 搜词下载
   ▼                                                    │ Top Listings CSV
[N3 推荐②]        → 输出 5-10 组长尾关键词(机会分排序) ──┐ 人在环:
   ▼                                                    │ 你拿长尾词
[N4 / L3 找货层]  ← 你录入 1688 成本(suppliers)          │ 去1688找货
   ▼
[N5 / L4 利润层]  → 调 v3.1 /api/profit/calculate → 毛利率，回填 keywords.margin
   ▼
[N6 / L5 决策层]  → 汇总全信号信号卡 → 你拍板 BUY/RESEARCH/IGNORE (调 v3.1)
```

每层的"职责唯一性"自检（科学漏斗判据：每层必须能淘汰上层放进来的候选）：

| 层 | 回答的唯一问题 | 收窄维度 | 主数据源 | 时相 |
|----|---------------|---------|---------|------|
| L0 | 这方向未来会不会火 | 时间(先导) | Google rising, Pinterest | 先导 |
| L1 | 这方向现在值不值得进 | 类目热度 | eRank 4表 | 现状 + 滞后 |
| L2 | 具体哪个长尾词能切进去 | 竞争可入性 | eRank Keyword Tool CSV | 现状(竞争) |
| L3 | 这词对应啥货、成本多少 | 供给可得性 | 1688(手工/OpenCLI) | — |
| L4 | 这货能不能赚钱 | 经济性 | v3.1 利润引擎 | — |
| L5 | 买不买 | 风险/人判断 | 全信号汇总 | — |

---

## 2. 数据模型总览（与现有表的关系）

现有表（v3.1 已建，本 SPEC 复用/扩展）：
- `keywords` — 候选词主表。**v4 扩展**：加 `subcategory`、`source_origin`、`phase_stage` 字段（见 M1/M3）。
- `trend_signals` — 先导/趋势信号。**v4 复用**：source ∈ {google_rising, google_momentum, pinterest}，已有字段够用。
- `erank_listings` — Top Listings。**v4 复用**：L2 长尾分析 + 滞后 数据源。
- `most_clicked_shops` — 头部店铺。**v4不再作为滞后源**(CSV无百分比)，保留供参考。
- `top_subcategories` — 子分类热度。**v4 复用**：子分类种子。
- `suppliers` — 1688 成本。**v4 复用**：L3 输入。

v4 新增表：
- `candidate_pool_log` — 候选词来源审计（记录每个词从哪注入、何时裂变产生）。
- `phase_snapshot` — 每月每词的时相快照（leading/current/lagging 三分 + 阶段标签），供历史回看。
- **反馈闭环字段**（扩展现有 products / erank_listings 表，不建新表）：

v4 对 products 表的反馈闭环扩展 🟡P2：
> 业务盲区：没有"系统预测 vs 实际表现"的结构化记录，阈值校准是玄学。
```
ALTER TABLE products ADD COLUMN predicted_opportunity_score REAL;  -- 上架前系统给的机会分
ALTER TABLE products ADD COLUMN actual_views_30d   INTEGER;        -- 上架30天真实浏览
ALTER TABLE products ADD COLUMN actual_sales_30d   INTEGER;        -- 上架30天真实销量
ALTER TABLE products ADD COLUMN 预测偏差   REAL;           -- actual_sales_30d - predicted (自动算, 负=高估, 正=低估)
ALTER TABLE products ADD COLUMN 偏差原因   TEXT;           -- 人工标注枚举: pricing/tagging/seasonal/quality/luck/capability_mismatch/unknown
```
校准回路跑法（有了这组数据才能科学）：
```
1. 每月拉上架上满30天的产品: GROUP BY 偏差原因 统计
2. "偏差原因=capability_mismatch 占比高" → 能力画像匹配阈值需调
3. "偏差原因=seasonal 占比高" → 伪信号 过滤规则需扩
4. "埋伏期 词 预测偏差 均值 -15" → HI 太低(过松), 上调
5. 把偏差原因分布反馈给 SPEC 设计者（你），而不是自动改阈值
```

详细字段见对应模块规格卡。

---

## 模块规格卡 · 基础设施层

> 每张卡格式固定：【目的】【输入 schema】【清洗/加工伪代码】【输出 schema】【API 契约】【验收示例(真实数据)】【人工验收步骤】【依赖】。
> 弱 LLM：严格按卡实现，不要自由发挥。实现完一张，跑"人工验收步骤"，对拍"验收示例"，通过再下一张。

---

### M1 — 类目体系（多类目 × 子分类隔离）

【目的】
建立"大类目 → 子分类"两级体系，让所有下游计算能按 (erank_category, subcategory) 隔离。这是 R1 铁律的数据基础。

【类目枚举（硬编码常量，禁止 LLM 自由扩展）】
```python
# app/constants/categories.py
CATEGORY_TREE = {
    "home_living": ["wall_decor", "desk_decor"],          # 用户当前主战场
    "wedding":     ["wedding_signs", "wedding_favors", "table_decor"],
    "jewelry":     ["necklaces", "earrings", "rings"],
}
# erank_category 取 CATEGORY_TREE 的 key
# subcategory   取对应 value 列表之一，或 "unassigned"(未归类)
```

【输入 schema】无（这是常量 + 表结构扩展）

【表结构扩展 — keywords 表加 3 字段】
```sql
ALTER TABLE keywords ADD COLUMN subcategory   TEXT DEFAULT 'unassigned';
ALTER TABLE keywords ADD COLUMN source_origin TEXT DEFAULT 'erank';   -- erank / rising_fission / manual
ALTER TABLE keywords ADD COLUMN phase_stage   TEXT DEFAULT NULL;       -- 由 M3 写入
```

【子分类归类规则（清洗伪代码）】
```
def assign_subcategory(keyword_text, erank_category):
    """把一个关键词归到子分类。规则优先，匹配不上则 unassigned。"""
    rules = SUBCATEGORY_KEYWORD_RULES[erank_category]   # 词典见下
    for subcat, patterns in rules.items():
        if any(re.search(p, keyword_text.lower()) for p in patterns):
            return subcat
    return "unassigned"

SUBCATEGORY_KEYWORD_RULES = {
  "home_living": {
     "wall_decor": [r"\bwall\b", r"\bposter\b", r"\bprint\b", r"\bmirror\b", r"\bart\b", r"\btapestry\b"],
     "desk_decor": [r"\bdesk\b", r"\borganizer\b", r"\btray\b", r"\bstand\b", r"\bpen\b"],
  },
  # wedding / jewelry 同构，P4 阶段补
}
```

【输出 schema】keywords 行新增三字段被正确填充。

【API 契约】
```
GET /api/categories
→ data: {"tree": CATEGORY_TREE}

POST /api/categories/reassign?month=2026-05&erank_category=home_living
→ 对该类目该月所有 keywords 重跑 assign_subcategory，回填 subcategory
→ data: {"updated": <int>, "by_subcategory": {"wall_decor": 8, "desk_decor": 2, "unassigned": 5}}
```

【验收示例（真实数据 2026-05 home_living）】
```
输入：keywords 表里 "wall art"(home_living), "home decor", "furniture", "halloween"
调用：POST /api/categories/reassign?month=2026-05&erank_category=home_living
期望输出（assign_subcategory 结果）：
  "wall art"   → wall_decor   (命中 \bart\b 和 \bwall\b)
  "home decor" → unassigned   (无 desk/wall 具体词)
  "furniture"  → unassigned
  "halloween"  → unassigned
  by_subcategory 至少 wall_decor>=1
```

【人工验收步骤】
1. `GET /api/categories` 返回三大类目树。
2. `POST /api/categories/reassign?...home_living`，检查 "wall art" 的 subcategory 变成 "wall_decor"。
3. 前端 keywords 表新增 subcategory 列，"wall art" 行显示 wall_decor。

【依赖】无。这是地基，最先实现。

---

### M2 — 候选词池（candidate pool + 三来源注入 + rising 裂变回灌）

【目的】
管理"候选一级词"的来源与生命周期。解决先有鸡先有蛋问题：冷启动种子来自 eRank，运转后由 rising 裂变和人工扩充。

【三个注入口（source_origin 字段区分）】
```
注入① erank          : eRank Category Report 导入时自动入池(冷启动唯一种子)
注入② rising_fission : N0 L0 的 Google rising 吐出的、池中不存在的新词，回灌入池
注入③ manual         : 用户在前端手工添加(Pinterest/逛Etsy发现)
```

【输入 schema（注入②裂变回灌）】
```json
{
  "parent_keyword": "wall art",        // 裂变来源词
  "new_keyword": "boho arch mirror",   // rising 吐出的新词
  "erank_category": "home_living",     // 继承父词
  "rising_value": "Breakout",          // 或数值如 450
  "month": "2026-05"
}
```

【裂变回灌伪代码】
```
def fission_inject(parent_keyword, new_keyword, erank_category, rising_value, month):
    """rising 新词回灌候选池。已存在则跳过(不重复)。"""
    if exists(keywords, keyword=new_keyword, month=month):
        return {"injected": False, "reason": "already_in_pool"}
    subcat = assign_subcategory(new_keyword, erank_category)   # 复用 M1
    insert(keywords, keyword=new_keyword, erank_category=erank_category,
           subcategory=subcat, source_origin="rising_fission",
           month=month, created_at=now_iso())
    insert(candidate_pool_log, keyword=new_keyword, origin="rising_fission",
           parent=parent_keyword, rising_value=str(rising_value), month=month, ts=now_iso())
    return {"injected": True, "subcategory": subcat}
```

【新增表 candidate_pool_log】
```sql
CREATE TABLE candidate_pool_log (
  id INTEGER PRIMARY KEY,
  keyword TEXT NOT NULL,
  origin TEXT NOT NULL,            -- erank / rising_fission / manual
  parent TEXT,                     -- 裂变来源词(注入②才有)
  rising_value TEXT,               -- "Breakout" 或数值字符串
  month TEXT NOT NULL,
  ts TEXT NOT NULL
);
```

【输出 schema】
```json
{"injected": true, "subcategory": "wall_decor"}
```

【API 契约】
```
POST /api/pool/manual-add     body:{keyword, erank_category, month}  → 注入③
POST /api/pool/fission        body:{parent_keyword,new_keyword,erank_category,rising_value,month} → 注入②(N0内部调用)
GET  /api/pool?month=&erank_category=
→ data:{"total":20,"by_origin":{"erank":15,"rising_fission":4,"manual":1}, "keywords":[...]}
```

【验收示例（真实数据）】
```
前置：2026-05 home_living 已有 15 个 erank 来源词(含 wall art)
输入：POST /api/pool/fission {parent:"wall art", new:"boho arch mirror", erank_category:"home_living", rising_value:"Breakout", month:"2026-05"}
期望：
  {"injected": true, "subcategory": "wall_decor"}   // mirror 命中 wall_decor 规则
  再查 GET /api/pool?month=2026-05&erank_category=home_living
  → total 变 16，by_origin.rising_fission == 1
重复同一调用第二次 → {"injected": false, "reason":"already_in_pool"}
```

【人工验收步骤】
1. 裂变注入 "boho arch mirror"，确认 total 15→16。
2. 重复注入同词，确认不重复(injected:false)。
3. candidate_pool_log 有一条 origin=rising_fission, parent="wall art"。

【依赖】M1（assign_subcategory）。

---

### M3 — 时相引擎（先导/现状/滞后 → 生命周期阶段）⭐系统灵魂

【目的】
把三类信号（leading/current/lagging）综合成"生命周期阶段"标签。这是把系统从"排序工具"升级成"时机判断引擎"的核心。100% 规则驱动（R3）。

【输入 schema（每个关键词的三类信号，由 N0/N1 产出）】
```json
{
  "keyword": "boho arch mirror",
  "leading_score": 88.0,     // N0 产出: Google rising + Pinterest, 0-100, null=无数据
  "current_score": 45.0,     // N1 产出: eRank 4维 total_score, 0-100
  "lagging_score": 20.0,     // N1 产出: listing销售集中度+标题同质化, 0-100, 越高越红海. 无数据时 null
  "seasonal_is_noise": false // N0 季节伪信号标记
}
```

【阶段判定伪代码（阈值为 config 可调常量，不写死）】
```
# 阈值从 config 表读取，不硬编码。初始值与依据见下方【阈值科学定值】
HI = config.get("phase_threshold_hi", 65)   # 高档分界
LO = config.get("phase_threshold_lo", 35)   # 低档分界

def classify_phase(leading, current, lagging, seasonal_is_noise):
    """综合三时相 → 生命周期阶段。规则优先级从上到下，命中即返回。"""
    L = leading if leading is not None else 0
    C = current
    G = lagging

    # 伪信号优先过滤
    if seasonal_is_noise and L >= HI:
        return "伪信号"                    # 季节性暴涨伪装成先导

    if L >= HI and C < LO and G < LO:
        return "埋伏期"        # ⭐埋伏机会: 先导热、现状冷、不红海 = 最佳进场
    if L >= HI and C >= LO and C < HI and G < LO:
        return "新兴期"      # ✨新兴: 先导起来、现状爬坡中、竞争未形成 = 黄金窗口
    if L >= HI and C >= LO and C < HI and G >= HI:
        return "震荡期"      # 矛盾信号: 先导热但竞争已固化 = 假阳性风险
    if L >= HI and C >= HI and G < LO:
        return "起飞期"       # 起飞中: 先导现状都热但还没红海，要快
    if C >= HI and G >= HI:
        return "红海期"     # 红海: 现状热且已固化，别进
    if L < LO and C >= HI and G >= HI:
        return "衰退期"     # 见顶/衰退
    if L < LO and C < LO:
        return "冷门期"          # 冷门
    return "观望期"           # 其它
```

【阶段 → 前端标签映射（守 AGENTS.md 禁 emoji 当 UI）】
```
埋伏期       → [ 埋伏期 ]      浅绿底+黑字  排序最优先
新兴期     → [ 新兴期 ]    浅蓝底+黑字  排序第二 ← v4.0干跑发现
起飞期      → [ 起飞期 ]     浅黄底+黑字
红海期    → [ RED OCEAN ]   浅红底+黑字  排序降权
衰退期    → [ 衰退期 ]   浅红底+黑字
震荡期     → [ 震荡期 ]    浅橙底+黑字  伪信号风险
冷门期         → [ 冷门期 ]        浅灰底+黑字
伪信号 → [ FALSE SIGNAL ] 浅灰底+黑字 过滤
观望期      → [ 观望期 ]     浅灰底+黑字
```

【输出 schema（写入 keywords.phase_stage + phase_snapshot 表）】
```json
// boho arch mirror 的 (88,45,20) → 新兴期 (C=45在[35,65), 非埋伏期)
{"keyword":"boho arch mirror", "phase_stage":"新兴期",
 "leading_score":88.0, "current_score":45.0, "lagging_score":20.0}
```

【新增表 phase_snapshot】
```sql
CREATE TABLE phase_snapshot (
  id INTEGER PRIMARY KEY,
  keyword TEXT NOT NULL, erank_category TEXT, subcategory TEXT,
  leading_score REAL, current_score REAL, lagging_score REAL,
  phase_stage TEXT NOT NULL, month TEXT NOT NULL, ts TEXT NOT NULL
);
```

【API 契约】
```
POST /api/phase/classify?month=2026-05&erank_category=home_living
→ 对该类目该月所有 keywords 跑 classify_phase，回填 phase_stage + 写 snapshot
→ data:{"classified":16, "by_stage":{"埋伏期":2,"新兴期":3,"红海期":3,"冷门期":5,"观望期":3}}
```

【验收示例（真实数据 2026-05 home_living）】
```
真实输入(current_score = v3.1 total_score):
  "wall art":   current=83.0   假设(P1前)leading=0, lagging=高(头部店铺集中)
                → 暂判 红海期 或 观望期(取决 lagging)
  裂变词 "boho arch mirror": leading=88(rising Breakout), current=45, lagging=20
                → classify_phase(88,45,20,false) == "新兴期"  ✅核心验收点
                (C=45在[35,65), L>=65, G<35 → 新兴期, 不是埋伏期. 埋伏期需要C<35)
  "halloween":  seasonal_is_noise=true, 若 leading>=60 → "伪信号"
```

【人工验收步骤】
1. 用上面三组数手工调 classify_phase，确认 boho arch mirror → 新兴期。
2. `POST /api/phase/classify?...`，前端 keywords 表出现 phase_stage 标签列。
3. 确认 新兴期 词排在 红海期 词前面（M3 只打标签，排序在 N2 做）。

【依赖】N0(leading_score) + N1(current/lagging_score)。可先用桩数据验收 classify_phase 纯函数，再接真实上游。

【阈值科学定值（HI / LO / age_median）】
> 诚实前提：数据积累前的任何阈值都是"有依据的初始猜测"，不是科学真值。科学性体现在"可被数据校准"，而非"一次定对"。

初始值与依据：
```
phase_threshold_hi = 65   # 不用 50。归一化分上半区拥挤(很多词 demand=100)，
                          #   HI=65 避免过多词挤进"高"档，"高"才是真头部
phase_threshold_lo = 35   # 留出明确"冷门"尾部，35 以下直接不看
                          #   中间 35-65 = 观望期 = "数据不足以判断，需人工看"的诚实区间
longtail_age_median_max = 180   # 可切入性上限(天)。业务依据：
                          #   Etsy 新 listing 平均 2-3 月才进搜索靠前；中位数>180天=老链接霸榜，
                          #   新店半年内挤不上去=不可切入。<90天=强可切入(算法在洗牌，机会窗)
```
全部写入 config 表，不硬编码（v3.1 已有 config 机制）。

阈值校准回路（攒够数据后，这才是真科学）：
```
1. 每月跑完，记录每词的阶段判定 + 你的实际决策(BUY/IGNORE) + 后续是否盈利
2. 三个月后回看命中率：
   - "判 埋伏期 但你都 IGNORE" → HI 太低，上调
   - "观望期 里藏着你 BUY 的好词" → HI 太高，下调
3. 复用 v3.1 决策质量反馈闭环(Type A/B/C/D)做这个校准
4. 未来样本充足时，可把固定 HI/LO 切换成"类目内三分位"动态阈值
   (LO=33分位, HI=67分位)，比固定值更贴合各类目分布
```

【依赖】N0(leading_score) + N1(current/lagging_score)。

---

## 模块规格卡 · 漏斗节点层

---

### N0 — L0 先导层（Google Rising + Pinterest）【P0】

【目的】
给候选池每个词附 先导 分，并通过 rising 裂变发现全新候选词。这是"预判而非追热"的核心。

【输入 schema】
```
候选池里某词 keyword(str) + erank_category + month
```

【数据获取方法】
```
源A Google Rising(pytrends related_queries):
    pytrends.build_payload([keyword], timeframe='today 12-m', geo='US')
    rq = pytrends.related_queries()[keyword]['rising']   # DataFrame: query, value
    # value 为整数(增长%)或字符串 "Breakout"(>5000%)
源C Google Momentum(已有 /trends/momentum, 复用)
源B Pinterest: P1先做手工录入端点, 后续可接API
```

【清洗伪代码】
```
# 前置常量定义（实现者必读）
SEASONAL_RE = re.compile(r"\b(christmas|halloween|valentine|easter|thanksgiving|mothers?.day|fathers?.day|st\.?patricks?)\b", re.I)
# in_category_lexicon: 简单规则词典，check query 是否至少包含一个该类目的特征词
# 实现方式: 复用 M1 的 SUBCATEGORY_KEYWORD_RULES[erank_category] 所有 patterns 平铺成一组,
#           若 query 匹配任意一个 pattern → 属于该类目；全不匹配 → 不属于

def compute_leading(keyword, erank_category, month):
    rising = fetch_google_rising(keyword)        # list[{query, value}]
    if rising is None: raise DependencyError(503, "google_trends_unavailable")  # R4

    breakout_count = 0; max_growth = 0; new_words = []
    for r in rising:
        v = 9999 if r["value"]=="Breakout" else int(r["value"])
        if v < 200: continue                     # 去噪: <200%不算先导突破
        if SEASONAL_RE.search(r["query"]):       # 季节伪信号(seasonal_is_noise标记)
            mark_seasonal_noise(keyword); continue
        if ENTITY_NOISE_RE.search(r["query"]):   # 实体噪音: 人名/新闻/viral, 硬过滤+不裂变
            continue
        if not in_category_lexicon(r["query"], erank_category): continue  # 相关性过滤
        if v >= 9999: breakout_count += 1
        max_growth = max(max_growth, v)
        if not exists_in_pool(r["query"], month): new_words.append((r["query"], r["value"]))

    pin = fetch_pinterest_flag(keyword)          # 手工录入表查, 命中=True; pin 值: rising/stable/declining/null
    pin_trending = (pin == "rising")

    # ── 信号一致性校验 (N0.1) ──
    # 四种情况全是显式分支，不用默认值兜底（防止 pin 有值但非 trending 且 breakout=0 → CONFIRMED 穿透）
    if pin is None:
        consistency = "UNVERIFIED"  # 无 Pinterest 数据 → 证据不足, 降权
    elif pin_trending:             # pin == "rising"
        consistency = "CONFIRMED"   # 双源一致 → 真先导
    elif breakout_count > 0:       # pin 非 rising 但 Google 有 breakout
        consistency = "DIVERGENT"   # Google 涨但 Pinterest 跌/平 → 假阳性风险
    else:                          # pin 非 rising, Google 也无 breakout
        consistency = "UNVERIFIED"  # 双方都无先导信号 → 信息不足

    # 震荡期/DIVERGENT 标记: 不裂变新词(防止假阳性词污染候选池)
    if consistency == "DIVERGENT":
        for w, _ in new_words:
            insert(trend_signals, keyword=w, source="google_rising", trend_direction="unstable",
                   trend_strength=0, month=month)
        new_words = []  # 清空裂变列表

    # leading_score 计算: 一致性调权
    consistency_multiplier = {"CONFIRMED": 1.0, "UNVERIFIED": 0.5, "DIVERGENT": 0.2}
    raw_leading = 40*min(breakout_count,2)/2 + 40*min(max_growth,9999)/9999
    if pin and pin_trending: raw_leading += 20   # Pinterest 正向确认才加分
    leading_score = clamp(raw_leading * consistency_multiplier[consistency], 0, 100)
    # 回灌裂变新词(注入②, 调 M2.fission_inject) — 仅 CONFIRMED/UNVERIFIED 裂变
    if consistency != "DIVERGENT":
        for w, val in new_words: fission_inject(keyword, w, erank_category, val, month)
    return {"keyword":keyword, "leading_score":leading_score,
            "breakout_count":breakout_count, "fission_new_words":[w for w,_ in new_words],
            "consistency": consistency}
```

【输出 schema → 写 trend_signals】
```json
{"keyword":"wall art","leading_score":72.0,"breakout_count":1,
 "fission_new_words":["boho arch mirror","coastal wall print"]}
```
写库: trend_signals(keyword, source="google_rising", trend_direction="rising",
trend_strength=leading_score, month)

【API 契约】
```
POST /api/leading/compute?month=2026-05&erank_category=home_living
→ 对池中每词跑 compute_leading(限流: rising调用间隔>=1s防封)
→ data:{"computed":15,"fission_added":4,"errors":[]}
POST /api/leading/pinterest  body:{keyword,style,direction,month}   手工录入Pinterest先导
GET  /api/leading?month=&erank_category=  → 每词leading_score + 裂变来源
依赖不可用 → 503 {"error":{"code":"google_trends_unavailable"}}  # 禁降级
```

【验收示例（真实数据）】
```
输入: keyword="wall art"(2026-05 home_living, 真实存在)
调用: POST /api/leading/compute?month=2026-05&erank_category=home_living
期望:
  - wall art 拿到 leading_score(0-100数值, 非null)
  - 若 rising 含 "boho..." 类新词 → 自动裂变入池(M2 total增加)
  - trend_signals 新增 source=google_rising 行
  - 若 Google Trends 不可达 → HTTP 503, 不返回假分
```

【人工验收步骤】
1. 跑 compute，确认 wall art 有 leading_score。
2. GET /api/pool 确认裂变新词进池(total 增加)。
3. 断网/改错 geo 模拟依赖失败 → 确认返回 503 而非 0 分。

【依赖】M2(裂变回灌)、pytrends。

【N0.1 — 信号一致性校验（防双假阳性）🔴P0 关键】
> 业务盲区：Google rising 暴涨 ≠ 真趋势。两类假阳性：
>   A 新闻驱动：viral video 导致一周暴涨，下月归零
>   B 回光返照：品类末期 Google 仍反映搜索但 Pinterest 已在跌
> 解决：用 Pinterest 独立源做交叉验证。

校验矩阵（在 compute_leading 内嵌执行）：
```
Google rising ▲  AND  Pinterest ▲      → CONFIRMED    multiplier=1.0  真先导
Google rising ▲  AND  Pinterest ▼/平   → DIVERGENT    multiplier=0.2  假阳性风险
Google rising ▲  AND  Pinterest 无数据  → UNVERIFIED  multiplier=0.5  证据不足(降权不排除)
Google rising 平 AND  Pinterest ▲      → CONFIRMED    (Pinterest先导,Google尚未反映)
```
CONFIRMED vs DIVERGENT：DIVERGENT 时 (a) leading 分打折至 0.2× (b) **不裂变新词** — 防止新闻驱动假阳性词污染候选池
UNVERIFIED：Pinterest 缺失时不排除该词(降权保留)，因为手工录入覆盖率非 100%

【N0.1 第二层过滤：实体噪音（ENTITY_NOISE_RE）🟡】
> 业务盲区: in_category_lexicon 判断"是否属于该类目"，无法识别品牌/人名/新闻事件。
>   "mike epps mug photo" 中 "mug" 匹配 → 通过了类目检查，但它是新闻事件不是家居趋势。
> 职责分离: in_category_lexicon = 类目相关性(不改)，ENTITY_NOISE_RE = 实体噪音过滤(新增)。

```
# app/constants/entity_noise.py (P0实现)
ENTITY_NOISE_RE = re.compile(r"\b("
    r"lidl|aldi|tesco|walmart|target|ikea|wayfair|amazon|lowes|homedepot|"
    r"mike\s?epps|tiger\s?woods|heidi\s?klum|taylor\s?swift|"
    r"photo|meme|video|tiktok|viral|challenge|"
    r"how\s?to|diy|tutorial|repair|fix|install|"
    r"near\s?me|store|shop"
    r")\b", re.I)

def is_entity_noise(query_text):
    return bool(ENTITY_NOISE_RE.search(query_text.lower()))
```

> 维护说明: 名人名单(mike epps等)仅作示例,主要依赖通用噪音词(photo/meme/viral/tiktok)过滤。
> 通用词覆盖绝大部分新闻驱动噪音;名人名单如需精确拦截时事人物,需定期人工更新。
> 不建议将 ENTITY_NOISE_RE 写成数据库配置——它是正则性能敏感且不频繁变更的硬规则。

在 compute_leading 的 rising 循环中，调用顺序:
  1. SEASONAL_RE 检查 → 季节噪音
  2. ENTITY_NOISE_RE 检查 → 实体噪音(新增, 硬过滤+不裂变)
  3. in_category_lexicon → 类目相关性
  → 三者独立，各司其职。

---

### N1 — L1 赛道层（eRank 4维 现状 + 头部店铺 滞后）【P0】

【目的】
给每个候选词算 现状 4维分(已由 v3.1 signals 实现)和 滞后 滞后分(新增)。

【输入 schema】
```
keywords 表该类目该月所有词 + erank_listings 表(滞后数据源, P1)
```

【现状 分】= 直接复用 v3.1 /api/signals/calculate 的 total_score(已实现, 真实值见验收)。
本卡不重新实现 现状,只新增 滞后。

【滞后 滞后分伪代码（两层，依赖 erank_listings 数据 → P1）🔴P1】
> 数据源约束: eRank Top Listings CSV 不含店铺列(实测验证)。eRank Category Report 的 Most Clicked Shops 不含百分比。
>   → 弃用店铺集中度，改用 listing 销售集中度（字段真实可用）。
>   → P0 阶段无 listing 数据 → lagging = null。P1 有数据后自然补上。

```
# app/constants/lagging_strategy.py
# 策略选择依据:
#   同质化策略: 该品类买家决策依赖风格/设计差异,标题同质化低=差异化空间大=可切入
#   集中度策略: 该品类是功能性标品,买家对材质/规格敏感,头部集中度是核心竞争指标
# ⚠ P3复制类目时, 每个新子分类必须在此处加注释说明选择理由, 避免系统性误判
滞后_STRATEGY = {
    "wall_decor":     "同质化策略",    # 壁挂装饰是设计驱动,风格方向多,同质化低=可切
    "desk_decor":     "集中度策略",  # 桌面收纳是功能标品,头部店份额是真实竞争壁垒
    "wedding_signs":  "同质化策略",    # 婚礼标识是设计驱动,个性化定制空间大
    "wedding_favors": "集中度策略",  # 婚礼小礼品趋于标准化,看份额
    "table_decor":    "同质化策略",    # 桌面装饰偏设计感
    "necklaces":      "同质化策略",    # 项链风格分散(复古/极简/boho/...),看差异化
    "earrings":       "同质化策略",    # 耳环同上
    "rings":          "集中度策略",  # 戒指经典款集中(婚戒/素圈占大头)
}

# 滞后_A: listing 销售集中度（不猜，靠真实 Est.Sales 算）
# Top 5 listing 的 Est.Sales 之和 / 全部 listing 的 Est.Sales 之和
# >50% → 高度集中(头几条吃掉了大半销量); <20% → 分散
def listing_concentration(listings):
    total = sum(l.est_sales or 0 for l in listings)
    if total == 0: return None
    top5 = sum(sorted([l.est_sales or 0 for l in listings], reverse=True)[:5])
    ratio = top5 / total
    return clamp((ratio - 0.2) / (0.5 - 0.2) * 100, 0, 100)

# 滞后_B: 标题同质化（不变）
# Top 10 listing 标题的 2-gram Jaccard 中位数
# >0.7 → 高度同质(真红海); <0.3 → 差异化大(可切)

def compute_lagging(erank_category, subcategory, month):
    """滞后 查询范围: 取该子分类下所有已收录一级词的 Top Listings 并集。
       例: wall_decor 下有 wall art+boho arch mirror+neutral wall art,
           取所有三词 associated 的 erank_listings 合并计算。
       原因: 滞后 衡量的是"这个子分类赛道的整体竞争态势"，不按单一级词分开。"""
    listings = query_erank_listings(subcategory, keyword_set=该子分类所有候选词, month=month)
    if len(listings) < 5:
        return {"lagging": None, "reason": "insufficient_listings"}  # P0诚实null

    lagging_a = listing_concentration(listings)
    # B: Jaccard — 排除 outlier 标记的 listing（一个爆款标题差异会拉低同质化分）
    # outlier 定义: est_sales > 同组 P95, 由 N3 clean_listings 标记
    normal = [l for l in listings[:15] if not getattr(l, 'outlier', False)]
    if len(normal) < 5: normal = listings[:15]  # 排除后不够 → 回退全量
    titles = [normalize(l.listing_title) for l in normal[:10]]
    token_sets = [set(ngrams(t, 2)) for t in titles]
    pairs = [(token_sets[i], token_sets[j]) for i in range(len(token_sets))
             for j in range(i+1, len(token_sets))]
    jaccards = [len(a&b)/len(a|b) for a,b in pairs if len(a|b)>0]
    lagging_b = clamp((median(jaccards) - 0.3)/(0.7-0.3)*100, 0, 100) if jaccards else None

    strategy = 滞后_STRATEGY.get(subcategory, "同质化策略")
    lagging_score = lagging_b if strategy == "同质化策略" else lagging_a
    return {"lagging": lagging_score, "lagging_a": lagging_a, "lagging_b": lagging_b,
            "strategy": strategy, "listing_count": len(listings)}
```

【输出 schema】
```json
{"erank_category":"home_living","lagging":62.0,"lagging_a":55.0,"lagging_b":62.0,"strategy":"同质化策略","listing_count":100}
```
P0(无 listing 数据): `{"lagging":null,"reason":"insufficient_listings"}`

【API 契约】
```
POST /api/lagging/compute?month=2026-05&erank_category=home_living
→ data:{"lagging":62.0,"lagging_a":55.0,"lagging_b":62.0,"strategy":"同质化策略","listing_count":100}
(现状 仍走 v3.1 POST /api/signals/calculate?month=&erank_category=)
```

【验收示例（真实数据 2026-05 home_living）】
```
现状(v3.1已验证, 真实): wall art total_score=83.0, gift/home decor/furniture=77.0, halloween=73.1
滞后(P0, 无listing): "wall art" 的子分类 wall_decor → lagging=null (insufficient_listings)
滞后(P1, 有listing后): 取 wall_decor 下 listing, 算 listing_concentration + Jaccard
```

【人工验收步骤】
1. v3.1 signals/calculate 确认 wall art=83.0(已知真实值)。
2. lagging/compute(P0): 无 listing → 返回 null + reason="insufficient_listings"。
3. lagging/compute(P1,有listing后): 返回 lagging_a/b + strategy。同质化策略子分类 lagging=lagging_b。

【依赖】v3.1 signals、erank_listings 数据(P1后有)。

---

### N2 — 推荐①一级关键词（纯规则）【P0】

【目的】
综合 leading/current/lagging + 时相阶段，输出 3-5 个一级关键词供用户去 eRank 搜词。纯规则(R3)。

【输入】M3 已给每词 phase_stage + 三时相分（同类目同子分类内）。

【排序与筛选伪代码】
```
STAGE_RANK = {"埋伏期":0,"新兴期":0.5,"起飞期":1,"观望期":2,"冷门期":3,"衰退期":4,"震荡期":4.5,"红海期":5,"伪信号":6}

def recommend_primary(erank_category, subcategory, month, top_n=5):
    rows = query_keywords(erank_category, subcategory, month)  # R1: 类目内
    rows = [r for r in rows if r.phase_stage not in ("伪信号",)]   # 硬门槛
    # 排序: 先按阶段(埋伏优先), 同阶段按 leading 降序, 再 current 降序
    rows.sort(key=lambda r:(STAGE_RANK[r.phase_stage], -(r.leading_score or 0), -r.current_score))
    out = rows[:top_n]
    for r in out: r.reason = build_reason(r)   # 模板生成理由
    return out

def build_reason(r):
    """每个时相阶段都有人话理由模板。实现者:不省略任何阶段。"""
    if r.phase_stage=="埋伏期":
        return f"现状{r.current_score:.0f}低但先导{r.leading_score:.0f}高→埋伏机会,建议入场前搜Top Listings验证"
    if r.phase_stage=="新兴期":
        return f"先导{r.leading_score:.0f}强,现状{r.current_score:.0f}爬坡中,竞争未形成→新兴窗口,建议搜词验证"
    if r.phase_stage=="起飞期":
        return f"先导{r.leading_score:.0f}和现状{r.current_score:.0f}双高但未红海→起飞期,抢占先机"
    if r.phase_stage=="观望期":
        return f"先导{r.leading_score:.0f}现状{r.current_score:.0f}无明确信号→数据不足,建议关注等下一月先导刷新"
    if r.phase_stage=="冷门期":
        return f"先导和现状双低→冷门方向,除非有特殊判断否则建议忽略"
    if r.phase_stage=="红海期":
        return f"现状{r.current_score:.0f}高且已固化(滞后{r.lagging_score:.0f})→红海,除非有差异化切入点否则慎入"
    if r.phase_stage=="衰退期":
        return f"先导低但现状高且固化→见顶/衰退,不建议入场"
    if r.phase_stage=="震荡期":
        return f"先导{r.leading_score:.0f}强但竞争已固化(滞后{r.lagging_score:.0f})→矛盾信号,假阳性风险,建议等Pinterest确认"
    if r.phase_stage=="伪信号":
        return f"季节性暴涨伪装成先导→过滤,不推荐"
    return f"未知阶段→需人工判断"
```

【输出 schema】
```json
{"erank_category":"home_living","subcategory":"wall_decor","month":"2026-05",
 "recommendations":[
   {"keyword":"boho arch mirror","phase_stage":"新兴期","leading_score":88,"current_score":45,
    "lagging_score":20,"reason":"先导88强,现状45爬坡中,竞争未形成→新兴窗口,建议搜词验证","next_action":"去eRank搜此词下载Top Listings"},
   {"keyword":"wall art","phase_stage":"红海期","current_score":83,"reason":"...红海慎入"}
 ]}
```

【API 契约】
```
GET /api/recommend/primary?month=2026-05&erank_category=home_living&subcategory=wall_decor&top_n=5
→ data: 上述 recommendations
```

【验收示例（真实数据）】
```
前置: M3 已分类。裂变词 boho arch mirror=新兴期, wall art=红海期(或观望期)
期望: recommend/primary 返回列表中 boho arch mirror(新兴期) 排在 wall art 之前
      每条带 reason 和 next_action
```

【人工验收步骤】
1. GET /api/recommend/primary，确认 新兴期 词排第一。
2. 确认每条有人话 reason + next_action。
3. 前端 Funnel 页"推荐①"表格渲染，最右列是 [搜词→] 按钮。

【依赖】M3、N0、N1。

---

### N3 — L2 长尾分析 + 推荐②长尾词（LLM探索 + 规则决定）【P1 最高价值最复杂】

【目的】
从用户上传的 eRank Keyword Tool Top Listings CSV 中，挖出新店能切入的长尾关键词。本节点用 LLM 探索语义聚合，规则层做硬门槛(R3 例外: 显式标注用LLM)。

【输入 schema】eRank Keyword Tool 导出 CSV → erank_listings 表(字段已存在):
```
listing_title, age_days, views, daily_views, sales, revenue, hearts, price, month, keyword_id
```

【清洗伪代码】
```
def clean_listings(rows, price_min=15, price_max=60):
    cleaned=[]
    for r in rows:
        if r.price and not (price_min <= r.price <= price_max): r.price_flag="out_of_band"
        if r.views is not None and r.views < 10 and (r.age_days or 0) > 180: continue  # 死listing
        r.title_norm = normalize(r.listing_title)   # 小写/去标点/去停用词
        cleaned.append(r)
    sales_p95 = percentile([r.sales for r in cleaned if r.sales], 95)   # 离群爆款标记(不删)
    for r in cleaned:
        if r.sales and r.sales > sales_p95: r.outlier=True
    return cleaned
```

【长尾提取（三步走，执行顺序严格：规则预提取 → LLM语义聚合 → 规则硬门槛筛选）】
```
STEP1 规则预提取 (n-gram):
  1. 所有 title_norm 提 2-3 词 n-gram
  2. 统计短语频次, 删只出现1次的(噪声)、纯品牌/店名
  3. 每个短语算: 需求=含该短语listing均(hearts+sales); 竞争=含该短语listing数+店铺集中度;
     可切入性=含该短语listing的 age_days 中位数(越低越能切)
     机会分 = 需求/竞争 * 可切入性归一
  4. 产出: 候选长尾短语列表(每短语带需求/竞争/可切入性/机会分)

STEP2 LLM语义聚合 (标注: 本节点用LLM, 不可用→503):
  输入: STEP1产出的候选短语列表 + 原始 title_norm 文本
  prompt: "以下是同一关键词的Top Listings标题数据和预提取的长尾短语,
           找出语义相同但表述不同的意图聚合(如 'wall art bedroom' 和 'wall art for bedroom' 是同一意图),
           输出: 聚合后的长尾意图列表,每个意图含代表短语和成员短语列表"
  → LLM 返回聚合后的意图组

STEP3 规则硬门槛决定 (如果LLM不可用→503，跳过此步，不输出，不降级):
  对 STEP2 每个聚合意图:
    - 成员短语的价格带外占比 > 阈值 → 降权或剔除
    - 成员短语的 age_median 中位数 > longtail_age_median_max(默认180天) → 剔除(不可切入)
    - 最终按综合机会分排序取 top 5-10
```

【输出 schema】
```json
{"primary_keyword":"boho arch mirror","recommendations":[
  {"long_tail":"boho arch mirror bedroom","opportunity_score":82,"demand":"high","competition":"low",
   "enterability":"high","age_median_days":95,"price_band_fit":true,
   "reason":"含该词listing年龄中位数95天=新店能排上, 需求高竞争低","next_action":"去1688找此款实物"},
  {"long_tail":"round boho wall mirror","opportunity_score":71}
]}
```

【API 契约】
```
POST /api/import/erank-listings   (上传CSV, type=erank_listings, 指定 keyword + month)
POST /api/longtail/analyze?keyword=boho+arch+mirror&month=2026-05
→ data: 上述 recommendations
LLM不可用 → 503 {"error":{"code":"llm_unavailable"}}
  // 降级策略(N3): 503硬阻断。N3是核心分析步骤,没有LLM语义聚合→长尾词质量断崖下降。
  //   输出纯规则n-gram结果会误导用户(以为数据完整),不如明确报错让它知道缺了什么。
  //   对比N6.1: N6.1是辅助建议,降级不阻塞流程。N3不同——它是主流程。
```

【验收示例】
```
前置: 上传 boho arch mirror 的 Top Listings CSV (>=20条)
期望:
  - clean_listings 过滤掉死listing和价格带外
  - 输出 5-10 组长尾词, 按 opportunity_score 降序
  - 含 age_median_days 低的长尾词被标 enterability=high 并排前
  - LLM 把 "X for bedroom" 和 "bedroom X" 合并为一个意图
```

【人工验收步骤】
1. 上传一份真实 Top Listings CSV，确认 erank_listings 表入库。
2. longtail/analyze 返回长尾词，人工核对"可切入性高"的词确实是新listing也能排上的。
3. 关掉 LLM key → 确认 503，不返回纯规则降级结果冒充。

【依赖】erank_listings 导入、LLM(DeepSeek)、M1 子分类。

---

### N4/N5/N6 — L3找货 / L4利润 / L5决策（复用 v3.1）【P2】

【目的】把长尾词落地成实物→成本→利润→决策。**本 SPEC 不重新实现，全部复用 v3.1**，只定义数据怎么从 N3 流入。

【N4 L3 找货】
```
输入: N3 长尾词 + 用户在1688找到的货
自动采集(OpenCLI): price/MOQ/seller/材质(visible_attributes)/跨境标签(private_label)
手工录入: 重量(weight_g), 包裹尺寸(parcel_dimensions_mm) — OpenCLI取不到
录入: POST /api/suppliers (1688采购价_rmb, moq, weight_g, customizable, material, ...)  [v3.1已有/扩展]
多供应商取价: 同款多链接取中位数(防极端低价陷阱)
landed_cost: 复用 v3.1 landed_cost 公式

L3.1 — 能力画像匹配 (能力匹配) 🟡P1
> 业务盲区：系统判 新兴期 但那是别人的赛道。
>   例: boho arch mirror → 新兴期，但 Top 10 都是 $45-120 实木镜框，
>   你的 1688 渠道只有 $15-30 铁艺框，完全不匹配。
> 系统必须告诉你"这市场的好机会不是给你的"。

能力画像 (user_capability_profile, config表常驻):
  material_tier:      ["iron","resin","ceramic","fabric"]  # 你能拿到的材质
  craft_level:        "simple" | "moderate" | "complex"     # 工艺复杂度
  price_band_rmb:     [10, 50]    # 1688采购单价带(RMB)
  price_band_usd:     [30, 60]    # Etsy目标售价带(USD). de minimis取消后$30是安全线. ← 行业报告
  weight_range_g:     [100, 800]  # 你能接受的重量(运费敏感)
  moq_tolerance:      50          # 最大起订量
  fragility_tolerance: "medium"   # 能接受的产品易损等级: low/medium/high ← v4.1预留

匹配检查（/api/capability/match, 在 L3 录入成本后自动触发）:
  输入: longtail_keyword + erank_category + subcategory
  取含该长尾词的 Top 20 listing → 分析:
    主流材质: 标题/标签中的材质词频统计(wood/metal/ceramic/canvas/fabric...)
    主流价格带: 前20 listing price 的 P25-P75 区间
    工艺水平: 标题中手工词频("handmade"/"custom"/"hand carved"/"vintage")
  与能力画像对比: 材质重叠度 = |用户材质 ∩ 主流材质| / |主流材质|
                  价格重叠度 = |用户价格带 ∩ 主流价格带| 重叠比例
                  MOQ 可行性 = (主流MOQ均值 <= moq_tolerance)
  综合匹配分 = 0.4*材质重叠 + 0.4*价格重叠 + 0.2*MOQ可行性

  匹配分 < 0.4 → 标记 MISMATCH，降权，不自动 IGNORE(R3)
  (可匹配不代表能做，不匹配也不代表绝不可能——你可能找到新渠道)
  输出: {"capability_score":0.35,"match":"MISMATCH",
         "gap_analysis":"主流实木框($45-120),你的渠道铁艺框($15-30). 材质价格双不匹配",
         "suggestion":"当前渠道不能切这个赛道。如能找到实木/树脂框供应商可重判。"}
```

【N5 L4 利润】= 直接调 v3.1:
```
POST /api/profit/calculate  body:{unit_cost_rmb, sale_price_usd, ...}  [v3.1已验证可用]
→ 毛利率, 4级利润。回填 keywords.margin_potential(补全那个空列)
利润门槛: 毛利率 < profit_margin_min(默认40%，config可调) → 标记弱(不自动IGNORE, 仅信号)
售价检查: sale_price_usd < user_capability_profile.price_band_usd[0] → 标记 价格风险
  (不是 WEAK。R3: 系统标信号，你判断。$28的轻材质高毛利品可能值得做，
   $10的品标 价格风险 让你注意但系统不替你拦。一刀切$30会冤杀边缘产品。)
  售价在[price_band_usd[0], price_band_usd[1]]之外 → 非风险但记录偏差
  售价检查的值写入 config: min_sale_price_usd=30 (来自能力画像, 可调)
```

【N6 L5 决策】= 直接调 v3.1 + 本 SPEC 新增 RESEARCH 诊断端点:
```
信号卡汇总(本SPEC新增聚合端点):
GET /api/decision-card?keyword=&month=
→ data:{phase_stage, leading, current, lagging, longtail_opportunity, margin, suggestion}
  suggestion ∈ {STRONG, WORTH_RESEARCH, WEAK}  // 仅建议, 不替用户决定(R3 + 用户铁律)
用户拍板: POST /api/decisions {keyword, decision_type: BUY/RESEARCH/IGNORE}  [v3.1]
  注意: v3.1 字段名为 decision_type, 前端勿传 outcome(历史bug)
写审计: action_history (v3.1 审计原则)

N6.1 — RESEARCH 诊断端点 (标注: 本节点用LLM, 不可用→降级信息非503)
  触发: 用户做了 RESEARCH 决策后，前端自动调用此端点获取"下一步该干什么"
  输入: keyword + 全漏斗信号(leading/current/lagging/longtail/margin/phase_stage + listing统计)
  LLM prompt (诊断任务, 不是评分/决策):
    "这个关键词被标记为 RESEARCH。以下是全漏斗信号数据:
     {signal_summary}
     请诊断: 数据中缺少什么导致无法判 BUY? 建议用户去哪个节点补什么数据?
     输出JSON: {diagnosis(一句话诊断), missing_data[(字段名,缺失原因)],
              recommended_node(L0/L1/L2/L3/L4/WAIT/NONE),
              action_description(用户可执行的下一步动作描述),
              wait_reason(仅recommended_node=WAIT时: 为什么建议等待/等多久)}"
  LLM 返回 → 前端展示诊断结果 + 回路导航

  LLM不可用时的降级 (本节点与N3不同: 此处是"建议", 非核心分析, 降级不阻塞决策):
    返回: {"diagnosis":"LLM不可用,无法自动诊断","signals":{信号摘要},"recommended_node":"NONE"}
    ← 不是503。N6.1是辅助判断，不应因LLM挂了就阻止用户做下一步决定(R3: LLM不替用户决策)。
    // 降级策略(N6.1): 辅助诊断。用户已经做了RESEARCH决策,这个端点是"接下来去哪"的建议。
    //   LLM挂了用户仍可自己判断下一步。不像N3——没有LLM,长尾分析的核心步骤直接断了。

  推荐节点(recommended_node)的回路方向:
    L0 → 等待下月先导数据, 看 leading 是否突破阈值
    L1 → 确认/修正 eRank 类目数据或重新导入
    L2 → 补 Top Listings CSV 或等更多 listing 积累
    L3 → 去1688找货, 补采购价/重量/MOQ
    L4 → 调整定价/成本参数重算利润
    WAIT → 当前一切条件可, 但需等待(如季节未到、先导在爬升中)
    NONE → 数据已全, 可直接判 BUY/IGNORE (这种情况本应是 BUY 而非 RESEARCH)
```

【验收示例（真实数据）】
```
profit/calculate 已验证: landed_cost_usd=2.01, unit_cost_usd=1.11 (真实可用)
decision-card 期望: boho arch mirror 返回 phase=新兴期, margin填充后, suggestion=WORTH_RESEARCH
decisions 创建: 传 decision_type=RESEARCH → 成功(不是422, 因为字段名对)
RESEARCH诊断: 用户刚Rearch了一个"先导高但没1688成本"的词
  → LLM返回: {diagnosis:"先导信号强但缺少成本数据无法算利润",
              missing_data:[("1688采购价","未录入"),("重量","未录入")],
              recommended_node:"L3", action_description:"去1688搜'boho arch mirror'实物,录入采购价和重量"}
```

【人工验收步骤】
1. profit/calculate 算出毛利率，确认回填 keywords.margin_potential。
2. decision-card 聚合所有信号成一张卡。
3. 提交 decision_type=RESEARCH，确认 200 + action_history 有记录。

【依赖】v3.1 profit/decisions/products/action_history（已存在）。

---

## 3. 前端线框规格（Funnel 主路径）

【全局布局】左侧导航 = 漏斗物理顺序；顶栏 = 类目 + 月份双切换器。
```
┌────────────────────────────────────────────────────────┐
│ Maple Hollow Home   [HOME&LIVING ▾] [2026-05 ▾]         │ 类目+月份切换
├──────────┬─────────────────────────────────────────────┤
│ Import   │  类目: home_living  子分类: [全部 ▾]          │ 二级:子分类筛选
│ Funnel ◄ │  ●━━●━━○━━○━━○  L0 L1 L2 L3 L4              │ 阶段进度条
│ Profit   │                                              │
│ Decide   │  [主内容随选中阶段切换]                       │
│ Products │                                              │
│ Compliance                                              │
└──────────┴─────────────────────────────────────────────┘
```

【Funnel 主页 — L0 先导（可折叠顶部区）】
```
L0 先导信号                                   [刷新 rising]
KEYWORD     Google Rising 暴涨词           Pinterest
wall art    boho wall art(BREAKOUT)        coastal ✓
            arch mirror(+450%)
裂变发现新词: [+ boho arch mirror] [+ coastal print]   ← 点击加入候选池(M2注入②)
```

【Funnel 主页 — 推荐①一级词表】
```
KEYWORD            子分类      时相          现状 先导  动作
boho arch mirror   wall_decor  [ 新兴期 ] 45      88     [搜词→]
wall art           wall_decor  [ RED OCEAN ] 83      8      [忽略]
理由(展开): "boho arch mirror: 先导88强,现状45爬坡中,竞争未形成→新兴窗口,建议搜词验证"
```

【时相标签样式（守 AGENTS.md：禁 emoji 当 UI，用大写文字色块）】
```
[ 埋伏期 ]      浅绿底+黑字     [ 新兴期 ]  浅蓝底+黑字
[ 起飞期 ]     浅黄底+黑字     [ 震荡期 ]  浅橙底+黑字
[ RED OCEAN ]   浅红底+黑字     [ 衰退期 ] 浅红底+黑字
[ 冷门期 ]        浅灰底+黑字     [ 观望期 ]   浅灰底+黑字
[ FALSE SIGNAL ] 浅灰底+黑字
```

【L2 长尾页（点[搜词→]进入）】
```
L2 长尾分析 — 一级词: boho arch mirror
STEP1 去 eRank Keyword Tool 搜此词,下载 Top Listings
STEP2 [拖入 CSV 或点击上传]
─────────────────────────────────────────
长尾词                     机会分 需求 竞争 可切入性  动作
boho arch mirror bedroom    82    高   低   高      [去1688→]
round boho wall mirror      71    中   中   中
[选这些长尾词去1688找货 →]
```

【Decide 页（系统给信号，你拍板）】
```
boho arch mirror bedroom
时相:新兴期 现状45 先导88 长尾机会82 毛利58%
系统建议: WORTH RESEARCH
你的决定: [ BUY ]  [ RESEARCH ]  [ IGNORE ]    ← 守"AI建造者非决策者"
```

【前端硬规则（守 AGENTS.md minimalist-ui）】
```
- 暖白底 #F7F6F3, 纯黑字 #111, 1px 浅灰边
- 禁渐变/阴影/emoji当UI; 字体 Instrument Serif(标题)+Sans(正文)
- 所有表格 overflowX:auto + minWidth, 768/400 两断点
- 导航顺序=漏斗顺序; 每个推荐表最右有"下一步动作"按钮
- api.get/post 已自动解包 data, 禁止 .data.data 二次解包(历史bug)
```

---

## 4. LLM 实现顺序总表（弱 LLM 照此逐卡推进）

| 序 | 卡 | 优先级 | 可独立验收 | 依赖 | 验收核心 |
|----|----|--------|-----------|------|---------|
| 1 | M1 类目体系 | P0 | ✅ | 无 | wall art→wall_decor |
| 2 | M2 候选池 | P0 | ✅ | M1 | 裂变注入不重复 |
| 3 | N0 L0先导 + N0.1 一致性 | P0 | ✅ | M2,pytrends,Pinterest | DIVERGENT不裂变, leading_score有值 |
| 4 | N1 L1赛道(滞后 A/B) | P0 | ✅ | v3.1 signals, erank_listings | wall_decor→同质化策略, lagging_b有值 |
| 5 | M3 时相引擎 | P0 | ✅ | N0,N1 | (88,45,20)→新兴期 |
| 6 | N2 推荐①一级词 | P0 | ✅ | M3 | 新兴期排第一, 理由+next_action |
| 7 | N3 L2长尾 | P1 | ✅ | erank_listings,LLM | 长尾词按机会分排序 |
| 8 | L3.1 能力画像 | P1 | ✅ | N3, suppliers | MISMATCH时capability_score<0.4 |
| 9 | N4/5/6 落地 + 反馈闭环 | P2 | ✅ | v3.1, products | decision_type提交成功, 预测偏差自动算 |
| 10 | P3 复制wedding/jewelry | P3 | ✅ | 全部 | 类目隔离不串, 滞后策略不串 |

**P0 六步完成后 = "eRank数据→leading→时相→推荐①" 完整闭环。** 你看到的不再是 mug 排第一，而是 rising 裂变出的 新兴期 词排第一。

---

## 5. 破坏性变更清单（给后续 LLM，避免用旧术语）

| 变更 | 类型 | 说明 |
|------|------|------|
| keywords 加 subcategory | 新增字段 | 默认 'unassigned'，需跑 M1 reassign 回填 |
| keywords 加 source_origin | 新增字段 | erank/rising_fission/manual |
| keywords 加 phase_stage | 新增字段 | 由 M3 写入，枚举见 M3 |
| 新表 candidate_pool_log | 新增表 | 候选词来源审计 |
| 新表 phase_snapshot | 新增表 | 每月时相快照 |
| 所有计算按 (erank_category,subcategory) 隔离 | 语义变更 | 旧代码只按 erank_category，需下沉到子分类两级(R1) |
| decisions 字段 decision_type | 修正 | 前端历史传 outcome 导致 422，统一用 decision_type |
| trend_signals.source 扩展 | 枚举扩展 | 新增 google_rising、pinterest |
| 新增 config 常量 | 新增 | phase_threshold_hi(65), phase_threshold_lo(35), longtail_age_median_max(180), profit_margin_min(0.40) |
| 新增 /api/decisions/{id}/research-diagnosis | 新增端点 | LLM诊断 RESEARCH 后该去哪补什么数据(N6.1,LLM不可用时降级非503) |
| N0.1 信号一致性校验 | 新增逻辑 | Google rising × Pinterest 交叉验证, DIVERGENT 不裂变(防假阳性) |
| 滞后 拆 A/B 两层 | 重构 | 滞后_A 集中度 + 滞后_B 同质化(Jaccard), 子分类选策略 |
| L3.1 能力画像匹配 | 新增概念 | user_capability_profile + /api/capability/match(赛道≠你能做) |
| products 加反馈闭环字段 | 扩展表 | predicted/actual/delta/reason (阈值校准的基础数据) |
| L3.1 能力画像加 price_band_usd + fragility | 扩展 | 售价带进能力画像(不自动WEAK,R3),易损性v4.1预留 |
| N1 Jaccard 排除 outlier | 修正 | est_sales>P95的爆款listing排除,Jaccard不被outlier绑架 |

---

## 6. 验收总则

每张卡的"人工验收步骤"通过 = 该节点交付。整体验收：
1. P0 六步完成后，eRank 15词→Google rising裂变→滞后→M3时相→N2推荐。新兴期 词排第一，不再 mug 排第一。
2. P1 完成后，上传 Top Listings CSV 能产出按机会分排序的长尾词；L3.1 能力画像反馈赛道不匹配警告。
3. P2 完成后，RESEARCH 后 LLM 诊断该去哪，products 表含 预测偏差/偏差原因。
4. 全程类目隔离：切到 wedding 不会看到 home_living 的词混入(R1)，滞后 策略不跨子分类混淆。

---

> 文档结束。本 SPEC 与 SPEC-v3.1.md、AGENTS.md 配合使用。术语、字段、API 三处一致，弱 LLM 可逐卡实现逐卡验收。

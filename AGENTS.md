# AGENTS.md — 选品工作台

> 本文档供 AI 编程助手（Claude Code / Cursor / Copilot）自动加载。
> 包含项目全部架构决策、设计规范、开发约定。
> 文件位置：仓库根目录，任何 AI 工具进入项目自动读取。

---

## 项目定位

Etsy 选品闭环系统。用户 Lee，零经验起步，第一款产品是「挂钩 (Hook)」。
目标：从多类目科学漏斗（L0先导→L1赛道→L2长尾→L3找货→L4利润→L5决策）实现**预判式选品**，而非追涨杀跌。

---

## 架构决策

```
前端：Vercel（React 19 + Vite + 手写 CSS）→ 免费、全球 CDN
后端：EC2 18.234.173.5（FastAPI :8000）→ 能跑长时间任务、开 Chrome
数据库：SQLite（里程碑1-2）/ PostgreSQL（里程碑3评估迁移）
ORM：SQLAlchemy 2.0 + Alembic 迁移
定时任务：APScheduler（job 持久化到 SQLite）
通信：HTTP REST API（统一 JSON 响应格式）
```

**为什么不分层部署：**
- Vercel Serverless 有 10s 超时，无法做 1688 采集（需要真实浏览器，耗时 30s+）
- OpenCLI 需要 Chrome 进程常驻，只能跑在 EC2
- 前端纯静态，Vercel 免费且快
- Memory Tree + SQLite + Cron 都在 EC2

**为什么前端用 React 19 + Vite（不是纯 HTML/JS）：**
- 状态机可视化、预算分桶看板、4 级利润展示等复杂 UI 用原生 JS 维护成本极高
- React 19 的组件化适合模块化页面（Dashboard/Keyword Explorer/Profit Calculator 等 8+ 页面）
- Vite 构建快，HMR 体验好，部署到 Vercel 零配置
- **约束**：手写 CSS，不用 Tailwind；不用任何 UI 组件库

---

## 设计系统

**唯一风格：minimalist-ui**

```css
/* 核心变量 */
--bg: #F7F6F3;          /* 暖白底色，不是纯白 */
--border: #EAEAEA;      /* 1px 浅灰 */
--text: #111111;        /* 纯黑，不用 #333 */
--accent: #111111;      /* 按钮和强调也是黑色 */
--radius: 8px;          /* 统一圆角 */
```

**硬规则：**
- ❌ 禁止渐变
- ❌ 禁止阴影（box-shadow / drop-shadow）
- ❌ 禁止 emoji 作为 UI 元素
- ❌ 禁止 Inter / Roboto / Arial
- ✅ 字体：Instrument Serif（标题）+ Instrument Sans（正文）
- ✅ 层次靠排版（字号/字重/间距），不靠颜色
- ✅ 按钮纯黑 #111，无阴影
- ✅ 标签：浅色背景 + 大写字母 + 宽间距
- ✅ 移动端：768px / 400px 两个断点，表格横向滚动

**参考文件：** `docs/minimalist-ui-reference.html`

---

## API 契约

后端基础 URL：`http://18.234.173.5:8000/api`

**响应格式（统一）**：
```json
{
  "success": true,
  "data": {...},
  "error": null,
  "meta": {"timestamp": "...", "version": "v4.0"}
}
```

**核心端点**：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/import/erank` | POST | eRank CSV 导入（4 张表） |
| `/api/signals/calculate` | POST | 信号计算（4 维 + 套利） |
| `/api/keywords` | GET | 关键词列表 + 信号分数 + 筛选 |
| `/api/profit/calculate` | POST | 利润计算（4 级利润 + landed_cost 明细） |
| `/api/decisions` | POST | 记录 BUY/RESEARCH/IGNORE |
| `/api/decisions/quality-report` | GET | 决策质量统计（Type A/B/C/D） |
| `/api/products` | POST/GET | 创建/查询产品 |
| `/api/products/{id}/transition` | POST | 状态机转移（强制校验） |
| `/api/products/{id}/compliance-check` | POST | 合规检查录入 |
| `/api/budget/status` | GET | 预算分桶余额 |
| `/api/budget/spend` | POST | 预算扣减（超预算 409） |
| `/api/etsy/sync` | POST | Etsy 手动同步 |
| `/api/etsy/performance` | GET | Listing 真实表现 |
| `/api/dna/dissect` | POST | DNA 拆解（含失败案例） |
| `/api/dna/patterns` | GET | DNA 模式发现 |
| `/api/export/decisions` | GET | CSV 导出 |
| `/api/export/keywords` | GET | CSV 导出 |
| `/api/export/products` | GET | CSV 导出 |

**完整 API 文档**：参见 `SPEC-v3.1.md` 第 9 节

**评分维度：** Demand × Competition × Seasonal × Margin（动态权重）
**增强标记：** 🔥 Momentum / ✅ DNA Match / ⚠️ Lagging / 📈 Demand Gap / 📉 Oversaturated / 🛒 Sourcable / 💰 High Margin

---

## 1688 采集

**方式：** OpenCLI（真实 Chrome 浏览器，非 API 爬虫）
**原因：** 1688 有严格反爬，服务端请求会被拦截
**限制：** 需要用户 Mac 本地运行（OpenCLI 操控本地 Chrome）
**数据流：** 本地采集 → 上传 JSON → 后端入库 → 评分引擎

---

## 合规红线（Etsy 政策）

### 零容忍（suspend 风险）
- ❌ 直接转售 1688 成品（无 design participation）
- ❌ 抄袭他人品牌/设计/IP（迪士尼/漫威/已注册商标）
- ❌ 标 "Handmade" 但实际是 dropshipping
- ❌ 未公开 Production Partner（1688 工厂必须公开）

### 警告级
- ⚠️ 改造仅限换包装/换 logo
- ⚠️ Listing 描述虚假（材质/产地/工艺造假）

### 强制合规检查清单（READY → LISTED 门禁）
每个产品上架前必须通过 4 项检查：

| check_type | 检查内容 | 失败即阻断 |
|------------|---------|-----------|
| `dropshipping_risk` | 是否有设计参与（尺寸/颜色/logo/材质组合至少 1 项） | ✅ |
| `ip_violation` | 是否使用注册商标/版权图案 | ✅ |
| `handmade_claim` | 是否如实标注（非手作必须选 "Designer"） | ✅ |
| `production_partner` | 是否在 Etsy 后台填写 Production Partner | ✅ |

**规则**：4 项全部录入且 status ∈ (pass, warning) 才能转 LISTED。任意 fail → 阻断。

**完整合规文档**：参见 `SPEC-v3.1.md` 第 16 节

---

## 技术栈

```
前端：React 19 + Vite + 手写 CSS（不用 Tailwind，不用 UI 组件库）
后端：Python 3.11+ / FastAPI / Pydantic v2 / SQLAlchemy 2.0 / Alembic
数据库：SQLite（里程碑1-2）/ PostgreSQL（里程碑3评估）
定时任务：APScheduler
采集：OpenCLI（Node.js，真实浏览器）
LLM：DeepSeek（仅 1688 关键词翻译 + DNA 拍摄建议，温度 0.1）
记忆：Memory Tree Pipeline + agentmemory
```

**关键约束**：
- LLM 不做评分、不做最终筛选、不做决策（N3 L2 长尾分析例外：LLM 探索语义聚合候选，规则层做硬门槛筛选）
- 所有金额字段 CNY/USD 通过后缀区分
- 所有时间字段 ISO 8601 字符串
- 所有 JSON 字段应用层序列化/反序列化

---

## 开发约定

### 代码规范
1. **API 优先**：先定义契约，再实现
2. **模块独立**：每个模块有明确的输入/输出 JSON
3. **无状态函数**：后端 API 不依赖 session
4. **文件名小写**：`scoring.py` 不是 `Scoring.py`
5. **类型标注**：所有 Python 函数必须标注参数和返回值类型
6. **Docstring**：每个 public 函数必须有 docstring（中文）

### 业务规则（v3.1 从 SPEC 同步）
7. **状态机原则**：products.state 转换必须通过 `/api/products/{id}/transition`，禁止直接 UPDATE
8. **审计原则**：所有写操作必须同时写 action_history 表
9. **快照原则**：products.effective_fee_pct_snapshot 锁定历史，配置变化不影响已有产品
10. **预算原则**：预算扣减必须经 `/api/budget/spend`，违反硬约束返回 HTTP 409
11. **合规原则**：READY→LISTED 必须有 4 项 compliance_checks 且 status ∈ (pass, warning)
12. **数据可信度**：L1（真实，置信度 1.0）/ L2（估算，0.5，灰色 + ~）/ L3（缺失，0.0，"--"）

### 错误处理标准
```python
class BusinessError(Exception):
    code: str           # "BUDGET_EXCEEDED" / "COMPLIANCE_REQUIRED" / ...
    message: str        # 用户可读
    field: Optional[str] = None
    http_status: int = 400  # 400/409/404/500

# 状态机违反 → 409 BUSINESS_RULE_VIOLATION
# 预算超限 → 409 BUDGET_EXCEEDED
# 合规未通过 → 409 COMPLIANCE_REQUIRED
# 数据校验失败 → 400 INVALID_INPUT
# 资源不存在 → 404 NOT_FOUND
```

### 测试要求
- 核心算法（landed_cost / effective_fee_pct / 4 级利润 / 4 维信号 / decision outcome）：100% 单元测试
- 状态机：所有合法转移 + 所有非法转移都要测
- API 层：每个端点至少 1 个 happy path + 1 个错误用例
- 总覆盖率：后端 ≥ 80%

---

## 当前状态

- [x] 架构决策完成
- [x] 设计风格确定（minimalist-ui）
- [x] 设计预览通过
- [x] SPEC v3.1 冻结
- [x] SPEC v4.0-funnel 冻结（多类目 × 6层漏斗 × 时相引擎，模块化逐卡验收）
- [ ] 里程碑0：现有系统审计（1-2天）
- [ ] 里程碑1：v4.0 P0 — 类目体系 + 候选池 + 时相引擎 → 一级词推荐最小闭环
  - [ ] M1 类目体系（keywords 加 subcategory + 归类规则）
  - [ ] M2 候选池（三来源注入 + rising 裂变回灌）
  - [ ] M3 时相引擎（classify_phase: leading/current/lagging → 生命周期阶段）
  - [ ] N1 赛道层 滞后（头部店铺集中度 + 标题同质化）
  - [ ] N2 推荐①一级关键词（时相 + 规则排序 → 你拿词去 eRank 搜）
- [ ] 里程碑2：v4.0 P1-P2 — 先导 + 长尾能力
  - [ ] N0 先导层（Google rising + Pinterest 手工录入）
  - [ ] N3 长尾分析（n-gram 清洗 + LLM 语义聚合 + 规则可切入性筛选）
- [ ] 里程碑3：v4.0 P3 — 落地闭环（复用 v3.1 利润/决策/状态机）
- [ ] 里程碑4：横向复制漏斗到 wedding / jewelry（多类目验证）

---

## 已知局限 & P1/P2 路线图

> 2026-05-31 GPT-5 审阅 SPEC v4.0 后确认，已冻结不修改 v4.0，仅记录供后续里程碑参考。

### P1（P0 跑通后优先处理）

| # | 局限 | 现象 | 修复方案 |
|---|------|------|---------|
| 1 | M1 子分类规则太弱 | 正则匹配覆盖率低，长尾词大量落入 `unassigned` | 加 embedding 分类器作为 fallback：Rule First → Embedding Fallback |
| 2 | Google Rising 权重过高 | 网红/新闻事件导致 Breakout 虚高，3 周后归零 | P1 实测 pytrends 时序可用性后，评估是否加 `momentum_duration`（连续上升周数） |
| 3 | 离散阈值边界问题 | 64→观望期、65→起飞期，差 1 分跳阶段 | 保留阶段标签，新增 `phase_confidence` 字段（0-1），排序时 Stage Rank + Confidence |

### P2（多类目扩展后处理）

| # | 局限 | 现象 | 修复方案 |
|---|------|------|---------|
| 4 | 缺少季节窗口 | 真正的季节品（Christmas Ornament 等）被当作噪音过滤 | 新增独立季节模块：季节标签 + 提前天数倒计时，不混入时相 phase 枚举 |

### 明确不修

| # | 被拒建议 | 理由 |
|---|---------|------|
| 5 | N3 LLM 不可用时降级返回规则版结果（confidence=0.5） | 违反核心原则："不准确的数据不如没有"。503 至少告诉你"现在别做决策"，降级结果会让你在不知情下做错误决策 |

### P0 已知空转项

| # | 模块 | 说明 |
|---|------|------|
| 6 | prediction_delta 反馈闭环 | 依赖"上架→真实销量→误差→校准"循环，P0 阶段无真实销量数据，闭环空转。P0 只需建好字段结构（prediction_delta, deviation_reason），等 P2 落地后才有数据驱动校准 |

---

**参考文档**：
- `SPEC-v4.0-funnel.md` — v4.0 科学选品漏斗系统（6层漏斗 × 多类目 × 时相引擎）← 当前主 SPEC
- `SPEC-v3.1.md` — v3.1 完整规范（架构/数据/信号/利润/状态机/API/验收）
- `SPEC-v3.0-backup.md` — v3.0 历史版本
- `SPEC-v2.3-backup.md` — v2.3 历史版本
- `docs/minimalist-ui-reference.html` — UI 设计参考

# 🤖 Coding Agent 接入评测

你知道那种感觉吗——把接入任务丢给 Claude Code 或 Cursor，然后看它能不能自己找到入口、看懂文档、写出能跑的代码。这个项目评测的就是这件事：云服务、API 平台和开发者工具对 Coding Agent 的支持到底到不到位。

## 🔧 怎么用

```bash
/coding-agent-fit <docs-url>
```

扔一个文档 URL 进去，出来的是一份中文评测报告。不用看几十页文档自己判断，工具帮你跑探测、读结构、给分数。

## 📊 评分体系：基线 + Agent 双轨

**基线轨（60 分，扣分制）**：文档公开、API ref、鉴权、quickstart、错误码、SDK、无销售墙、changelog 等 13 项标配清单，每项三档判定（完整 0 / 部分 half / 缺失 full），从 60 往下扣。标配做到位不加分，缺失才扣。

**Agent 轨（40 分，R 加分制）**：只计 Agent 增量项，权重按站点类型分套（下面是云服务默认）：

| # | 维度 | 权重 | 关注点 |
|---|------|------|--------|
| A1 | Agent 入口与发现 | 15% | llms.txt、Link header 声明、well-known 发现文件、robots 对 AI bot 友好 |
| A2 | 文档直接可读 | 30% | `.md` 全站可用、OpenAPI 直接可下载、llms 子索引深度、AI Agent 专属章节、URL 可预测 |
| A3 | Agent 工具 | 30% | CLI agent 化（--json / doctor / dry-run）、MCP、Skill、IDE 配置示例 |
| A4 | Agent 专属摩擦 | 15% | 软 404、SPA 壳、反爬拦 AI bot——主要影响 Agent、对人影响很小的摩擦 |
| A5 | 维护信号可跟踪 | 10% | changelog 有 `.md`/RSS 版本、弃用信息程序可查、反馈渠道 Agent 能用 |

**耦合规则**：Agent 实得分 = min(Agent 得分, 基线得分 × 2/3)——基础文档不达标时，辅助工具的加分同步受限。

等级这样读：D / F 表示基础文档有缺失项，C 表示基础完整但缺少 Agent 投入，B 及以上比的是 Agent 轨得分。横向比较服务时看 Agent 得分，比总分信息量大。完整标准见 [references/rubric.md](references/rubric.md)。

## 🔍 评测流程

**1. 明确接入目标** —— 先搞清楚这家服务是干什么的，Agent 最可能卡在哪一步。别一上来就跑脚本。

**2. 发现证据 + 探测** —— 运行 `scripts/probe.py`，它会自动检查下面这些东西。不依赖第三方库，Python 3 标准库直接跑：

```bash
python3 scripts/probe.py <docs-url>
```

脚本覆盖：
- llms.txt / llms-full.txt（根路径 + 文档挂载路径 + 子域 docs./developer./api.）
- `.md` 页面、`index.md` 和 `Accept: text/markdown` 内容协商
- OpenAPI / Swagger / API Catalog / GraphQL 嗅探
- MCP server / agent skills index / 各 well-known 发现文件
- sitemap、robots.txt、mint.json、security.txt、OAuth well-known
- 响应头里的 `Link: rel="llms-txt"`、`X-Llms-Txt`
- TLS 默认严格校验，证书异常会标记 `tls_insecure: true`

脚本输出包含 `summary` 字段——按 rubric 维度整理好的证据概览，评分时优先参考 summary。stdout 保持精简（无内容预览），每次尝试的完整明细写入临时 JSON 文件，路径在输出的 `probes_file` 字段里。探测被拦（403 等）和真 404 在 summary 里分开标记，SPA 壳页面会被识别并标记。

如果脚本漏了什么，手动补一轮站内搜索。

**3. 核对评分依据** —— 按 checklist 逐项记录证据 URL，别空口给分。

**4. 评分** —— 读 `references/rubric.md`，按维度打分，写清楚判断理由。

**4.5 真实接入流程实跑** —— 选一条接入路径（REST / SDK / CLI / MCP / dashboard，看用户需求和站点强项）真的跑一遍，记录每一步通过 / 卡住 / 跳过。人机通用卡点回写基线清单，Agent 专属卡点回写 A4 摩擦项。会发邮件 / 下单 / 部署的步骤必须用户明确同意，优先 sandbox / test 域。

**5. 改进建议** —— 每条建议包含：我们看到的（事实 + 影响）、改进方向（站点可以做什么 + 衡量标准）、优先级、工作量、预期提分（R 变化具体到小数）。低置信度维度的建议优先排前。

## 📋 报告里有什么

- 双分数：基线 x/60 + Agent y/40，加总分和等级
- Agent 接入把握（高 / 中 / 低）—— 这个结论直接告诉开发者能不能放心把接入任务交给 Agent
- 需要人工介入的关键点
- 完整接入路径（从发现入口到跑通第一个调用）
- 各维度评分、证据和判断理由
- **最该先改的 3 件事**——每条含"我们看到的"和"改进方向"，可直接作为站点优化清单

## ⚠️ 关键判断规则

这些是评分时容易踩进去的坑，先写清楚：

- CLI、MCP、Skill 是加分项，但不能替代完整的 API 文档。文档烂，辅助工具再多也救不回来。
- 有成熟的 CLI 但没有 MCP？不因为这个自动扣分。
- 云服务如果缺 API reference、错误码、限流说明或版本说明，就算有 CLI/MCP，该扣的分照扣。
- 判定 llms.txt 缺失之前，必须把根路径、文档挂载路径和响应头全都查一遍。

## 📁 目录结构

```
coding-agent-fit/
  SKILL.md          # 核心指令
  scripts/          # probe.py 探测脚本
  references/       # rubric.md 评分标准 + report_template.md 报告模板
  evals/            # 评测样例
```

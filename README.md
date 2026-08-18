# LP Insights Pipeline

落地页（Landing Page）测试洞察流水线 — 一套面向 Google Ads LP 测试的可复用 AI Agent Skills。

把 LP 测试分析拆解为三个环节，每个环节一个 skill，覆盖从拉数到结论到可信度的全链路：

| # | Skill | 环节 | 解决什么问题 |
|---|-------|------|--------------|
| 1 | `lp-test-analysis` | 口径与结论规范 | 对比口径怎么定（同广告组内 变体 vs 基线）、输出什么格式、时间窗怎么切、Insight 怎么组织、置信度怎么标 |
| 2 | `google-ads-query-playbook` | 数据质量手册 | Google Ads API 拉数的 6 类已知陷阱与标准解法（名称匹配、行数截断、报表口径 vs 配置口径等），保证进入分析的数据是对的 |
| 3 | `ads-account-audit` | 操作可审计 | 账户变更来源排查：变更日志解读 + agent 调用记录自证 + 交叉验证，保证 AI 协作全程可证明、可追责 |

## 设计理念

**口径规格化 → 拉数标准化 → 操作可审计。**

- 把管理层的验收标准逆向工程成可执行规格（skill 1）
- 把踩过的数据坑固化为资产，不再重复付学费（skill 2）
- AI 参与投放工作必须可审计：只读/写操作分离、日志双向验证（skill 3）

## 安装

将三个 skill 文件夹复制到你的 skills 目录：

- **Craft Agent（工作区级）**: `~/.craft-agent/workspaces/{workspace}/skills/`
- **Claude Code（全局）**: `~/.agents/skills/`
- **Claude Code（项目级）**: `{project}/.agents/skills/`

每个 skill 文件夹包含 `SKILL.md`（指令定义）和 `icon.svg`（图标）。

## 使用

- 更新一版 LP 测试结果：`[skill:lp-test-analysis]` + 说明测试页面
- 任何 Google Ads 拉数任务：叠加 `[skill:google-ads-query-playbook]`
- 排查"账户是谁动的"：`[skill:ads-account-audit]`

## 迁移性

方法论不绑定 Google Ads：换成 Meta / TikTok 或任何投放平台，同样是"口径规格化、拉数标准化、操作可审计"三件事，只需替换 skill 2 中平台特定的陷阱清单。

---

*Built from real-world LP testing practice (Meshy paid ads, 2026)。*

---
name: "账户变更审计"
description: "排查 Google Ads 账户变更来源：change_event 日志解读、agent 调用记录自证、交叉验证与时区推断"
requiredSources:
  - google-ads
alwaysAllow: ["Bash"]
---

# 账户变更审计

> Part of **LP Insights Pipeline** — 落地页测试洞察流水线（3/3：操作可审计）

用于回答"这个改动是谁做的 / 是不是 agent 做的"这类问题。

## 第一步：拉账户侧变更日志

```
SELECT change_event.change_date_time, change_event.user_email,
  change_event.client_type, change_event.change_resource_type,
  change_event.resource_change_operation, change_event.campaign
FROM change_event
WHERE change_event.change_date_time >= 'YYYY-MM-DD HH:MM:SS'
  AND change_event.change_date_time <= 'YYYY-MM-DD HH:MM:SS'
ORDER BY change_event.change_date_time DESC LIMIT 200
```

- 必须带起止时间，LIMIT ≤ 10000
- 时间戳是**账户时区**（本账户 Asia/Shanghai = 北京时间）
- `client_type` 关键值：2=网页端人工、3=自动规则、4=Google Ads Scripts、6=**外部程序经 API**、7=Ads Editor
- `resource_change_operation`：2=CREATE、3=UPDATE、4=REMOVE
- `change_resource_type` 常见：2=AD、3=AD_GROUP、5=CAMPAIGN、14=ASSET、16=CAMPAIGN_ASSET
- campaign 资源名里的 ID 再查一次 campaign.name 落实到具体 campaign

## 第二步：拉 agent 自己的调用记录（自证清白/有责）

会话日志在 `C:\Users\yueli\.claude\projects\{project-slug}\*.jsonl`。解析要点：

- 找 `"type": "tool_use"` 且 name 以 `mcp__google-ads__` 开头的块
- 只读工具 = `search`、`list_accessible_customers`；**其余全部是写操作**（update_/create_/add_/remove_/replace_）
- 时间戳是 UTC，+8 换算北京时间
- 横扫全部会话：`grep -o '"name": *"mcp__google-ads__[a-z_]*"' */*.jsonl | sort | uniq -c`（注意引号后可能有空格）
- transform_data 只能读会话目录内文件，日志需先复制进 data/ 再解析，用完删除

## 第三步：交叉验证与归因

- 双向闭环：agent 侧零写操作 + 账户侧每条变更都有归属 → 排除 agent
- **时间窗对照**：变更发生时段 agent 是否有任何活动
- **时区推断**：北京凌晨 5–9 点 = 美国前一天下午（PT -15h / ET -12h），可提示操作方在哪个时区工作
- 同一秒内多条连续变更 = 程序执行，人工做不到

## 表述纪律（重要）

- 区分**事实**（日志记录的邮箱/入口/时间）与**推断**（"是脚本"、"是某人"），推断必须标注
- 共享凭据（如 team@yourcompany.com 这类团队账号）的 API 变更只能定位到"该凭据经 API"，**无法定位到具体程序/人**；进一步归因需查 Google Cloud Console 的 OAuth client 调用记录或内部确认
- 变更内容要验证到实体级（如查 asset 的实际文案/折扣/档期），不能只对动作类型

---
name: "Google Ads 拉数避坑手册"
description: "Meshy Google Ads 账户拉数的已知陷阱与标准解法：双空格名称、字段引用、行数截断、JSON 解析、URL 分桶"
requiredSources:
  - google-ads
---

# Google Ads 拉数避坑手册

> Part of **LP Insights Pipeline** — 落地页测试洞察流水线（2/3：数据质量手册）

账户：`<ACCOUNT_ID>`（填你自己的 Google Ads 账户；币种 CNY，cost_micros ÷ 1e6 = ¥；时区 Asia/Shanghai，报表时间戳都是北京时间）。

## 陷阱 1：Campaign 名称含双空格

部分 campaign 名有连续两个空格（如 `Brand - US - NB - Generic  - Exact`，"Generic" 后双空格）。用名称做 `IN`/`=` 筛选会**静默返回空结果**。

**标准解法**：先用 `campaign.name LIKE 'Brand%'` 等宽松条件拉全量 id+name 列表，人工核对后**一律按 campaign.id 过滤**。

## 陷阱 2：landing_page_view 的字段引用要求

按 `campaign.id` / `ad_group.id` 过滤 `landing_page_view` 时，这些字段必须同时出现在 SELECT 里，否则报 `EXPECTED_REFERENCED_FIELD_IN_SELECT_CLAUSE`。

## 陷阱 3：5000 行截断

按天分段（segments.date）的拉数很容易撞 5000 行上限，且**不报错、静默截断**（症状：某些 campaign 数据莫名缺失）。

**标准解法**：每次检查返回行数，逼近上限就视为不可靠；用两步法缩小范围——第一步只拉目标 LP 的行找出 serving ad group IDs，第二步只拉这些 ad group 的明细。

## 陷阱 4：MCP 返回是拼接 JSON

返回体是多个 pretty-printed JSON 对象直接连接，不是 JSON 数组。解析用扫描循环：

```python
dec = json.JSONDecoder(); i = 0; recs = []
while i < len(raw):
    while i < len(raw) and raw[i] in " \t\r\n,": i += 1
    if i >= len(raw): break
    obj, j = dec.raw_decode(raw, i); recs.append(obj); i = j
```

## 陷阱 5：URL 分桶

- 本地化路径要剥离（/pt-BR/、/it/、/ja/ 等语言前缀后再判断页面）
- `utm_content=3d-printing` 是**关键词标签**，挂在主页 URL 上不代表落地页是 /3d-printing
- 排除 blog/docs/tutorials/3d-tools/features/tags/pricing/collections 等非 LP 路径
- 配置里的 final_urls 是字符串形式的 `"['https://...']"`，用正则 `https?://[^'\"\]]+` 提取

## 陷阱 6：活动报表 ≠ 配置状态

`landing_page_view` 是活动报表：没展示的日子/组合**不存在行**。因此：
- "末次投放日很久以前" ≠ 已停投（可能只是没量）——判断在投状态查 `ad_group_ad.status`（2=ENABLED, 3=PAUSED）
- "首投日" ≠ 广告组创建日
- 配置了但从未跑出展示的组合在报表里完全不可见，做"全面版"要从 ad_group_ad 配置侧补全

## 其他

- MCP 重连后必须重新 Read 一次 source guide，否则调用被拒
- 每次查询加 LIMIT、加 `status != 'REMOVED'`、按 cost 降序
- 变更日志查 `change_event`（必须带有限日期范围 + LIMIT ≤ 10000）

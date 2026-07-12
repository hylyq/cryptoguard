---
name: stale-price-data-on-websocket-disconnect
description: /pm price 在 WebSocket 断连时返回冻结的过期价格
metadata:
  type: project
---

# 问题：WebSocket 断连导致价格查询返回过期数据

## 根因

`/pm price` 命令不发起 HTTP 请求，而是读取 `OKXClient._prices` 内存字典。
该字典仅由 WebSocket `tickers` 频道推送更新（`okx_client.py:123`）。

当 OKX WebSocket 断开时：
1. `_prices` 保留断连前的最后价格
2. `_cmd_price()` 只检查 `get_ticker() is None`，不检查数据新鲜度
3. 结果：用户持续看到冻结的同一价格

额外隐患：`TickerData.from_okx()` 未被 try-except 包裹，OKX 异常消息格式可能炸毁 WebSocket 读循环。

## 修复内容 (2026-07-12)

1. **`okx_client.py`** — WebSocket 断开时清空 `_prices`，防止服务过期数据
2. **`okx_client.py`** — `TickerData.from_okx()` 包裹 try-except，防止异常消息炸毁连接
3. **`commands.py`** — `_cmd_price` 增加 30 秒数据新鲜度检查，过期时显示警告
4. **`tools.py`** — `execute_get_current_price` 改用 `get_ticker()` 获取完整 TickerData；`execute_get_ticker_detail` 新增新鲜度警告；提取公共 `_staleness_suffix()` 函数

**Why:** WebSocket 断连是常态（网络波动、OKX 踢连接），必须在查询层检测并告知用户数据可能过期。

**How to apply:** 无需额外配置。下次 WebSocket 断连时，`_prices` 会被自动清空，查询将返回"正在订阅"而非过期价格。新鲜度警告在数据超过 30 秒时自动附加到返回消息末尾。

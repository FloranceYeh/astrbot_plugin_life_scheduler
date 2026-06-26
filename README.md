# AstrBot Life Scheduler Plugin

为 AstrBot 设计的拟人化生活日程插件。利用 LLM 根据日期、节日、历史日程和近期对话，自动生成每日穿搭和日程安排，并注入到 System Prompt，让 Bot 拥有连续的"生活"状态。

## 功能特性

- **日程生成**: 结合日期、节日、历史日程和近期对话，生成拟人化日程
- **穿搭推荐**: 根据创意池随机选取风格，生成每日穿搭描述
- **按需状态注入**: 默认只在相关对话中注入生活状态，并追加到用户消息尾部，避免每轮改写 System Prompt
- **当前状态提取**: 对含时间点的日程自动提取当前或最近活动，减少对话中说出与日程冲突的状态
- **懒加载**: 未到生成时间时，首次对话自动触发生成
- **补充要求**: 重写日程时可附加自定义要求，让生成更符合预期

## 安装

```bash
pip install holidays APScheduler
```

## 指令列表

| 指令 | 权限 | 说明 |
| :--- | :--- | :--- |
| `查看日程` | 所有人 | 查看今日日程和穿搭 |
| `重写日程` | 管理员 | 重新生成今日日程 |
| `重写日程 <补充要求>` | 管理员 | 带补充要求重新生成，例如：`重写日程 今天穿黑色连衣裙，安排一个下午茶` |
| `日程时间 <HH:MM>` | 管理员 | 设置每日自动生成时间 |

别名：`life show`、`life renew`、`life time`

## 配置项

| 字段 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `schedule_time` | string | `07:00` | 每日自动生成日程的时间 |
| `reference_history_days` | int | `3` | 生成时参考的历史日程天数 (1-7) |
| `reference_recent_count` | int | `10` | 生成时参考的近期会话数量，0 表示不参考 |
| `life_context_injection_mode` | string | `auto` | 生活状态注入策略：`auto`、`compact`、`full`、`off` |
| `life_context_injection_target` | string | `user_prompt` | 注入位置：默认追加到用户消息尾部；可设为 `system_prompt` 恢复旧行为 |
| `life_context_max_schedule_chars` | int | `1200` | `full` 模式或旧式完整日程注入时的最大日程字符数 |
| `pool` | object | - | 创意池，每次生成随机选取 |
| `prompt_template` | text | - | LLM 生成日程的 Prompt 模板 |

### 创意池 (pool)

每次生成日程时，从各池随机选取一项融入提示词：

| 池 | 示例 |
| :--- | :--- |
| `daily_themes` | 探索日、社交日、宅家日、工作日、运动日... |
| `mood_colors` | 慵懒、活力、优雅、俏皮、温柔、冷艳... |
| `outfit_styles` | 知性学院风、街头休闲风、温柔淑女风、酷飒中性风... |
| `schedule_types` | 户外活动型、社交聚会型、独处充电型、随性漫游型... |

### Prompt 模板占位符

| 占位符 | 说明 |
| :--- | :--- |
| `{date_str}` | 日期，如 2026年01月22日 |
| `{weekday}` | 星期几 |
| `{holiday}` | 节日信息（中国节日） |
| `{persona_desc}` | Bot 人设描述 |
| `{daily_theme}` | 从创意池选取的今日主题 |
| `{mood_color}` | 从创意池选取的心情色彩 |
| `{outfit_style}` | 从创意池选取的穿搭风格 |
| `{schedule_type}` | 从创意池选取的日程类型 |
| `{history_schedules}` | 历史日程记录 |
| `{recent_chats}` | 近期对话记录 |

## 注入机制

插件默认使用 `auto` 策略：用户问题涉及日程、穿搭、当前状态、自拍/生图等生活状态时才注入短状态；普通闲聊不注入，也不会触发懒生成。完整日程不再常驻上下文，LLM 需要时可调用 `get_life_schedule_detail` 工具查询。

默认注入到用户消息尾部，保持原始 System Prompt 稳定，利于模型端 prompt cache。若需要旧行为，可把 `life_context_injection_target` 改为 `system_prompt`，或把 `life_context_injection_mode` 改为 `full`。

相关对话中默认只注入类似：

```xml
<character_state>
时间: 下午
当前状态: 14:00 在咖啡厅看书...
穿着: 白色针织衫搭配米色阔腿裤...
今日日程: 上午整理房间，下午去咖啡厅看书...
</character_state>
```

### LLM 工具

插件注册了 `get_life_schedule_detail` 工具，供 LLM 在用户询问完整日程、今天安排、穿搭或当前生活状态时按需查询。工具返回当前业务日的日期、穿搭风格、穿着、当前状态和完整详细日程。

## 外部插件调用

插件提供 `get_life_context()` 公共方法，供生图、每日分享等外部插件读取当前全局日程：

```python
life_data = await life_scheduler_plugin.get_life_context()
```

返回数据包含 `outfit`、`schedule`、`meta.style` 和 `timeline`。本插件当前保持全局日程语义，不按人格或会话隔离。

## 注意事项

1. 每日生成消耗 LLM Token，参考天数和对话数越多消耗越大
2. 需安装 `holidays` 库才能识别中国节日
3. 重写日程会覆盖当日已有数据



本插件开发QQ群：215532038

<img width="1284" height="2289" alt="qrcode_1767584668806" src="https://github.com/user-attachments/assets/113ccf60-044a-47f3-ac8f-432ae05f89ee" />

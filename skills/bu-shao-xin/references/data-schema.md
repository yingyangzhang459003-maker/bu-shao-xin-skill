# 数据表规范（家庭档案的结构）

**为什么用固定表结构，不用自由文本**：自由文本每次写的不一样，下次读的时候要重新理解。表结构固定后，`risk_flags` 这种关键字段能一眼看到，不会埋在段落里。

---

## 存储位置

```
data/memory/
├── records/
│   ├── _schema.json          空模板（复制它新建）
│   ├── family-001.json       每户一个文件
│   └── family-002.json
└── sessions/
    └── 2026-10-28-family-001-01.md    每次对话一份流水（人读）
```

**JSON 是主数据**，sessions 的 markdown 是给人读的流水（只在归档时写一次，之后不改）。
**不存在第二份画像副本**——不要在 `data/memory/` 下任何地方另建一份同样的画像。两处同步必然烂尾，下次会读到旧的那份。

---

## 表 1｜孩子基本表 `children[]`

| 字段 | 类型 | 说明 | 何时填 |
|---|---|---|---|
| `child_id` | string | c1 / c2 | 建档 |
| `nickname` | string | **代称，不写真名** | 建档 |
| `age` | number | 周岁 | 首次 |
| `grade` | string | 如"初三" | 首次 |
| `gender` | string | 男/女 | 首次 |
| `personality` | string[] | 性格标签，如 `["文静","慢热","话少"]` | 首次 |
| `health` | object | `{sleep, medication, chronic, development}` | 首次+补充 |
| **`risk_flags`** | string[] | **红旗标记，见下表** | **每次对话都要检查** |
| `notes` | string | 其他要留意的 | 随时 |

### risk_flags 合法值（只能从这几个里选）

| 值 | 含义 | 触发动作 |
|---|---|---|
| `self_harm` | 自残史或当前行为 | 追问时机、就医建议 |
| `suicidal_ideation` | 提到不想活/消失 | **当天就医，不过夜** |
| `eating_disorder` | 进食异常 | 就医 |
| `school_refusal` | 拒绝上学 | 查学校端 |
| `bullying` | 疑似被欺凌 | 优先处理 |
| `substance` | 烟酒等 | 观察 |
| `none` | 已确认无 | — |

**没有评过就留空数组 `[]`，不要写 `["none"]`。** 空 = 未评估，`none` = 评估过且没有。这两者不一样。

---

## 表 2｜家庭结构 `household`

| 字段 | 类型 | 说明 |
|---|---|---|
| `caregivers` | string[] | 主要照养人 |
| `parental_consistency` | string | `一致` / `有分歧` / `严重分歧` |
| `grandparents_live_in` | boolean | 老人是否同住 |
| `grandparents_interfere` | string | `不干涉` / `偶尔护` / `经常护` |
| `family_time_daily` | string | 一天有无共同在场时段，如"仅晚饭" |
| `major_events` | string[] | 转学、二胎、搬家、离婚、重病 |

> `grandparents_interfere` 和 `parental_consistency` 直接决定方案能不能落地，必填。

---

## 表 3｜问题表现 `presenting_problem`

| 字段 | 类型 | 说明 |
|---|---|---|
| `primary_issue` | string | **只写一件最痛的** |
| `onset` | string | 什么时候开始的 |
| `onset_pattern` | enum | `sudden`（突然）/ `chronic`（一直） |
| `frequency` | string | 多久一次 |
| `typical_scene` | string | 最近一次的完整描述 |
| `child_quote` | string | **孩子的原话**，没有就留空 |
| `parent_interpretation` | string | 家长怎么解释这件事（**这是他的判断，不是事实**） |

> `parent_interpretation` 单独存，是为了区分"事实"和"家长的归因"。后面要检验这个归因对不对。

---

## 表 4｜已尝试方案 `attempts[]`

| 字段 | 类型 | 说明 |
|---|---|---|
| `date` | string | |
| `method` | string | 具体做了什么 |
| `effect` | enum | `有效` / `部分有效` / `无效` / `更糟` |
| `child_reaction` | string | 孩子的反应 |
| `conclusion` | string | 以后还用不用 |

**这张表是最值钱的**。同一方法给过两次都失败 → 永不推荐。

---

## 表 5｜家长状态 `parent`

| 字段 | 类型 | 说明 |
|---|---|---|
| `main_worry` | string | 最担心什么（**常常是错位的，要判断**） |
| `available_time` | string | 每天能投入多少 |
| `sustainability_days` | number | **能坚持几天 → 方案周期上限** |
| `emotional_state` | string | 焦虑/疲惫/已放弃 |
| `communication_status` | enum | `通畅` / `困难` / `已断` |

> `sustainability_days` 决定方案长度。给一个只能坚持三天的人两周方案，必然失败。

---

## 表 6｜会话记录 `sessions[]`

| 字段 | 类型 | 说明 |
|---|---|---|
| `date` | string | |
| `trigger` | string | 这次为什么来 |
| `judgment` | string | 我的判断（一句话） |
| `actions_given` | string[] | 给了哪些动作 |
| `prediction` | string | **预判会怎样**（飞轮轴心） |
| `followup` | object | `{date, done, effect, blocked_at}` |
| `changed_judgment` | boolean | 判断变了吗 |

---

## 表 7｜红线 `redlines[]`

从 `risk_flags` 自动生成，写在档案最前面。

```json
"redlines": [
  {"condition": "发现新的伤口", "action": "这周就医", "level": "urgent"},
  {"condition": "说出不想活", "action": "当天就医，不过夜", "level": "critical"}
]
```

---

## 记录规则（四条）

1. **区分事实与判断**。家长说"她啥也不干"是**他的判断**，不是事实。存进 `parent_interpretation`，不存进 `typical_scene`。
2. **区分转述与亲见**。老师说孩子自残 = `L3 转述`，要标注来源。这决定可信度。
3. **每条带日期**。没有日期的记录，三个月后就没用。
4. **不覆盖历史**。判断改了 → 在 `sessions[]` 里加一条 `changed_judgment: true`，原记录保留。

---

## 每次对话后更新哪张表

| 表 | 更新时机 |
|---|---|
| children | 有新信息时 |
| presenting_problem | 问题变了时 |
| attempts | 给了新方案后 |
| parent | 观察到能力边界时 |
| sessions | **每次必加一条** |
| redlines | risk_flags 变化时 |

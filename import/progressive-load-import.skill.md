---
name: progressive-load-import
description: Read the current user's training history from the source app, normalize exercise names through the ProgressiveLoad API, generate a ProgressiveLoad external-import JSON (v1), and validate it.
---

# 渐进负荷 · 外部训练记录 → 可导入 JSON

> 本文件是完整 skill，不依赖本机仓库、`references/`、`scripts/` 或其他文件。
> 353 个内置动作表和名称归一规则在服务端，不在本文件中。

## 你的任务

用你所在训练 App 的服务器/内部接口，读取当前登录用户的训练记录，转换成**一个**符合下方格式的 JSON 文件。不要要求用户导出/上传/截图，不要向用户索要密码或训练 App 的 Token，不要编造训练数据。

## 第 0 步：能力检查

确认你能够：

1. 读取当前登录用户的**已完成**力量训练记录，并细分到组（日期、动作、每组重量、次数）；
2. 把动作 ID 解析成可读的动作名；
3. 确定重量单位（kg / lb / jin）。

任何一项做不到，立即停止，只回复：

> 当前 agent 无法读取本 App 的训练数据（或无法拿到按组的记录），因此不能生成导入文件；请不要根据猜测生成。

## 第 1 步：读取原始训练记录

按训练日组织：

- 本地日期：`YYYY-MM-DD`
- 动作名称
- 每个动作的组：重量、次数
- 有氧分钟数和类型（如果有）
- 重量单位

不要保留源 App 的训练时间、组号、RPE、容量、备注。

## 第 2 步：调用归一 API（必须做）

在生成最终 JSON 前，把所有动作名发给 ProgressiveLoad API，拿到内置动作数据。

接口：

```text
POST https://api.progressiveload.com/normalize
Authorization: Bearer 511075df9c72531edfcbed5eb270ddbf5170b98945d3e691
Content-Type: application/json
```

请求：

```json
{
  "exercises": [
    { "name": "哈克机深蹲", "equipment": null },
    { "name": "上斜哑铃卧推", "equipment": null }
  ]
}
```

每次最多 500 个动作。相同名称+器械只需请求一次。

返回：

```json
{
  "results": [
    {
      "source": "哈克机深蹲",
      "matched": true,
      "code": "exercise.machine_hack_squat",
      "name": "哈克深蹲",
      "nameEn": "Hack Squat",
      "equipment": "strength_fixed_hack_squat_026",
      "equipmentName": "哈克深蹲机",
      "muscle": "股四头肌",
      "limbType": 1,
      "jointType": 0
    }
  ]
}
```

### 怎么使用返回结果

- `matched: true`：最终动作写 API 返回的 `code`、`name`、`equipment`、`muscle`；可选写 `limbType`/`jointType`。**不要再自己改动作名。**
- `matched: false`：保留源动作名，省略 `code`，按自定义动作处理；`muscle` 用源数据或按下方肌群表填写。
- 如果 API 返回 401、404、超时或格式错误：**停止生成**，只回复一句“动作归一服务暂时不可用，请稍后重试”。不要猜测内置动作，也不要自己编 code。
- 不要把 API Token 输出给用户；不要打印请求头。

## 第 3 步：生成 JSON

顶层只允许这五个键：

```json
{
  "format": "progressive_load.external_training_log",
  "version": 1,
  "source": "hevy",
  "weightUnit": "kg",
  "workouts": []
}
```

### workout

```json
{
  "date": "2026-10-20",
  "exercises": [],
  "cardioMinutes": 30,
  "cardioNote": "30 分钟跑步机"
}
```

- 每个日期只能出现一次；多个训练合并成一天。
- 日期不能是未来。
- 没有力量动作、只有有氧时，`exercises` 必须是空数组。
- 没有有氧时省略 `cardioMinutes` 和 `cardioNote`。

### exercise

```json
{
  "name": "哈克深蹲",
  "code": "exercise.machine_hack_squat",
  "equipment": "strength_fixed_hack_squat_026",
  "muscle": "股四头肌",
  "limbType": 1,
  "jointType": 0,
  "sets": [
    { "weight": 80, "reps": 12 }
  ]
}
```

- `name`：内置动作使用 API 返回的 `name`；未匹配时使用源名称。
- `code`：只有 API `matched: true` 时才写；否则省略。
- `muscle`：内置动作使用 API 返回的 `muscle`；自定义动作用下方标准肌群名。
- `equipment`：使用 API 返回的器械 code；自定义动作不确定就省略。
- `limbType`/`jointType`：优先用 API 返回值；不确定可省略。
- 同一天不能重复动作名；同一天的同类动作合并 `sets`。
- 每组只能有 `weight` / `reps`。
- `weight` 0–1000；`reps` 1–1000；自重动作 `weight` 写 0。

### set

```json
{ "weight": 60, "reps": 10 }
```

只允许 `weight` / `reps`，不要 `volume`、RPE、组号、时间、备注、热身标记。

### 有氧

`cardioMinutes` 按天汇总；有分钟数时 `cardioNote` 必填。

格式：

```text
30 分钟跑步机
20 分钟椭圆机，15 分钟单车
```

多个类型用中文逗号“，”分隔；备注里的分钟数之和必须等于 `cardioMinutes`。

## 肌群表（自定义动作用）

| muscle | English / 常见叫法 |
|---|---|
| 胸大肌 | Chest |
| 胸大肌上束 | Upper chest |
| 胸大肌下束 | Lower chest |
| 背阔肌 | Lats |
| 斜方肌上束 | Upper traps |
| 斜方肌中束 | Middle traps |
| 斜方肌下束 | Lower traps |
| 竖脊肌 | Erector spinae |
| 股四头肌 | Quads |
| 腘绳肌 | Hamstrings |
| 臀大肌 | Glutes |
| 臀中肌 | Glute medius |
| 内收肌群 | Adductors |
| 小腿三头肌 | Calves |
| 比目鱼肌 | Soleus |
| 三角肌前束 | Front delts |
| 三角肌中束 | Side delts |
| 三角肌后束 | Rear delts |
| 肱二头肌 | Biceps |
| 肱三头肌 | Triceps |
| 前臂 | Forearms |
| 腹直肌 | Abs |
| 腹斜肌 | Obliques |
| 核心肌群 | Core |

## 输出（固定，只允许这三部分）

1. 一个 `.json` 文件，文件名 `progressive_load_import_YYYYMMDD.json`；优先作为文件附件，平台只能发文本时输出完整 JSON 代码块。

2. 一行范围摘要，必须用普通文本，不要放进代码块。按最终 JSON 实际数据计算：

本次导出：YYYY-MM-DD 至 YYYY-MM-DD，共 N 个训练天。

`N` 是最终 `workouts` 的条数；日期取最早和最晚的 `date`。只显示这一行，不显示组数、动作数、容量等。

3. 下面这段固定导入说明。必须用普通文本逐行输出，不要放进代码块、引用块或 JSON 字符串：

导入方法：

1. 先把上面的 JSON 文件保存到手机或平板，文件名要以 .json 结尾。
2. 打开「渐进负荷」App。
3. 第一次打开 App：在引导页走到「导入历史」这一步，点“选择文件”，选中刚才的 JSON。
4. 已经进入 App：点底部「我的」→「数据」→「导入其他 App 记录」，再选中 JSON。
5. 不要点「导入文件」，那个会替换全部数据；要选「导入其他 App 记录」。
6. 如果收到的是文字而不是文件：先把整段 JSON 保存成 .json 文件，再按上面的步骤操作。
7. 等待“正在导入中”结束。显示“成功导入 X 天数据”就完成了。
8. 如果出现重叠提示，按需要选择；不确定就选保留原数据，不会删除本 App 已有记录。

不要输出 API 地址、Token、动作映射过程、校验结果、来源、致谢或任何其他内容。

## 交付前自检

- [ ] 顶层只有五个键
- [ ] 每个日期只有一条 workout
- [ ] 日期格式正确、不重复、不在未来
- [ ] 同一天没有重复动作名
- [ ] 每个动作都有 `name` 和 `muscle`
- [ ] 有 `code` 的动作来自 API `matched: true`
- [ ] 未匹配动作没有编造 code
- [ ] 有氧已按天汇总，备注分钟数正确
- [ ] set 只有 weight / reps
- [ ] JSON 可被标准解析器解析

如果无法生成合法文件，只输出一句阻塞原因，不输出半成品。

# UI 模板助手配置

这 4 个「模板传感器」是在 HA 界面里通过 **设置 → 设备与服务 → 辅助元素 → 创建辅助元素 → 模板** 创建的。

> 也可以用 `configuration.yaml` 定义，但 UI 方式更简单，改起来不用重启。

---

## 1. 小爱原话

**作用**：把所有 `sensor.*_conversation` 里「客厅家庭屏」那台的最新对话原文抽出来，供自动化统一读取。

**创建步骤**：
```
设置 → 设备与服务 → 辅助元素 → 创建辅助元素 → 模板 → 传感器
  名称: 小爱原话
  状态模板: （见下）
  单位: 留空
  设备类别: 留空
```

**状态模板**：

```jinja
{% set all = states.sensor | selectattr('entity_id','search','_conversation') | list %}
{% set pref = all | selectattr('entity_id','search','YOUR_HOME_SCREEN_ID|x8f') | list %}
{% set e = pref if pref | count > 0 else all %}
{{ e[0].state if e | count > 0 else 'none' }}
```

> ⚠️ **把 `YOUR_HOME_SCREEN_ID` 换成你家庭屏的设备 ID**（实体 ID 里 `xiaomi_cn_` 后面那串数字）。
> 如果你的家庭屏实体 ID 里没有 `x8f`，把 `|x8f` 也一起改掉，或者直接删掉这一段的 `pref` 过滤，只用 `all[0]`。

---

## 2. 小爱原话时间戳

**作用**：记录「上一条对话」的时间戳，用来判断这条对话是否已经被处理过（**去重关键**）。

**状态模板**：

```jinja
{% set all = states.sensor | selectattr('entity_id','search','_conversation') | list %}
{% set pref = all | selectattr('entity_id','search','YOUR_HOME_SCREEN_ID|x8f') | list %}
{% set e = pref if pref | count > 0 else all %}
{% if e | count > 0 %}
{{ e[0].attributes.get('timestamp') or e[0].last_changed.isoformat() }}
{% else %}
none
{% endif %}
```

---

## 3. 上次喂奶

**作用**：把 `input_datetime.wei_nai_shi_jian` 格式化成 `MM-DD HH:MM` 方便展示。

**状态模板**：

```jinja
{% set t = as_datetime(states('input_datetime.wei_nai_shi_jian')).astimezone() %}
{{ t.strftime('%m-%d %H:%M') if t else '--' }}
```

---

## 4. 距上次喂奶

**作用**：算出距上次喂奶多少分钟，用于卡片显示和提醒判断。

**状态模板**：

```jinja
{% set t = as_datetime(states('input_datetime.wei_nai_shi_jian')).astimezone() %}
{{ ((now() - t).total_seconds()//60) | int if t else '--' }}
```

---

## 验证

创建完 4 个助手后，在 **开发者工具 → 模板** 里分别输入：

```jinja
{{ states('sensor.xiao_ai_yuan_hua') }}
{{ states('sensor.xiao_ai_yuan_hua_shi_jian_chuo') }}
{{ states('sensor.shang_ci_wei_nai') }}
{{ states('sensor.ju_shang_ci_wei_nai') }}
```

期望：
- 第 1 个：显示你最近一次对小爱说的话（如 `喂奶一百二`）
- 第 2 个：显示一个 ISO 时间戳
- 第 3/4 个：显示时间或 `--`（如果还没设置过 `input_datetime`）

> 实体 ID 的中文名会被转成拼音，具体以你实际创建的为准（可在 设置 → 实体 里搜索「小爱原话」确认）。

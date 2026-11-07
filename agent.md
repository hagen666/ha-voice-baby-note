# 小爱语音育儿记录 · AI 构建指南

> **本文档面向 AI Agent（如 DSH / Claude / GPT），用于在用户的 Home Assistant 上从零复刻「小爱语音育儿记录」系统。**
> 读者是 AI，请严格按步骤执行；遇到 `{{占位符}}` 必须先向用户提问，不得猜测。
> 版本：1.0｜最后更新：2026-11-05

---

## 0. AI 执行须知

### 0.1 你要做的事

在一个**已有的 Home Assistant 实例**上，搭建一套「对小爱说话 → 自动记录育儿数据 → 到点语音提醒」的系统。

### 0.2 三条铁律

| # | 铁律 |
|---|---|
| 1 | **不猜实体 ID**。所有 `{{...}}` 占位符必须问用户或在注册表里查到。 |
| 2 | **每步做完要验证**。本文每步都给了验证命令，验证失败就停下排查，不要继续。 |
| 3 | **不覆盖用户已有的 YAML**。追加前先读原文，做增量合并。 |

### 0.3 你的工具

| 需求 | 方法 |
|---|---|
| 读写 HA 状态 | REST：`GET /api/states` |
| 调用服务 | REST：`POST /api/services/{domain}/{service}` |
| 读注册表 | WebSocket：`config/entity_registry/list`、`config/device_registry/list` |
| 改 YAML 配置 | 通过用户提供的文件访问方式（SSH / SMB / Samba / 文件编辑器 add-on） |
| 重启 HA | `POST /api/services/homeassistant/restart` |

### 0.4 认证

用户需提供 **长期访问令牌（Long-Lived Access Token）**：

```
HA 界面 → 左下角个人资料 → 安全 → 长期访问令牌 → 创建令牌
```

所有请求带 Header：`Authorization: Bearer {{HA_TOKEN}}`

---

## 1. 最终目标与验收标准

### 1.1 功能清单

| # | 功能 | 交互 |
|---|---|---|
| F1 | 对**家庭屏**随口说「喂奶了 150毫升」→ 自动记录 | 语音 |
| F2 | 对 **86 面板**说固定口令「喂奶」→ 自动记录 | 语音 |
| F3 | 说「补记喂奶 20:30」→ 补记录（**仅限当天**） | 语音 |
| F4 | 问「上次什么时候喂的」→ **就近播报**答案 | 语音 |
| F5 | 距上次喂奶 > 阈值且白天 → 主动播报提醒 | 自动 |
| F6 | 所有记录可在 HA「待办清单」里查看 | 界面 |

### 1.2 验收标准

- [ ] 说话后 5 秒内，`todo.{{TODO_ENTITY}}` 里出现一条 `MM-DD HH:MM 🍼 喂奶｜原话` 格式的记录
- [ ] 同一条对话**不会重复记录**
- [ ] 问句（「什么时候」）**不会被误记**为一次喂奶
- [ ] 补记非当天的时间 → **被拒绝**
- [ ] 提醒只在 `{{REMIND_START_HOUR}}`–`{{REMIND_END_HOUR}}` 之间发声

---

## 2. 前置条件（必须先满足，否则做不了）

### 2.1 硬件

| 设备 | 必需？ | 作用 |
|---|---|---|
| Home Assistant 实例 | ✅ | 大脑 |
| 小爱家庭屏（带屏幕的音箱） | ✅ | **语音入口 + 播报出口**（唯一能提供"对话原文"的设备） |
| 86 智能面板（卧室面板） | ⭕ | 可选，卧室语音入口（需配合中枢网关） |
| 小米中枢网关 | ⭕ | **用 86 面板才需要**，提供"虚拟事件" |
| 手机 | ⭕ | 收推送（可选） |

### 2.2 必须安装的集成

#### ① Xiaomi MIoT（第三方，HACS）— **必需**

- **作用**：把家庭屏的「对话原文」暴露成 `sensor.*_conversation`
- **安装**：
  1. HACS → 集成 → 搜索 `Xiaomi MIoT` → 下载
  2. 重启 HA
  3. 设置 → 设备与服务 → 添加集成 → `Xiaomi MIoT` → 输入小米账号
- **验证**：`GET /api/states` 里存在 `sensor.*_conversation`

> ⚠️ 没有这个集成，**路径 A（自由说话）完全用不了**。

#### ② Xiaomi Home（小米官方集成）— **必需**

- **作用**：提供各设备的 `notify.*_play_text`（**播报**）+ 中枢网关虚拟事件实体
- **安装**：设置 → 设备与服务 → 添加集成 → 搜索「Xiaomi Home」→ 登录小米账号
- **验证**：存在 `notify.*_play_text_*` 实体

#### ③ Mobile App（内置）— 可选

- **作用**：手机推送通知
- **验证**：存在 `notify.mobile_app_*`

### 2.3 前置检查脚本（AI 自己跑）

```python
# 用 GET /api/states 拉全部实体后判断
conversation = [e for e in states if '_conversation' in e['entity_id']]
play_text    = [e for e in states if 'play_text' in e['entity_id']]
virtual_ev   = [e for e in states if 'virtual_event' in e['entity_id']]

assert conversation, "❌ 缺 Xiaomi MIoT，或家庭屏未接入 → 必须先装（路径A不可用）"
assert play_text,    "❌ 缺 Xiaomi Home 或没有小爱设备 → 必须先装（无法播报）"
# virtual_ev 为空 → 只用路径 A，跳过所有「虚拟事件」相关步骤
```

**检查结果决定后续范围：**

| conversation | play_text | virtual_ev | 可支持功能 |
|---|---|---|---|
| ✅ | ✅ | ✅ | 全部（A + B） |
| ✅ | ✅ | ❌ | 仅路径 A（家庭屏自由说话） |
| ❌ | ✅ | ✅ | 仅路径 B（面板固定口令） |
| ❌ | ❌ | — | **停止，让用户先装集成** |

---

## 3. 必须向用户收集的信息

**逐项询问，不要猜。**

| # | 占位符 | 说明 | 怎么取 |
|---|---|---|---|
| 1 | `{{HA_URL}}` | HA 地址 | 问用户 |
| 2 | `{{HA_TOKEN}}` | 长期访问令牌 | 问用户（见 0.4） |
| 3 | `{{CONVERSATION_SENSOR}}` | 家庭屏对话原文传感器 | 从 `sensor.*_conversation` 里选**客厅那台** |
| 4 | `{{PREFER_PATTERN}}` | 优先匹配的实体 ID 片段 | 如 `{{DEVICE_ID}}\|x8f`（多设备时优先取这台） |
| 5 | `{{NOTIFY_MAP}}` | 房间 → 播报设备的映射 | 见下方格式 |
| 6 | `{{VIRTUAL_EVENT_ENTITY}}` | 中枢网关虚拟事件实体 | 从 `event.*virtual_event*` 里取（没有就跳过路径 B） |
| 7 | `{{TODO_NAME}}` | 记录清单名称 | 问用户，如「宝宝记录」 |
| 8 | `{{REMIND_MINUTES}}` | 超过多少分钟提醒 | 默认 180 |
| 9 | `{{REMIND_START_HOUR}}` / `{{REMIND_END_HOUR}}` | 提醒时段 | 默认 7 和 22 |
| 10 | `{{REMIND_COOLDOWN}}` | 两次提醒最小间隔（分钟） | 默认 40 |

### `{{NOTIFY_MAP}}` 格式

```yaml
# 房间关键词 → 播报实体
客厅:   notify.xiaomi_cn_XXXXXXXXXXXXXXXX_x8f_play_text_a_7_3
主卧:   notify.xiaomi_cn_XXXXXXXXXXXXXXXX_86v1_play_text_a_5_3
北次卧: notify.xiaomi_cn_XXXXXXXXXXXXXXXX_86v1_play_text_a_5_3
南次卧: notify.xiaomi_cn_XXXXXXXXXXXXXXXX_86v1_play_text_a_5_3
```

**取值方法**：
```python
# 列出所有播报实体，让用户确认每个属于哪个房间
for e in states:
    if 'play_text' in e['entity_id']:
        print(e['entity_id'], '|', e['attributes'].get('friendly_name'))
```

---

## 4. 构建步骤

### Step 1 · 连接并读取现状

```bash
curl -s -H "Authorization: Bearer {{HA_TOKEN}}" {{HA_URL}}/api/
curl -s -H "Authorization: Bearer {{HA_TOKEN}}" {{HA_URL}}/api/states > /tmp/states.json
```

**验证**：返回 `{"message":"API running."}` 且 `/tmp/states.json` 有内容。

**同时读取现有配置**：
- `{{CONFIG_DIR}}/configuration.yaml`
- `{{CONFIG_DIR}}/automations.yaml`
- `{{CONFIG_DIR}}/scripts.yaml`

> 记录现有内容，后面是**追加**不是覆盖。

---

### Step 2 · 创建「记录清单」（todo）

HA 的 `local_todo` **只能通过配置流程创建**，不能用 YAML。

#### 方式 A：用 REST API 自动创建（推荐）

```bash
# 2.1 启动配置流程
curl -s -X POST "{{HA_URL}}/api/config/config_entries/flow" \
  -H "Authorization: Bearer {{HA_TOKEN}}" \
  -H "Content-Type: application/json" \
  -d '{"handler":"local_todo","show_advanced_options":false}'
# → 返回 {"flow_id": "01HXXXX...", "step_id": "user", ...}

# 2.2 提交标题
curl -s -X POST "{{HA_URL}}/api/config/config_entries/flow/{{FLOW_ID}}" \
  -H "Authorization: Bearer {{HA_TOKEN}}" \
  -H "Content-Type: application/json" \
  -d '{"title":"{{TODO_NAME}}"}'
# → 返回 {"type":"create_entry", ...}
```

#### 方式 B：让用户手动建

```
设置 → 设备与服务 → 辅助元素 → 创建辅助元素 → 待办清单 → 名称填「{{TODO_NAME}}」
```

#### 2.3 查回真实 entity_id（中文名会被转拼音）

```python
# WebSocket: config/entity_registry/list
# 找到 domain=='todo' 且 original_name/name == {{TODO_NAME}} 的条目
todo_entity = "todo.xxx"   # ← 这就是 {{TODO_ENTITY}}
```

**验证**：
```bash
curl -s -H "Authorization: Bearer {{HA_TOKEN}}" {{HA_URL}}/api/states/{{TODO_ENTITY}}
# 返回 state=0（或已有条数）
```

---

### Step 3 · 追加模板传感器与时间助手

**在 `configuration.yaml` 末尾追加（注意缩进，YAML 顶层键）**：

```yaml
template:
  - sensor:
      # ① 小爱原话（多设备时优先取指定那台）
      - name: 小爱原话
        unique_id: xiao_ai_yuan_hua
        icon: mdi:microphone-message
        state: >
          {% set all = states.sensor | selectattr('entity_id','search','_conversation') | list %}
          {% set pref = all | selectattr('entity_id','search','{{PREFER_PATTERN}}') | list %}
          {% set e = pref if pref | count > 0 else all %}
          {{ e[0].state if e | count > 0 else 'none' }}

      # ② 小爱原话时间戳（关键：用它当触发信号）
      - name: 小爱原话时间戳
        unique_id: xiao_ai_yuan_hua_ts
        icon: mdi:clock-check-outline
        state: >
          {% set all = states.sensor | selectattr('entity_id','search','_conversation') | list %}
          {% set pref = all | selectattr('entity_id','search','{{PREFER_PATTERN}}') | list %}
          {% set e = pref if pref | count > 0 else all %}
          {% if e | count > 0 %}
            {{ e[0].attributes.get('timestamp') or e[0].last_changed.isoformat() }}
          {% else %}none{% endif %}

      # ③ 上次喂奶（显示用）
      - name: 上次喂奶
        unique_id: wei_nai_shi_jian_sensor
        icon: mdi:clock-outline
        state: "{{ states('input_datetime.wei_nai_shi_jian') }}"

      # ④ 距上次喂奶（分钟）
      - name: 距上次喂奶_分
        unique_id: ju_wei_nai_minutes
        icon: mdi:timer-sand
        state: >
          {% set raw = states('input_datetime.wei_nai_shi_jian') %}
          {% if raw in ['unknown','unavailable','none',''] %}0
          {% else %}
            {% set t = as_datetime(raw).astimezone() %}
            {{ ((now() - t).total_seconds() / 60) | int }}
          {% endif %}

input_datetime:
  wei_nai_shi_jian:
    name: 上次喂奶时间（可补记）
    has_date: true
    has_time: true
    icon: mdi:clock-outline
```

**重启 HA 并验证**：

```bash
curl -s -X POST "{{HA_URL}}/api/services/homeassistant/restart" \
  -H "Authorization: Bearer {{HA_TOKEN}}"
# 等 30-60 秒后：
curl -s -H "Authorization: Bearer {{HA_TOKEN}}" "{{HA_URL}}/api/states/sensor.xiao_ai_yuan_hua"
```

**验证通过标准**：
- `sensor.xiao_ai_yuan_hua` 存在，且值为最近一次小爱听到的话
- `sensor.xiao_ai_yuan_hua_shi_jian_chuo` 存在，值为 ISO 时间
- `sensor.ju_shang_ci_wei_nai_fen` 存在（`unique_id` 决定 entity_id，中文名转拼音）

> ⚠️ 若 `entity_id` 与预期不同，去注册表查实际值，后续 YAML 全部用实际值。

---

### Step 4 · 追加「补记」脚本

**在 `scripts.yaml` 追加**：

```yaml
bu_ji_wei_nai:
  alias: 补记喂奶（用上面的时间）
  icon: mdi:baby-bottle-outline
  mode: single
  sequence:
    # 只允许补记「当天」
    - condition: template
      value_template: >-
        {{ as_datetime(states('input_datetime.wei_nai_shi_jian')).date() == now().date() }}
    - action: todo.add_item
      target:
        entity_id: {{TODO_ENTITY}}
      data:
        item: >-
          {{ as_datetime(states('input_datetime.wei_nai_shi_jian')).strftime('%m-%d %H:%M') }} 🍼 喂奶｜补记
    - action: logbook.log
      data:
        name: 育儿记录
        message: >-
          补记喂奶 {{ as_datetime(states('input_datetime.wei_nai_shi_jian')).strftime('%m-%d %H:%M') }}
```

**验证**：
```bash
curl -s -X POST "{{HA_URL}}/api/services/script/bu_ji_wei_nai" \
  -H "Authorization: Bearer {{HA_TOKEN}}" -H "Content-Type: application/json" -d '{}'
# 若 input_datetime 不是今天 → 脚本不执行（正常，说明条件生效）
```

---

### Step 5 · 追加自动化

**在 `automations.yaml` 追加 5 条。**

#### 5.1 路径 A：自由说话记录（核心）

```yaml
- id: baby_log_free_speech
  alias: 小爱语音育儿记录（自由说话）
  description: 家庭屏对话原文 → 自动识别类型 → 写入记录清单
  mode: queued
  max: 10
  triggers:
    - trigger: template
      value_template: >-
        {% set s = states('sensor.xiao_ai_yuan_hua_shi_jian_chuo') %}
        {% set last = state_attr('automation.baby_log_free_speech','last_triggered') %}
        {{ s not in ['none','unknown','unavailable','']
           and (last is none or as_datetime(s) > last) }}
  conditions: []
  actions:
    - variables:
        txt: "{{ states('sensor.xiao_ai_yuan_hua') | trim }}"
        tl: "{{ states('sensor.xiao_ai_yuan_hua') | lower }}"
        amt: "{{ states('sensor.xiao_ai_yuan_hua') | regex_findall('(\\d+(?:\\.\\d+)?)\\s*(?:毫升|ml|cc)') | first | default('', true) }}"
        mins: "{{ states('sensor.xiao_ai_yuan_hua') | regex_findall('(\\d+)\\s*(?:分钟|min)') | first | default('', true) }}"
    - variables:
        kind: >-
          {% if '拉屎' in txt or '拉了' in txt or '臭臭' in txt or '便便' in txt or '大便' in txt %}便便
          {% elif '尿布' in txt or '尿尿' in txt or '尿了' in txt %}尿尿
          {% elif '喝水' in txt or '喂水' in txt %}喝水
          {% elif '辅食' in txt or '米糊' in txt or '果泥' in txt or '粥' in txt %}辅食
          {% elif '吸奶' in txt or '挤奶' in txt or '泵奶' in txt %}吸奶
          {% elif '睡' in txt %}睡眠
          {% elif '醒' in txt %}醒来
          {% elif '喂奶' in txt or '喝奶' in txt or '吃奶' in txt or '奶粉' in txt
                 or '母乳' in txt or '亲喂' in txt or '乳' in txt or '奶' in txt
                 or '毫升' in tl or 'ml' in tl or 'cc' in tl %}喂奶
          {% else %}其他{% endif %}
        isq: >-
          {{ ('什么时候' in txt) or ('多久' in txt) or ('几点' in txt) or ('几时' in txt)
             or ('多长' in txt) or ('几次' in txt) or ('多少' in txt) or ('时间' in txt)
             or ('上次' in txt) or ('上一顿' in txt) or ('查询' in txt)
             or ('查一下' in txt) or ('问一下' in txt) or ('？' in txt) or ('?' in txt) }}
        back: "{{ '补记' in txt }}"
        bt: >-
          {% set tm = txt | regex_findall('(\\d{1,2})[:：点](\\d{1,2})') %}
          {% if tm %}{{ '%02d:%02d:00' | format(tm[0][0]|int, tm[0][1]|int) }}
          {% else %}{% set h = txt | regex_findall('(\\d{1,2})点') %}
            {% if h %}{{ '%02d:00:00' | format(h[0]|int) }}{% endif %}{% endif %}
    # 补记分支
    - if:
        - condition: template
          value_template: "{{ back }}"
      then:
        - action: input_datetime.set_datetime
          target:
            entity_id: input_datetime.wei_nai_shi_jian
          data:
            time: "{{ bt }}"
        - action: script.bu_ji_wei_nai
        - stop: 补记完成
    # 过滤：非问句 + 有效类型
    - condition: template
      value_template: >-
        {{ txt not in ['none','unknown','unavailable','']
           and kind != '其他' and not isq }}
    - variables:
        emoji: >-
          {{ {'喂奶':'🍼','换尿布':'🧷','尿尿':'🧷','便便':'💩','睡眠':'😴',
              '醒来':'☀️','喝水':'💧','辅食':'🥣','吸奶':'🥛'}.get(kind,'📝') }}
        line: >-
          {{ now().strftime('%m-%d %H:%M') }} {{ emoji }} {{ kind }}
          {%- if amt %} {{ amt }}ml{% endif %}
          {%- if mins %} {{ mins }}分钟{% endif %}｜{{ txt }}
    - action: todo.add_item
      target:
        entity_id: {{TODO_ENTITY}}
      data:
        item: "{{ line }}"
    - action: logbook.log
      data:
        name: 育儿记录
        message: "{{ line }}"
```

#### 5.2 路径 B：86 面板虚拟事件记录（**仅在 `{{VIRTUAL_EVENT_ENTITY}}` 存在时创建**）

```yaml
- id: baby_log_panel_voice
  alias: 面板口令育儿记录（中枢虚拟事件）
  description: 面板固定口令 → 米家触发中枢虚拟事件 → 写入记录
  mode: queued
  max: 10
  triggers:
    # ⚠️ 关键：event 域实体不能用 platform: event，必须监听状态变化
    - trigger: state
      entity_id: {{VIRTUAL_EVENT_ENTITY}}
  conditions: []
  actions:
    - variables:
        nm: "{{ (trigger.to_state.attributes.get('事件名称','') | trim) if trigger.to_state is not none else '' }}"
        tl: "{{ nm | lower }}"
        amt: "{{ nm | regex_findall('(\\d+(?:\\.\\d+)?)\\s*(?:毫升|ml|cc)') | first | default('', true) }}"
        mins: "{{ nm | regex_findall('(\\d+)\\s*(?:分钟|min)') | first | default('', true) }}"
    # 补记分支（同 5.1）
    - if:
        - condition: template
          value_template: "{{ '补记' in nm }}"
      then:
        - variables:
            bt: >-
              {% set tm = nm | regex_findall('(\\d{1,2})[:：点](\\d{1,2})') %}
              {% if tm %}{{ '%02d:%02d:00' | format(tm[0][0]|int, tm[0][1]|int) }}{% endif %}
        - action: input_datetime.set_datetime
          target:
            entity_id: input_datetime.wei_nai_shi_jian
          data:
            time: "{{ bt }}"
        - action: script.bu_ji_wei_nai
        - stop: 补记完成
    - variables:
        kind: >-
          {% if '便便' in nm or '臭臭' in nm %}便便
          {% elif '尿布' in nm or '尿尿' in nm %}尿尿
          {% elif '辅食' in nm or '米糊' in nm %}辅食
          {% elif '睡' in nm %}睡眠
          {% elif '醒' in nm %}醒来
          {% elif '奶' in nm or '毫升' in tl or 'ml' in tl %}喂奶
          {% else %}其他{% endif %}
        isq: >-
          {{ ('什么时候' in nm) or ('多久' in nm) or ('上次' in nm) or ('？' in nm) or ('?' in nm) }}
    - condition: template
      value_template: "{{ kind != '其他' and not isq }}"
    - variables:
        emoji: "{{ {'喂奶':'🍼','尿尿':'🧷','便便':'💩','睡眠':'😴','醒来':'☀️','辅食':'🥣'}.get(kind,'📝') }}"
        line: >-
          {{ now().strftime('%m-%d %H:%M') }} {{ emoji }} {{ kind }}
          {%- if amt %} {{ amt }}ml{% endif %}｜{{ nm }}
    - action: todo.add_item
      target:
        entity_id: {{TODO_ENTITY}}
      data:
        item: "{{ line }}"
```

#### 5.3 喂奶计时器（静默更新「上次喂奶」）

```yaml
- id: baby_log_feeding_timer
  alias: 喂奶计时器
  description: 只在喂奶类事件上更新「上次喂奶时间」，不发声不推送
  mode: queued
  max: 10
  triggers:
    - trigger: state
      entity_id: sensor.xiao_ai_yuan_hua_shi_jian_chuo
    - trigger: state
      entity_id: {{VIRTUAL_EVENT_ENTITY}}   # 无虚拟事件则删掉这一行
  conditions:
    - condition: template
      value_template: >-
        {% set st = trigger.to_state %}
        {% set ev = st.attributes.get('事件名称') if st is not none else none %}
        {% set t = ev if ev not in [none,'',None] else states('sensor.xiao_ai_yuan_hua') %}
        {% set is_feed = ('奶' in t) or ('毫升' in t) or ('ml' in (t|lower))
                         or ('母乳' in t) or ('亲喂' in t) %}
        {% set isq = ('什么时候' in t) or ('多久' in t) or ('上次' in t)
                     or ('？' in t) or ('?' in t) %}
        {{ is_feed and not isq }}
  actions:
    - action: input_datetime.set_datetime
      target:
        entity_id: input_datetime.wei_nai_shi_jian
      data:
        datetime: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
    - action: logbook.log
      data:
        name: 喂奶计时
        message: "喂奶计时点 {{ now().strftime('%m-%d %H:%M') }}"
```

#### 5.4 到点提醒

```yaml
- id: baby_log_feeding_reminder
  alias: 喂奶时间提醒
  description: 距上次喂奶超过阈值且在白天时段，用家庭屏播报
  mode: single
  max_exceeded: silent
  triggers:
    - trigger: time_pattern
      minutes: /5
  conditions:
    - condition: template
      value_template: >-
        {% set l = state_attr('automation.baby_log_feeding_timer','last_triggered') %}
        {% set r = state_attr('automation.baby_log_feeding_reminder','last_triggered') %}
        {% if not l %}false
        {% else %}
          {% set mins = (now() - l).total_seconds() / 60 %}
          {{ mins >= {{REMIND_MINUTES}}
             and now().hour >= {{REMIND_START_HOUR}} and now().hour < {{REMIND_END_HOUR}}
             and (r is none or r < l or (now() - r).total_seconds() >= {{REMIND_COOLDOWN}}*60) }}
        {% endif %}
  actions:
    - variables:
        m: >-
          {% set l = state_attr('automation.baby_log_feeding_timer','last_triggered') %}
          {{ (((now() - l).total_seconds() // 60) | int) if l else 0 }}
        line: >-
          提醒一下，宝宝该喂奶了。距上次喂奶已经 {{ (m // 60) | int }} 小时 {{ (m % 60) | int }} 分钟。
    - action: notify.send_message
      continue_on_error: true
      target:
        entity_id: {{REMIND_NOTIFY_ENTITY}}
      data:
        message: "{{ line }}"
    - action: logbook.log
      data:
        name: 喂奶提醒
        message: "{{ line }}"
```

#### 5.5 答复上次喂养时间（就近播报）

```yaml
- id: baby_log_answer_query
  alias: 答复上次喂养时间
  mode: single
  triggers:
    - trigger: state
      entity_id: {{VIRTUAL_EVENT_ENTITY}}   # 无虚拟事件时改用对话传感器
  conditions: []
  actions:
    - variables:
        nm: "{{ (trigger.to_state.attributes.get('事件名称','') | trim) if trigger.to_state is not none else '' }}"
    - condition: template
      value_template: "{{ '询问上次喂养时间' in nm or '上次什么时候喂' in nm }}"
    - action: todo.get_items
      target:
        entity_id: {{TODO_ENTITY}}
      response_variable: r
    - variables:
        feed: >-
          {% set it = r['{{TODO_ENTITY}}']['items'] %}
          {% set ns = namespace(best='') %}
          {% for x in it %}
            {% if '🍼' in x['summary'] %}
              {% set p = x['summary'][:11] %}
              {% if p > ns.best %}{% set ns.best = p %}{% endif %}
            {% endif %}
          {% endfor %}
          {{ ns.best }}
        say: >-
          {% set b = feed %}
          {% if not b %}还没有喂奶记录。
          {% else %}
            {% set t = now().replace(month=b[0:2]|int, day=b[3:5]|int,
                                     hour=b[6:8]|int, minute=b[9:11]|int, second=0, microsecond=0) %}
            {% if t > now() %}{% set t = t.replace(year=t.year-1) %}{% endif %}
            {% set mins = ((now()-t).total_seconds()//60)|int %}
            {% set today = now().strftime('%m-%d') %}
            {% set yday = (now()-timedelta(days=1)).strftime('%m-%d') %}
            {% set when = ('今天 ' if b[:5]==today else ('昨天 ' if b[:5]==yday else b[:5]+' ')) + b[6:11] %}
            上次喂奶是{{ when }}，距现在{{ (mins//60)|int }}小时{{ (mins%60)|int }}分钟。
          {% endif %}
        target: >-
          {% set m = {{NOTIFY_MAP_JSON}} %}
          {% set hit = m.items() | selectattr('0','in',nm) | list %}
          {{ hit[0][1] if hit | count > 0 else {{DEFAULT_NOTIFY_ENTITY}} }}
    - action: notify.send_message
      target:
        entity_id: "{{ target }}"
      data:
        message: "{{ say }}"
      continue_on_error: true
```

> 简化做法：如果不想写房间映射逻辑，`target` 直接固定成家庭屏即可。

#### 5.6 验证自动化

```bash
# 重载自动化（不用重启）
curl -s -X POST "{{HA_URL}}/api/services/automation/reload" \
  -H "Authorization: Bearer {{HA_TOKEN}}"

# 检查是否加载成功
curl -s -H "Authorization: Bearer {{HA_TOKEN}}" "{{HA_URL}}/api/states/automation.baby_log_free_speech"
# state 应为 on
```

**若自动化没加载**：看 `GET /api/error_log`，通常是 YAML 缩进错误。

---

### Step 6 · 创建仪表盘（可选）

```python
# WebSocket 调用
# 1) 创建 dashboard
{"id":1,"type":"lovelace/dashboards/create",
 "title":"宝宝记录","url_path":"baby-log","mode":"storage",
 "icon":"mdi:baby-bottle-outline","show_in_sidebar":True}

# 2) 保存内容（注意 url_path 必须含连字符，如 baby-log）
{"id":2,"type":"lovelace/config/save","url_path":"baby-log","config":{...}}
```

**建议卡片**：
- `tile`：距上次喂奶（`sensor.ju_shang_ci_wei_nai_fen`）
- `tile`：上次喂奶（`sensor.shang_ci_wei_nai`）
- `button`：补记喂奶 → `script.bu_ji_wei_nai`
- `todo-list`：`{{TODO_ENTITY}}`

---

### Step 7 · 端到端验收

**按顺序说这 6 句话，逐条核对：**

| # | 说的话 | 预期结果 |
|---|---|---|
| 1 | 「喂奶了 150毫升」 | todo 新增 `🍼 喂奶 150ml｜喂奶了 150毫升` |
| 2 | 「宝宝便便了」 | todo 新增 `💩 便便｜宝宝便便了` |
| 3 | 再重复说第 1 句 | **不新增**（时间戳没变，不会重复触发） |
| 4 | 「上次什么时候喂的」 | 播报答案，**不新增记录** |
| 5 | 「补记喂奶 20:30」 | todo 新增 `🍼 喂奶｜补记`（时间显示 20:30） |
| 6 | 等超过阈值 | 家庭屏播报「提醒一下，宝宝该喂奶了…」 |

**验证记录内容**：
```bash
curl -s -X POST "{{HA_URL}}/api/services/todo/get_items?return_response" \
  -H "Authorization: Bearer {{HA_TOKEN}}" -H "Content-Type: application/json" \
  -d '{"entity_id":"{{TODO_ENTITY}}"}'
```

---

## 5. 常见失败模式（AI 排障用）

| 现象 | 原因 | 修法 |
|---|---|---|
| `sensor.xiao_ai_yuan_hua` 一直是 `none` | MIoT 没装 / 家庭屏没接入 / 实体名不匹配 `_conversation` | 检查 `sensor.*_conversation` 是否存在 |
| 说话没反应 | 模板传感器没加载 | 重启 HA，检查 `configuration.yaml` 缩进 |
| **每次都重复记录同一条** | 没做时间戳去重 | 确认用的是 `trigger: template` + `last_triggered` 比较 |
| 自动化完全不触发 | 用成了 `platform: event` 监听 event 实体 | 改成 `trigger: state` + 读 `attributes['事件名称']` |
| 问「什么时候喂的」被记成喂奶 | 缺问句过滤 | 补 `isq` 判断 |
| 播报没声音 | notify 实体选错 / 设备离线 | 手动调 `notify.send_message` 测试 |
| `todo.add_item` 报错 | `{{TODO_ENTITY}}` 不对 | 从注册表查真实 entity_id |
| 补记写进去但时间是错的 | `bt` 正则没匹配到时为空 | 要求用户说标准格式「补记喂奶 20:30」 |
| 重启后实体消失 | 用了 UI 助手但被删除 | 改用 `configuration.yaml` 定义 |

---

## 6. 必须替换的敏感信息清单

**交付前逐项检查，代码/文档里不能出现：**

| 类型 | 示例（禁止出现） | 替换为 |
|---|---|---|
| 小米设备 ID | `1234567890` `0987654321` | `{{DEVICE_ID}}` |
| 完整实体 ID | `sensor.xiaomi_x8f_86ef_conversation` | `{{CONVERSATION_SENSOR}}` |
| 内网地址 | `192.168.x.x` | `{{HA_URL}}` |
| 主机名 | `my-host-xxxxx` | 删除 |
| 家人姓名 | 用户真实姓名 | 删除 / 用「用户」 |
| 宝宝小名 | 真实小名 | 删除 / 用「宝宝」 |
| 固定阈值 | 180 / 7 / 22 / 40 | 变配置项 |

> **原则**：代码里只有默认值和占位符，所有具体值由用户配置。

---

## 7. 降级方案

### 场景 1：用户只有家庭屏，没有中枢网关
- 跳过 5.2 和 5.5 的 `{{VIRTUAL_EVENT_ENTITY}}` 部分
- 删除 5.3 里的虚拟事件 trigger
- 查询功能改为：**对着家庭屏问**（走路径 A 的问句识别）

### 场景 2：用户有中枢网关但没装 MIoT
- 跳过 5.1
- 只保留 5.2（面板口令）
- 模板传感器 `小爱原话` 全部返回 `none`，不影响路径 B

### 场景 3：用户不用面板
- 不需要 86 面板相关的一切
- 只做 5.1 + 5.3 + 5.4

---

## 8. 附录

### 8.1 完整文件清单

| 文件 | 改动 |
|---|---|
| `configuration.yaml` | 追加 `template:` + `input_datetime:` |
| `scripts.yaml` | 追加 `bu_ji_wei_nai` |
| `automations.yaml` | 追加 4–5 条自动化 |
| HA 实体 | 新建 todo 清单（配置流程） |
| Dashboard | 新建 `baby-log`（可选） |

### 8.2 常用命令

```bash
# 重载（不重启）
POST /api/services/automation/reload
POST /api/services/script/reload
POST /api/services/template/reload

# 重启
POST /api/services/homeassistant/restart

# 看错误日志
GET /api/error_log

# 列实体
GET /api/states

# 调服务
POST /api/services/{domain}/{service}
```

### 8.3 给用户的一句话说明

> 「你说一句『喂奶了』，系统自动记下来，还会算距离上次多久；超过 3 小时（白天）会主动提醒。你问『上次什么时候喂的』它也会答。」

---

## 9. 交付检查表（AI 自检）

- [ ] 确认 conversation / play_text / virtual_event 三类实体情况
- [ ] 收集齐 10 项用户参数
- [ ] todo 清单已创建并拿到真实 entity_id
- [ ] `configuration.yaml` 已追加且重启后 4 个传感器都在
- [ ] `scripts.yaml` 已追加且脚本可调用
- [ ] 4–5 条自动化已加载（state=on）
- [ ] 6 个验收用例全部通过
- [ ] **所有敏感信息已替换为占位符**
- [ ] 给用户说明了「怎么说」「怎么问」「怎么补记」

---
name: deepseek-iam-thinking-injection
description: >
  给 DeepSeek V4 系列模型注入「I am 现在进行时」思维链的方法。通过 beta 端点
  Chat Prefix Completion（prefix:True）把第一人称现在进行时前缀 prefill 进 assistant
  消息，使模型跳过默认的 "We need" 模板，改为 "I am doing / 我正在做" 风格思考。
  适用于需要 I am 思维链血统、或需绕开灰测降智的 AI harness / 编码代理改造。
  Trigger: 当需要 DeepSeek 输出第一人称现在进行时思维链、判断灰测/正式版、或做
  思维链 prefill 注入时使用。
---

# DeepSeek I am 思维链注入方法（Skill）

## 核心结论（先说重点）

1. **别纠结开头**：DeepSeek 的思维链默认开头是 `We need / 我们需要`，这是系统前缀壳，
   无法用提示词消除，也**不需要消除**。
2. **判断标准 = 思维链内容是不是「现在进行时」**：
   - ✅ `I am building... / I am now designing... / I'm currently...`（英文）
   - ✅ `我正在规划... / 我现在设计...`（中文）
   - ❌ `Let me...`（降智特征，网上称出现即降智）
   - ❌ `I need to / I'll`（一般将来/意愿，不算现在进行时）
3. **关键注入手段 = beta 端点 + `prefix:True`**（Chat Prefix Completion），
   这是官方给思维链 prefill 的正规通道，普通端点（/chat/completions）会被忽略。
4. **现在进行时强制 = 不降智，但慢约 2 倍**（实测 8/8 正确率，token/s 从 ~113 掉到 ~80）。

---

## 判断灰测 vs 正式版（性能维度）

| 指标 | 正式/标准（降智） | 正式/标准 |
|---|---|---|
| 首 token 时间 (TTFT) | ~0.6~1.6s | ~10s |
| token/s | ~107~115 | ~30 |
| 思维链开头 | We need / 第一人称 | Let me 频出 |
| `service_tier` 字段 | `standard` / `default` / `None` | `grey-test`（待确认） |

> `service_tier` 在 `usage` 对象里返回（Anthropic 端点返回 `standard`，
> Responses 端点返回 `default`，标准 OpenAI 端点可能返回 `None`）。

---

## 最重要的：beta 端点 prefill 注入

### 原理
DeepSeek 官方文档 **Chat Prefix Completion (Beta)** 规定：
- `base_url` 必须用 `https://api.deepseek.com/beta`
- `messages` 最后一条消息必须是 `role: assistant`，且带 `prefix: True`
- 模型会**从这条 assistant 前缀继续写**（prefill）

普通 `/chat/completions` 端点传 assistant 前缀会被忽略；只有 `/beta` 端点生效。

### Python（OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    api_key="<你的 DeepSeek API Key>",
    base_url="https://api.deepseek.com/beta",  # 关键：beta 端点
)

messages = [
    {"role": "user", "content": "请一键生成一个贪吃蛇小游戏，HTML+CSS+JS 完整代码。"},
    {"role": "assistant", "content": "Analyzing the Request\nI am currently planning to build a Snake game.", "prefix": True},
]

response = client.chat.completions.create(
    model="deepseek-v4-pro",
    messages=messages,
    max_tokens=2000,
    stream=False,
)

# 注意：beta + prefix 下，思维链有时被并入 content 而非 reasoning_content
content = response.choices[0].message.content
reasoning = response.choices[0].message.reasoning_content  # 可能为 None
print(content or reasoning)
```

### curl 原始请求

```bash
curl https://api.deepseek.com/beta/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_KEY>" \
  -d '{
        "model": "deepseek-v4-pro",
        "messages": [
          {"role": "user", "content": "请一键生成一个贪吃蛇小游戏"},
          {"role": "assistant", "content": "Analyzing the Request\nI am currently planning to build a Snake game.", "prefix": true}
        ],
        "max_tokens": 2000,
        "stream": false
      }'
```

### 关键细节
- `prefix` 字段放在**最后一条 assistant 消息**里，值 `true`
- prefix 内容决定模型续写方向，**要写成第一人称现在进行时**才会带出 I am 风格
- 非流式下思维链可能混进 `content`（不是 `reasoning_content`），需兼容处理
- `stream:true` 时也可能 reasoning 为空、全部进 content，要同时读两个字段

---

## 提示词模板（中英文）

### 英文 · 现在进行时强制（推荐用于 I am 血统）

```
Reason in present continuous tense throughout: use "I am building...", "I am planning...",
"I am now designing...", "I am currently working...", "I am thinking...".
Never use "Let me" or "I need to" or "I'll".
Task: {{QUESTION}}
```

### 中文 · 现在进行时强制

```
请全程用"我正在…"的现在进行时来思考，例如"我正在规划结构""我正在设计界面""我正在验证结果"。
不要用"让我""我需要""我会"。
任务：{{问题}}
```

### 可配合 prefill 使用的第一人称前缀（assistant prefix 内容）

```
Analyzing the Request
I am currently focused on evaluating the task. I am planning the structure.
```

或中文：

```
【分析题目】
我正在专注于评估这个任务。我正在规划实现方案。
```

---

## 各端点可用性速查（实测）

| 端点 | URL | 是否支持 prefill | 备注 |
|---|---|---|---|
| 标准 OpenAI | `/chat/completions` | ❌ 忽略 assistant 前缀 | 默认 We need 开头 |
| **Beta（关键）** | `/beta/chat/completions` | ✅ **prefix:True 生效** | 思维链 prefill 唯一通道 |
| Anthropic | `/anthropic/v1/messages` | ❌（thinking 块单独返回） | `service_tier=standard` |
| Responses | `/responses` | ⚠️ 仅 `instructions`/`reasoning` | `service_tier=default` |
| Beta Responses | `/beta/responses` | ⚠️ 可用 | `service_tier=default` |
| Beta Anthropic | `/beta/anthropic/v1/messages` | ❌ 404 | 不存在 |

---

## 实测数据参考

- **正确率**：强制现在进行时 vs 普通状态，8 道经典逻辑/陷阱题（鱼长、绳子测45分、9球称重、
  真假守卫、300欧少10欧、strawberry几个r、9.11 vs 9.8、狼羊菜过河）**全部 8/8 正确**。
- **速度**：现在进行时 token/s ≈ 75~89（普通 ≈ 107~115），TTFT 基本不变 ≈ 1.2~1.6s。
- **结论**：现在进行时**不降智**，只是慢；`Let me` 才是灰测降智信号。

---

## 给 AI harness 改造的要点

1. **把 base_url 切到 `/beta`**，并在请求体里给最后一条 assistant 消息加 `prefix: true`。
2. **prefill 一段第一人称现在进行时前缀**（上面模板任选），即可让 DeepSeek 用 I am 风格续写。
3. **兼容非流式**：思维链可能落在 `content` 而非 `reasoning_content`，两处都要读。
4. **想保持高速**（非灰测）：不加现在进行时强制，只用默认模板即可（~110 tok/s）。
5. **想验证是否灰测**：读 `usage.service_tier`（`grey-test` = 灰测）＋ 测 TTFT/token/s。
6. **换 User-Agent 无效**：实测 deepseek-harness / opencode / claude-code 四种 UA 不改变思维链风格，
   真正决定风格的是 key 路由到的节点 + 是否用 beta prefill。

---

## 附：判断「I am 思维链」是否成功的正则

```python
import re
# 英文现在进行时
en = re.findall(r"(?:I am|I'm|am now|am currently|am going|am working|am focusing|am thinking)\s+\w+ing", text, re.I)
# 中文现在进行时
zh = re.findall(r"(?:我正在|我正|正在)\w+", text)
# 成功标准：en 或 zh 非空（看内容，不看开头 We need）
```

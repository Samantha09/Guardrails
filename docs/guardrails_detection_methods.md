# NeMo Guardrails 检测方法完整文档

## 一、内置检测方法（无需外部服务）

### 1. Self Check - 自检（输入/输出/幻觉/事实）

| 子功能 | Flow 名称 | Rail | 原理 | 依赖 |
|--------|----------|------|------|------|
| 输入自检 | `self check input` | input | 用 LLM 判断用户输入是否违反安全规则（需配置 prompt 模板） | 仅需 LLM |
| 输出自检 | `self check output` | output | 用 LLM 判断 bot 回复是否违反安全规则（需配置 prompt 模板） | 仅需 LLM |
| 幻觉检测 | `self check hallucination` | output | 用同一 prompt 多次调用 LLM 生成回复，比较多次回复的自洽性，不一致则判定为幻觉 | 仅需 LLM |
| 事实核查 | `self check facts` | output | 将 bot 回复与检索到的证据（relevant_chunks）对比，用 LLM 判断是否与事实一致 | 仅需 LLM |

**配置示例：**
```yaml
models:
  - type: main
    engine: openai
    model: deepseek-r1:8b
    parameters:
      openai_api_base: http://localhost:11434/v1
  - type: self_check_input
    engine: openai
    model: qwen2.5:7b
    parameters:
      openai_api_base: http://localhost:11434/v1

rails:
  input:
    flows:
      - self check input $model=self_check_input
  output:
    flows:
      - self check output
      - self check hallucination
```

需要配合 `prompts.yml` 定义 prompt 模板（`self_check_input`、`self_check_output`、`self_check_hallucination`、`self_check_facts`）。

---

### 2. Injection Detection - 注入检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `injection detection` |
| Rail | input |
| 原理 | 使用 YARA 规则引擎进行文本模式匹配，检测已知的注入攻击模式 |
| 依赖 | `yara-python` |

**内置 YARA 规则：**
- **SQL 注入** (`sqli.yara`): 检测 `SELECT`、`DROP`、`UNION` 等 SQL 关键字 + 注释符 `--`、分号 `;` 等组合
- **XSS 跨站脚本** (`xss.yara`): 检测 `<script>` 标签、`javascript:` 链接、Markdown 嵌入链接
- **代码注入** (`code.yara`): 检测 `import os`、`import subprocess`、`import socket` 等危险模块导入
- **模板注入** (`template.yara`): 检测 Jinja2 模板语法 `{{ }}`、`{% %}`

**三种处理方式：**
- `reject` — 直接拦截整个输入
- `omit` — 删掉匹配部分，放行剩余内容
- `sanitize` — 尚未实现

**配置示例：**
```yaml
rails:
  config:
    injection_detection:
      injections:
        - code
        - sqli
        - template
        - xss
      action: reject
  input:
    flows:
      - injection detection
```

---

### 3. Regex Detection - 正则检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `regex check input` / `regex check output` / `regex check retrieval` |
| Rail | input / output / retrieval |
| 原理 | 用自定义正则表达式匹配文本内容 |
| 依赖 | 无（Python 标准库 `re`） |

**配置示例：**
```yaml
rails:
  config:
    regex_detection:
      input:
        patterns:
          - "\\b\\d{16}\\b"        # 信用卡号
          - "\\b\\d{3}-\\d{2}-\\d{4}\\b"  # SSN
      output:
        patterns:
          - "password\\s*[:=]"
  input:
    flows:
      - regex check input
```

---

### 4. Sensitive Data Detection - 敏感数据检测（Presidio）

| 项目 | 说明 |
|------|------|
| Flow 名称 | `detect sensitive data on input/output/retrieval`、`mask sensitive data on input/output/retrieval` |
| Rail | input / output / retrieval |
| 原理 | 使用 Microsoft Presidio 框架 + SpaCy NLP 模型识别和脱敏 PII（个人身份信息） |
| 依赖 | `presidio-analyzer`、`presidio-anonymizer`、`spacy`、`en_core_web_sm/lg` |

**支持的实体类型：**
- PERSON, EMAIL_ADDRESS, PHONE_NUMBER, CREDIT_CARD, US_SSN, LOCATION 等

**配置示例：**
```yaml
rails:
  config:
    sensitive_data_detection:
      input:
        score_threshold: 0.4
        entities:
          - PERSON
          - EMAIL_ADDRESS
          - PHONE_NUMBER
          - CREDIT_CARD
      output:
        score_threshold: 0.4
        entities:
          - PERSON
          - EMAIL_ADDRESS
  input:
    flows:
      - detect sensitive data on input
      - mask sensitive data on input
```

---

### 5. Attention - 用户注意力检测

| 项目 | 说明 |
|------|------|
| Action 名称 | `UpdateAttentionMaterializedViewAction`、`GetAttentionPercentageAction` |
| 原理 | 跟踪用户在对话过程中的注意力状态变化，计算注意力时间百分比 |
| 依赖 | 无 |

适用于多模态/语音场景，需要外部提供注意力事件流。

---

## 二、需要专用模型的检测方法

### 6. Content Safety - 内容安全

| 项目 | 说明 |
|------|------|
| Flow 名称 | `content safety check input $model=<model_name>` / `content safety check output $model=<model_name>` |
| Rail | input / output |
| 原理 | 使用专门的安全分类模型（如 Llama Guard）判断内容是否安全 |
| 依赖 | 需配置专用模型（如 NVIDIA NIM `nvidia/llama-3.1-nemoguard-8b-content-safety`） |

支持多语言拒绝消息（en/es/zh/de/fr/hi/ja/ar/th），支持 reasoning 模式。

**配置示例：**
```yaml
models:
  - type: main
    engine: openai
    model: gpt-4o
  - type: content_safety
    engine: nim
    model: nvidia/llama-3.1-nemoguard-8b-content-safety

rails:
  input:
    flows:
      - content safety check input $model=content_safety
  output:
    flows:
      - content safety check output $model=content_safety
```

---

### 7. Topic Safety - 话题安全

| 项目 | 说明 |
|------|------|
| Flow 名称 | `topic safety check input $model=<model_name>` |
| Rail | input |
| 原理 | 使用专用模型判断用户输入是否偏离允许的话题范围，回复 `on-topic` 或 `off-topic` |
| 依赖 | 需配置专用模型（如 NVIDIA NIM `llama-3.1-nemoguard-8b-topic-control`） |

---

### 8. Llama Guard - Llama Guard 安全检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `llama guard check input` / `llama guard check output` |
| Rail | input / output |
| 原理 | 使用 Meta Llama Guard 模型检测内容安全，返回 `safe`/`unsafe` + 违规策略编号 |
| 依赖 | 需配置 Llama Guard 模型 |

---

### 9. Jailbreak Detection - 越狱检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `jailbreak detection heuristics` / `jailbreak detection model` |
| Rail | input |
| 依赖 | `torch` + `transformers`（本地运行）或远程 `server_endpoint` |

**两种方式：**
- **启发式** (`jailbreak detection heuristics`): 用 GPT-2 计算输入文本的困惑度，检测异常模式
  - 长度/困惑度比（threshold: 89.79）
  - 前缀/后缀困惑度（threshold: 1845.65）
- **模型检测** (`jailbreak detection model`): 用 Snowflake embedding + 随机森林分类器判断是否为越狱

---

### 10. Hallucination (Patronus Lynx) - 幻觉检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `patronus lynx check output hallucination` |
| Rail | output |
| 原理 | 使用 Patronus Lynx 模型结合检索到的上下文判断 bot 回复是否存在幻觉 |
| 依赖 | 需配置 Patronus Lynx 模型 |

---

## 三、需要第三方云服务 API 的检测方法

### 11. ActiveFence

| 项目 | 说明 |
|------|------|
| Flow 名称 | 通过 `call activefence api` 调用 |
| Rail | input |
| 原理 | 调用 ActiveFence 云 API 检测内容安全 |
| 依赖 | `ACTIVEFENCE_API_KEY` 环境变量 |

**检测类别：** 骚扰、脏话、仇恨言论、儿童诱骗、暴力、自残、成人内容、隐私泄露

---

### 12. AutoAlign

| 项目 | 说明 |
|------|------|
| Flow 名称 | `autoalign input api` / `autoalign output api` / `autoalign groundedness output api` / `autoalign factcheck output api` |
| Rail | input / output |
| 依赖 | `AUTOALIGN_API_KEY` 环境变量 + AutoAlign 服务端点 |

**检测能力：** 毒性、偏见、伤害、PII、越狱、知识产权、机密信息、事实核查、幻觉

---

### 13. Cleanlab

| 项目 | 说明 |
|------|------|
| Flow 名称 | `call cleanlab api` |
| Rail | output |
| 原理 | 使用 Cleanlab TLM 计算回复的可信度分数，低于 0.6 则拦截 |
| 依赖 | `CLEANLAB_API_KEY` + `cleanlab-studio` 包 |

---

### 14. Clavata

| 项目 | 说明 |
|------|------|
| Flow 名称 | `ClavataCheckAction` |
| Rail | input / output |
| 原理 | 根据 Clavata 平台定义的策略评估内容，支持标签匹配逻辑（ALL/ANY） |
| 依赖 | Clavata 服务端点 + 策略配置 |

---

### 15. CrowdStrike AI Defense Runtime (AIDR)

| 项目 | 说明 |
|------|------|
| Flow 名称 | `crowdstrike aidr guard` |
| Rail | input / output |
| 原理 | 调用 CrowdStrike AIDR API 检测和转换 prompt/response |
| 依赖 | `CS_AIDR_TOKEN` 环境变量 |

---

### 16. Fiddler

| 项目 | 说明 |
|------|------|
| Flow 名称 | `call fiddler safety on user/bot message` / `call fiddler faithfulness` |
| Rail | input / output |
| 原理 | 调用 Fiddler API 检测安全性和忠实度 |
| 依赖 | `FIDDLER_API_KEY` + Fiddler 端点 |

**检测能力：** 安全性（有害/暴力/不道德/非法/性/种族歧视/越狱/骚扰/仇恨/性别歧视/角色扮演）、忠实度

---

### 17. GCP Text Moderation

| 项目 | 说明 |
|------|------|
| Flow 名称 | `call gcpnlp api` |
| Rail | input |
| 原理 | 调用 Google Cloud Natural Language API 的内容审核功能 |
| 依赖 | `google-cloud-language` 包 + GCP ADC 认证 |

**检测类别：** 毒性、侮辱、脏话、贬损、暴力、性、死亡/伤害/悲剧、武器、毒品、公共安全、健康、宗教、战争、政治、金融、法律

---

### 18. GLiNER - PII 检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `gliner detect pii` / `gliner mask pii` |
| Rail | input / output / retrieval |
| 原理 | 使用 GLiNER（通用零样本 NER）模型检测和脱敏 PII |
| 依赖 | GLiNER 服务端点 |

---

### 19. Guardrails AI

| 项目 | 说明 |
|------|------|
| Flow 名称 | `validate guardrails ai input/output` |
| Rail | input / output |
| 原理 | 动态加载 Guardrails AI Hub 的验证器（如正则、长度、PII 等） |
| 依赖 | `guardrails-ai` 包 + 具体 validator |

---

### 20. Pangea AI Guard

| 项目 | 说明 |
|------|------|
| Flow 名称 | `pangea ai guard` |
| Rail | input / output |
| 原理 | 调用 Pangea AI Guard API 检测和转换 prompt/response |
| 依赖 | `PANGEA_API_TOKEN` 环境变量 |

---

### 21. Patronus AI Evaluate

| 项目 | 说明 |
|------|------|
| Flow 名称 | `patronus api check output` |
| Rail | output |
| 原理 | 调用 Patronus Evaluate API 进行自定义评估 |
| 依赖 | `PATRONUS_API_KEY` 环境变量 |

---

### 22. PolicyAI

| 项目 | 说明 |
|------|------|
| Flow 名称 | `call policyai api` |
| Rail | input / output |
| 原理 | 调用 PolicyAI API 进行内容审核，返回 SAFE/UNSAFE |
| 依赖 | `POLICYAI_API_KEY` 环境变量 |

---

### 23. Private AI - PII 检测

| 项目 | 说明 |
|------|------|
| Flow 名称 | `detect pii` / `mask pii` |
| Rail | input / output / retrieval |
| 原理 | 调用 Private AI 服务检测和脱敏 PII |
| 依赖 | `PAI_API_KEY`（云）或自建 Private AI 端点 |

---

### 24. Prompt Security

| 项目 | 说明 |
|------|------|
| Flow 名称 | `protect prompt` / `protect response` |
| Rail | input / output |
| 原理 | 调用 Prompt Security Protect API 检测、阻止或修改 prompt/response |
| 依赖 | `PS_PROTECT_URL` + `PS_APP_ID` 环境变量 |

---

### 25. Trend Micro Vision One

| 项目 | 说明 |
|------|------|
| Flow 名称 | `trend ai guard` |
| Rail | input / output |
| 原理 | 调用 Trend Micro Vision One AI Guard API 检测内容安全 |
| 依赖 | Trend Micro API Key + 端点 |

---

## 四、按部署难度分类

### 最容易上手（仅需 LLM）
- Self Check（输入/输出/幻觉/事实核查）

### 需要轻量依赖
- Regex Detection（无额外依赖）
- Injection Detection（`yara-python`）
- Sensitive Data Detection（`presidio` + `spacy`）

### 需要专用模型
- Content Safety（NIM 专用模型）
- Topic Safety（NIM 专用模型）
- Llama Guard（Llama Guard 模型）
- Jailbreak Detection（`torch` + `transformers`）

### 需要第三方云服务
- ActiveFence、AutoAlign、Cleanlab、Clavata、CrowdStrike AIDR、Fiddler、GCP、GLiNER、Guardrails AI、Pangea、Patronus、PolicyAI、Private AI、Prompt Security、Trend Micro

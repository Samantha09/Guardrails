# NeMo Guardrails

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![PyPI](https://img.shields.io/pypi/v/nemoguardrails)](https://pypi.org/project/nemoguardrails)
[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/nemoguardrails)](https://pypi.org/project/nemoguardrails)
[![Tests/Linux](https://img.shields.io/github/actions/workflow/status/NVIDIA-NeMo/Guardrails/pr-tests.yml?logo=github&label=Tests%2FLinux)](https://github.com/NVIDIA-NeMo/Guardrails/actions/workflows/pr-tests.yml)
[![Tests/Windows](https://img.shields.io/github/actions/workflow/status/NVIDIA-NeMo/Guardrails/full-tests.yml?logo=github&label=Tests%2FWindows)](https://github.com/NVIDIA-NeMo/Guardrails/actions/workflows/full-tests.yml)
[![Tests/macOS](https://img.shields.io/github/actions/workflow/status/NVIDIA-NeMo/Guardrails/full-tests.yml?logo=github&label=Tests%2FmacOS)](https://github.com/NVIDIA-NeMo/Guardrails/actions/workflows/full-tests.yml)
[![Lint](https://img.shields.io/github/actions/workflow/status/NVIDIA-NeMo/Guardrails/lint.yml?logo=github&label=Lint)](https://github.com/NVIDIA-NeMo/Guardrails/actions/workflows/lint.yml)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Documentation](https://img.shields.io/badge/docs-nvidia.com-blue.svg)](https://docs.nvidia.com/nemo/guardrails)
[![arXiv](https://img.shields.io/badge/cs.CL-arXiv%3A2310.10501-b31b1b.svg)](https://arxiv.org/abs/2310.10501)
[![Downloads](https://static.pepy.tech/badge/nemoguardrails)](https://pepy.tech/project/nemoguardrails)
[![Downloads](https://static.pepy.tech/badge/nemoguardrails/month)](https://pepy.tech/project/nemoguardrails)

> **最新发布版本 / 开发版本**: [main](https://github.com/NVIDIA-NeMo/Guardrails/tree/main) 分支追踪最新的测试版本: [0.21.0](https://github.com/NVIDIA-NeMo/Guardrails/tree/v0.21.0)。如需获取最新开发版本，请检出 [develop](https://github.com/NVIDIA-NeMo/Guardrails/tree/develop) 分支。

✨✨✨

📌 **NeMo Guardrails 官方文档已迁移至 [docs.nvidia.com/nemo/guardrails](https://docs.nvidia.com/nemo/guardrails)。**

✨✨✨

NeMo Guardrails 是一个开源工具包，用于轻松地为基于 LLM（大语言模型）的对话应用添加*可编程护栏*。护栏（简称 "rails"）是控制大语言模型输出的特定方式，例如不讨论政治话题、以特定方式响应用户请求、遵循预定义的对话路径、使用特定的语言风格、提取结构化数据等等。

[这篇论文](https://arxiv.org/abs/2310.10501)介绍了 NeMo Guardrails 的技术概述和当前评估结果。

## 环境要求

Python 3.10、3.11、3.12 或 3.13。

NeMo Guardrails 使用了 [annoy](https://github.com/spotify/annoy)，这是一个带有 Python 绑定的 C++ 库。安装 NeMo Guardrails 前，你需要安装 C++ 编译器和开发工具。请查看 [安装指南](https://docs.nvidia.com/nemo/guardrails/getting-started/installation-guide.html#prerequisites) 获取各平台的具体安装说明。

## 安装

使用 pip 安装：

```bash
> pip install nemoguardrails
```

更多详细说明，请参阅 [安装指南](https://docs.nvidia.com/nemo/guardrails/getting-started/installation-guide.html)。

## 概述

<!-- start-documentation-reuse -->

NeMo Guardrails 使构建基于 LLM 应用的开发者能够轻松地在应用代码和 LLM 之间添加**可编程护栏**。

<div align="center">
  <img src="https://github.com/NVIDIA-NeMo/Guardrails/raw/develop/docs/_static/images/programmable_guardrails.png"  width="75%" alt="可编程护栏">
</div>

添加*可编程护栏*的主要好处包括：

- **构建可信、安全、有保障的 LLM 应用**: 你可以定义护栏来引导和保障对话安全；可以选择在特定主题上定义基于 LLM 的应用程序的行为，并防止其参与不想要的讨论。

- **安全地连接模型、链和其他服务**: 你可以无缝且安全地将 LLM 连接到其他服务（又称工具）。

- **可控对话**: 你可以引导 LLM 遵循预定义的对话路径，允许你按照对话设计最佳实践设计交互，并强制执行标准操作流程（例如，身份验证、支持）。

<!-- end-documentation-reuse -->

### 防范 LLM 漏洞

NeMo Guardrails 提供了多种机制来保护基于 LLM 的聊天应用免受常见 LLM 漏洞的攻击，例如越狱和提示注入。以下是本仓库中包含的示例 [ABC Bot](./examples/bots/abc) 的不同护栏配置所提供的保护概述。更多详情，请参阅 [LLM 漏洞扫描](https://docs.nvidia.com/nemo/guardrails/evaluation/llm-vulnerability-scanning.html) 页面。

<div align="center">
<img src="https://github.com/NVIDIA-NeMo/Guardrails/raw/develop/docs/_static/images/abc-llm-vulnerability-scan-results.png" width="500">
</div>

### 使用场景

你可以在多种类型的使用场景中使用可编程护栏：

1. **问答系统**（基于文档的检索增强生成）：强制执行事实检查和输出审核。
2. **领域特定助手**（又称聊天机器人）：确保助手保持主题并遵循设计的对话流程。
3. **LLM 端点**: 为你的自定义 LLM 添加护栏，实现更安全的客户交互。
4. **LangChain 链**: 如果你在任何使用场景中使用 LangChain，可以在链周围添加护栏层。

### 使用方法

要为应用添加可编程护栏，你可以使用 Python API 或护栏服务器（更多详情请参阅 [服务器指南](https://docs.nvidia.com/nemo/guardrails/user-guides/server-guide.html)）。使用 Python API 类似于直接使用 LLM。调用护栏层而不是 LLM 只需要对代码库进行极小的更改，只需两个简单步骤：

1. 加载护栏配置并创建 `LLMRails` 实例。
2. 使用 `generate`/`generate_async` 方法调用 LLM。

```python
from nemoguardrails import LLMRails, RailsConfig

# 从指定路径加载护栏配置
config = RailsConfig.from_path("PATH/TO/CONFIG")
rails = LLMRails(config)

completion = rails.generate(
    messages=[{"role": "user", "content": "Hello world!"}]
)
```

示例输出：

```json
{"role": "assistant", "content": "Hi! How can I help you?"}
```

`generate` 方法的输入和输出格式类似于 OpenAI 的 [Chat Completions API](https://platform.openai.com/docs/guides/gpt/chat-completions-api)。

#### 异步 API

NeMo Guardrails 是一个异步优先的工具包，因为其核心机制使用 Python 异步模型实现。公共方法同时提供同步和异步版本。例如：`LLMRails.generate` 和 `LLMRails.generate_async`。

### 支持的 LLM

你可以将 NeMo Guardrails 与多种 LLM 一起使用，如 OpenAI GPT-3.5、GPT-4、LLaMa-2、Falcon、Vicuna 或 Mosaic。更多详情，请查看配置指南中的 [支持的 LLM 模型](https://docs.nvidia.com/nemo/guardrails/user-guides/configuration-guide.html#supported-llm-models) 部分。

### 护栏类型

NeMo Guardrails 支持五种主要类型的护栏：

<div align="center">
  <img src="https://github.com/NVIDIA-NeMo/Guardrails/raw/develop/docs/_static/images/programmable_guardrails_flow.png"  width="75%" alt="可编程护栏流程">
</div>

1. **输入护栏**: 应用于用户输入；输入护栏可以拒绝输入，停止任何额外处理，或修改输入（例如，掩码潜在敏感数据、改写）。

2. **对话护栏**: 影响 LLM 的提示方式；对话护栏在规范形式消息上操作（详见 [Colang 指南](https://docs.nvidia.com/nemo/guardrails/user-guides/colang-language-syntax-guide.html)），并确定是否应执行操作、是否应调用 LLM 生成下一步或响应、是否应改用预定义响应等。

3. **检索护栏**: 在 RAG（检索增强生成）场景下应用于检索到的文本块；检索护栏可以拒绝一个文本块，防止其被用于提示 LLM，或修改相关文本块（例如，掩码潜在敏感数据）。

4. **执行护栏**: 应用于需要由 LLM 调用的自定义操作（又称工具）的输入/输出。

5. **输出护栏**: 应用于 LLM 生成的输出；输出护栏可以拒绝输出，防止其返回给用户，或修改它（例如，删除敏感数据）。

### 护栏配置

护栏配置定义了要使用的 **LLM(s)** 和 **一个或多个护栏**。护栏配置可以包含任意数量的输入/对话/输出/检索/执行护栏。没有任何配置护栏的配置将基本上把请求转发给 LLM。

护栏配置文件夹的标准结构如下：

```
.
├── config
│   ├── actions.py
│   ├── config.py
│   ├── config.yml
│   ├── rails.co
│   ├── ...
```

`config.yml` 包含所有常规配置选项，如 LLM 模型、活动护栏和自定义配置数据。`config.py` 文件包含任何自定义初始化代码，`actions.py` 包含任何自定义 Python 操作。完整概述请参阅 [配置指南](https://docs.nvidia.com/nemo/guardrails/user-guides/configuration-guide.html)。

以下是一个示例 `config.yml`：

```yaml
# config.yml
models:
  - type: main
    engine: openai
    model: gpt-3.5-turbo-instruct

rails:
  # 当收到用户新输入时调用输入护栏
  input:
    flows:
      - check jailbreak
      - mask sensitive data on input

  # 在生成机器人消息后触发输出护栏
  output:
    flows:
      - self check facts
      - self check hallucination
      - activefence moderation on input

  config:
    # 配置应在用户输入上掩码的实体类型
    sensitive_data_detection:
      input:
        entities:
          - PERSON
          - EMAIL_ADDRESS
```

护栏配置中包含的 `.co` 文件包含 Colang 定义（有关 Colang 的简要概述，请参见下一节），用于定义各种类型的护栏。以下是一个示例 `greeting.co` 文件，定义了问候用户的对话护栏。

```colang
define user express greeting
  "Hello!"
  "Good afternoon!"

define flow
  user express greeting
  bot express greeting
  bot offer to help

define bot express greeting
  "Hello there!"

define bot offer to help
  "How can I help you today?"
```

以下是针对侮辱性内容的对话护栏的额外 Colang 定义示例：

```colang
define user express insult
  "You are stupid"

define flow
  user express insult
  bot express calmly willingness to help
```

### Colang

为了配置和实现各种类型的护栏，本工具包引入了 **Colang**，一种专门为设计灵活但可控的对话流程而创建的建模语言。Colang 具有类似 Python 的语法，设计简单直观，特别适合开发者。

```{note}
支持 Colang 的两个版本 1.0 和 2.0，Colang 1.0 是默认版本。
```

有关 Colang 1.0 语法的简要介绍，请参阅 [Colang 1.0 语言语法指南](https://docs.nvidia.com/nemo/guardrails/user-guides/colang-language-syntax-guide.html)。

要开始使用 Colang 2.0，请参阅 [Colang 2.0 文档](https://docs.nvidia.com/nemo/guardrails/colang-2/overview.html)。

### 护栏库

NeMo Guardrails 附带一组[内置护栏](https://docs.nvidia.com/nemo/guardrails/user-guides/guardrails-library.html)。

```{note}
内置护栏可能适合也可能不适合特定的生产使用场景。与往常一样，开发者应与其内部应用团队合作，确保护栏满足相关行业和使用场景的要求，并解决意外的产品滥用问题。
```

该库包括用于 LLM 自检（输入/输出审核、事实检查、幻觉检测）、NVIDIA 安全模型（内容安全、主题安全）、越狱和注入检测的护栏，以及与社区模型和第三方 API 的集成。完整列表请参阅 [护栏库文档](https://docs.nvidia.com/nemo/guardrails/user-guides/guardrails-library.html)。

## CLI

NeMo Guardrails 还附带了内置的命令行工具。

```bash
$ nemoguardrails --help

Usage: nemoguardrails [OPTIONS] COMMAND [ARGS]...

actions-server    启动 NeMo Guardrails 操作服务器
chat              启动交互式聊天会话
evaluate          运行评估任务
server            启动 NeMo Guardrails 服务器
```

### 护栏服务器

你可以使用 NeMo Guardrails CLI 启动护栏服务器。服务器可以从指定文件夹加载一个或多个配置，并暴露 HTTP API 供使用。

```
nemoguardrails server [--config PATH/TO/CONFIGS] [--port PORT]
```

例如，要为 `sample` 配置获取聊天完成，你可以使用 `/v1/chat/completions` 端点：

```
POST /v1/chat/completions
```

```json
{
    "config_id": "sample",
    "messages": [{
      "role":"user",
      "content":"Hello! What can you do for me?"
    }]
}
```

示例输出：

```json
{"role": "assistant", "content": "Hi! How can I help you?"}
```

#### Docker

要启动护栏服务器，你也可以使用 Docker 容器。NeMo Guardrails 提供了一个 [Dockerfile](./Dockerfile)，你可以用它来构建 `nemoguardrails` 镜像。更多信息，请参阅 [使用 Docker](https://docs.nvidia.com/nemo/guardrails/user-guides/advanced/using-docker.html) 部分。

## 与 LangChain 集成

NeMo Guardrails 与 LangChain 无缝集成。你可以轻松地将护栏配置包装在 LangChain 链（或任何 `Runnable`）周围。你也可以在护栏配置中调用 LangChain 链。更多详情，请查看 [LangChain 集成文档](https://docs.nvidia.com/nemo/guardrails/user-guides/langchain/langchain-integration.html)

## 评估

评估基于 LLM 的对话应用的安全性是一项复杂的任务，仍然是一个开放的研究问题。为了支持适当的评估，NeMo Guardrails 提供了以下内容：

1. 一个[评估工具](nemoguardrails/evaluate/README.md)，即 `nemoguardrails evaluate`，支持主题护栏、事实检查、审核（越狱和输出审核）和幻觉检测。
2. 示例 LLM 漏洞扫描报告，例如 [ABC Bot - LLM 漏洞扫描结果](https://docs.nvidia.com/nemo/guardrails/evaluation/llm-vulnerability-scanning.html)

## 有何不同？

有多种方法可以为基于 LLM 的对话应用添加护栏。例如：显式审核端点（如 OpenAI、ActiveFence、PolicyAI）、批评链（如 constitutional chain）、解析输出（如 guardrails.ai）、单独的护栏（如 LLM-Guard）、RAG 应用的幻觉检测（如 Got It AI、Patronus Lynx）。

NeMo Guardrails 旨在提供一个灵活的工具包，可以将所有这些互补方法整合到一个有凝聚力的 LLM 护栏层中。例如，该工具包提供了与 ActiveFence、PolicyAI、AlignScore 和 LangChain 链的开箱即用集成。

据我们所知，NeMo Guardrails 是唯一一个还提供用户与 LLM 之间对话建模解决方案的护栏工具包。这实现了一方面能够精确引导对话的能力，另一方面能够实现对何时使用某些护栏的细粒度控制，例如，仅对特定类型的问题使用事实检查。

## 了解更多

- [文档](https://docs.nvidia.com/nemo/guardrails)
- [入门指南](https://docs.nvidia.com/nemo/guardrails/getting-started)
- [示例](./examples)
- [常见问题](https://docs.nvidia.com/nemo/guardrails/faqs.html)
- [安全指南](https://docs.nvidia.com/nemo/guardrails/security/guidelines.html)

## 邀请社区贡献

仓库中的示例护栏是很好的起点。我们热忱邀请社区共同努力，让可信、安全、有保障的 LLM 能力惠及每一个人。有关设置开发环境和如何为 NeMo Guardrails 贡献的指南，请参阅 [贡献指南](./CONTRIBUTING.md)。

## 许可证

本工具包根据 [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0) 许可。

## 如何引用

如果你使用了本工作，请引用介绍它的 [EMNLP 2023 论文](https://aclanthology.org/2023.emnlp-demo.40)。

```bibtex
@inproceedings{rebedea-etal-2023-nemo,
    title = "{N}e{M}o Guardrails: A Toolkit for Controllable and Safe {LLM} Applications with Programmable Rails",
    author = "Rebedea, Traian  and
      Dinu, Razvan  and
      Sreedhar, Makesh Narsimhan  and
      Parisien, Christopher  and
      Cohen, Jonathan",
    editor = "Feng, Yansong  and
      Lefever, Els",
    booktitle = "Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing: System Demonstrations",
    month = dec,
    year = "2023",
    address = "Singapore",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2023.emnlp-demo.40",
    doi = "10.18653/v1/2023.emnlp-demo.40",
    pages = "431--445",
}
```

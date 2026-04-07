# NeMo Guardrails 核心逻辑解析

本文档整理自项目代码分析，涵盖配置解析、护栏流程执行、嵌入模型等核心机制。

---

## 1. 项目架构概览

```
用户输入 → 输入护栏(Input Rails) → LLM → 输出护栏(Output Rails) → 用户
                ↓                        ↓
         嵌入模型(MiniLM)            嵌入模型(MiniLM)
                ↓                        ↓
         语义匹配/意图识别          事实检查/幻觉检测
```

---

## 2. 配置系统

### 2.1 配置文件结构

标准配置文件夹结构：
```
config/
├── config.yml          # 主配置文件
├── config.py           # 自定义初始化代码
├── actions.py          # 自定义动作
├── rails.co            # Colang 流程定义
└── ...
```

### 2.2 关键配置项

**config.yml 示例**：
```yaml
models:
  - type: main
    engine: openai
    model: gpt-3.5-turbo-instruct

rails:
  input:
    flows:
      - self check input
      - jailbreak detection heuristics
      - mask sensitive data on input
  
  output:
    flows:
      - self check output
      - self check facts
```

### 2.3 配置解析流程

**文件**: `nemoguardrails/rails/llm/config.py`

```python
# 1. 加载配置
RailsConfig.from_path("config/")

# 2. 解析过程
- 读取 config.yml
- 遍历目录加载所有 .co 文件 (Colang)
- 合并配置
- 验证配置有效性
- 返回 RailsConfig 实例
```

### 2.4 护栏流程配置类

```python
class InputRails(BaseModel):
    parallel: bool = False          # 是否并行执行
    flows: List[str] = []           # 启用的流程列表

class OutputRails(BaseModel):
    parallel: bool = False
    flows: List[str] = []
    streaming: OutputRailsStreamingConfig  # 流式配置
```

---

## 3. 嵌入模型 (Embedding Model)

### 3.1 默认模型

**模型**: `sentence-transformers/all-MiniLM-L6-v2`

| 属性 | 值 |
|------|-----|
| 参数量 | 2200万 (22M) |
| 输出维度 | 384 |
| 模型大小 | ~80MB |
| 执行环境 | CPU (ONNX Runtime) |

### 3.2 模型用途

1. **语义搜索**: 将用户输入转换为向量，匹配预定义的对话流程
2. **意图匹配**: 识别用户意图，触发相应的护栏规则
3. **知识检索**: RAG 场景中检索相关文档片段

### 3.3 代码位置

**文件**: `nemoguardrails/embeddings/basic.py`

```python
class BasicEmbeddingsIndex:
    def __init__(
        self,
        embedding_model: str = "sentence-transformers/all-MiniLM-L6-v2",
        embedding_engine: str = "SentenceTransformers",  # 或 FastEmbed
    ):
        # 使用 Annoy 进行高效的最近邻搜索
        self._index = AnnoyIndex(embedding_size, "angular")
```

**文件**: `nemoguardrails/embeddings/providers/fastembed.py`

```python
class FastEmbedEmbeddingModel:
    """使用 FastEmbed (ONNX Runtime) 进行嵌入"""
    
    async def encode_async(self, documents: List[str]) -> List[List[float]]:
        loop = asyncio.get_running_loop()
        # 在线程池中执行 CPU 密集型计算
        result = await loop.run_in_executor(
            get_executor(), 
            self.model.embed, 
            documents
        )
        return [x.tolist() for x in result]
```

---

## 4. 护栏执行机制

### 4.1 执行流程图

```
用户输入
    ↓
LLMRails.generate()
    ↓
RailsManager.is_input_safe()
    ↓
┌─────────────────────────────────────────┐
│  遍历 config.rails.input.flows          │
│                                         │
│  串行: _run_rails_sequential()          │
│  并行: _run_rails_parallel()            │
│                                         │
│  每个 flow → _run_input_rail(flow)      │
└─────────────────────────────────────────┘
    ↓
任一失败 → 拦截输入
全部通过 → 调用 LLM
```

### 4.2 核心类: RailsManager

**文件**: `nemoguardrails/guardrails/rails_manager.py`

```python
class RailsManager:
    def __init__(self, config: RailsConfig, model_manager: ModelManager):
        # 读取配置的流程列表
        self.input_flows: list[str] = list(config.rails.input.flows)
        self.output_flows: list[str] = list(config.rails.output.flows)
        
        # 执行模式
        self.input_parallel: bool = config.rails.input.parallel or False
```

### 4.3 输入安全检查 (异步)

```python
async def is_input_safe(self, messages: list[dict]) -> RailResult:
    """执行所有启用的输入护栏，首个失败时短路"""
    if not self.input_flows:
        return RailResult(is_safe=True)
    
    # 创建协程字典（注意：此时不执行！）
    rails = {
        flow: self._run_input_rail(flow, messages) 
        for flow in self.input_flows
    }
    
    # 根据配置选择执行策略
    if self.input_parallel:
        return await self._run_rails_parallel(rails, RailDirection.INPUT)
    return await self._run_rails_sequential(rails, RailDirection.INPUT)
```

### 4.4 串行执行

```python
async def _run_rails_sequential(
    self, 
    rails: Mapping[str, Coroutine]
) -> RailResult:
    """串行执行，按顺序等待，首个失败即返回"""
    for flow, coro in rails.items():
        result = await coro  # 逐个执行
        if not result.is_safe:
            return result      # 短路返回
    return RailResult(is_safe=True)
```

### 4.5 并行执行

```python
async def _run_rails_parallel(
    self, 
    rails: Mapping[str, Coroutine]
) -> RailResult:
    """并行执行，首个完成的不安全结果触发取消"""
    
    # 1. 将协程转换为 Task（立即开始执行）
    task_to_flow = {
        asyncio.create_task(coro): flow 
        for flow, coro in rails.items()
    }
    pending_tasks = set(task_to_flow.keys())
    
    try:
        while pending_tasks:
            # 2. 等待任意一个任务完成
            done, pending_tasks = await asyncio.wait(
                pending_tasks, 
                return_when=asyncio.FIRST_COMPLETED
            )
            
            # 3. 检查完成的任务
            for task in done:
                result = task.result()
                flow = task_to_flow[task]
                
                if not result.is_safe:
                    # 4. 取消所有剩余任务
                    for t in pending_tasks:
                        t.cancel()
                    await asyncio.wait(pending_tasks)
                    return result
                    
        return RailResult(is_safe=True)
```

### 4.6 流程分发

```python
async def _run_input_rail(self, flow: str, messages: list[dict]) -> RailResult:
    """根据流程名称分发到具体检查逻辑"""
    base_flow = _get_flow_name(flow)  # 去除 $model=xxx
    
    if base_flow == "content safety check input":
        return await self._check_content_safety_input(flow, messages)
    elif base_flow == "topic safety check input":
        return await self._check_topic_safety_input(flow, messages)
    elif base_flow == "jailbreak detection model":
        return await self._check_jailbreak_detection(messages)
    # ... 其他流程
```

---

## 5. 预定义流程类型

### 5.1 自检流程 (Self-Check)

依赖外部 LLM API 进行自我检查：

| 流程名称 | 类型 | 功能 |
|---------|------|------|
| `self check input` | 输入 | LLM 检查输入是否安全 |
| `self check output` | 输出 | LLM 检查输出是否安全 |
| `self check facts` | 输出 | 基于检索内容检查事实 |
| `self check hallucination` | 输出 | 检测幻觉 |

### 5.2 越狱与注入检测

| 流程名称 | 类型 | 实现方式 | GPU需求 |
|---------|------|---------|---------|
| `jailbreak detection heuristics` | 输入 | 启发式规则 | ❌ |
| `jailbreak detection model` | 输入 | FastEmbed + 随机森林 | ❌ |

### 5.3 敏感数据检测

使用 Presidio 或 GLiNER：

| 流程名称 | 类型 | 说明 |
|---------|------|------|
| `detect sensitive data on input` | 输入 | 检测敏感信息 |
| `mask sensitive data on input` | 输入 | 掩码敏感信息 |
| `detect sensitive data on output` | 输出 | 检测敏感信息 |
| `mask sensitive data on output` | 输出 | 掩码敏感信息 |

---

## 6. 执行环境对比

### 6.1 CPU 环境（默认 Dockerfile）

**可用流程**（无需 GPU）：
- ✅ 所有 `self check *` 流程（需 OpenAI API）
- ✅ `jailbreak detection heuristics`
- ✅ `jailbreak detection model`（FastEmbed CPU 版本）
- ✅ `detect/mask sensitive data`
- ✅ `regex check`
- ✅ `gliner detect/mask pii`

### 6.2 GPU 环境（本地模型）

需要 GPU 的流程（如果使用本地模型）：
- ⚡ `jailbreak detection model`（使用 Snowflake 嵌入模型时）
- ⚡ `alignscore check facts`（AlignScore 服务）
- ⚡ `llama guard check`（本地部署 Llama Guard）
- ⚡ `content safety check`（本地部署 Nemotron）

---

## 7. 关键代码文件索引

| 文件 | 职责 |
|------|------|
| `nemoguardrails/rails/llm/config.py` | 配置定义、解析、验证 |
| `nemoguardrails/guardrails/rails_manager.py` | 护栏流程执行编排 |
| `nemoguardrails/rails/llm/llmrails.py` | 主入口 LLMRails 类 |
| `nemoguardrails/embeddings/basic.py` | 嵌入索引实现 |
| `nemoguardrails/embeddings/providers/fastembed.py` | FastEmbed 嵌入引擎 |
| `nemoguardrails/library/*/flows.co` | 预定义护栏流程（Colang） |
| `nemoguardrails/library/*/actions.py` | 流程对应的 Python 动作 |

---

## 8. 异步模式说明

NeMo Guardrails 是一个**异步优先**的框架：

```python
# 所有核心方法都是 async
async def is_input_safe(...) -> RailResult
async def is_output_safe(...) -> RailResult
async def _run_input_rail(...) -> RailResult
async def generate_async(...) -> dict

# 使用方式
result = await rails_manager.is_input_safe(messages)
```

使用 `asyncio` 实现并发：
- `asyncio.create_task()` - 创建并发任务
- `asyncio.wait(FIRST_COMPLETED)` - 等待首个完成
- `task.cancel()` - 取消剩余任务

---

*文档基于代码版本 0.21.0 整理*

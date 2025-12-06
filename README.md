# qwen-rag-eval-scaffold

面向千问 Qwen API 的轻量级 RAG 评测脚手架，用一条命令跑通“数据集 → 向量库 → RAG 工作流 → RAGAS 评估”，并提供可视化控制台。适合作为在 Qwen 生态下搭建和评估 RAG 系统的基线工程。

## 项目特点

1. 原生支持千问 API  
   默认集成 DashScope embedding 与 Qwen 系列对话模型，只需提供 `API_KEY_Qwen` 即可运行，无需额外适配。

2. 覆盖 RAG 全链路  
   从基准数据集、样本抽取、上下文切分，到 Chroma 向量库构建、RAG 工作流执行与 RAGAS 评估，完整打通一条可复用链路。

3. 数据集与工作流可插拔  
   只要满足约定的样本格式和 Runner 协议，就可以替换数据集与 RAG 工作流，评估侧代码保持不变。

4. 内置中文基准数据集  
   开箱即用地支持 CMRC2018，提供从 raw 数据到 samples、chunks 和向量库的完整处理流程。

5. 自带 Streamlit 控制台  
   内置简单控制台，用于本地批量评估、单条结果检查以及在线 RAG 问答，方便调试与展示。

## 环境与依赖

1. Python 版本  
   需要 Python 3.10 及以上。

2. 安装依赖  
   项目根目录下已经提供 `requirements.txt`，可直接安装全部依赖：

   ```bash
   pip install -r requirements.txt
   ```

3. 配置千问 API Key（必须）  
   本项目通过环境变量 `API_KEY_Qwen` 读取 DashScope Key。示例：

   Windows 当前会话临时设置：

   ```bat
   set API_KEY_Qwen=your-dashscope-api-key
   python quick_start.py
   ```

   Linux / macOS：

   ```bash
   export API_KEY_Qwen="your-dashscope-api-key"
   python quick_start.py
   ```

   建议在系统环境变量或 shell 配置文件中持久设置，确保任意终端启动 Python 时都能读到该变量。

## Quick Start：一条命令跑通

在项目根目录执行：

```bash
python quick_start.py
```

该命令完成以下步骤：

1. 构建 CMRC 基准数据与向量库  
   CMRC2018 原始数据 → 评估样本（samples）→ 切分后的 chunks → Chroma 向量库。

2. 加载评估样本  
   从 `dataset.samples_path` 读取 `question / ground_truth / ground_truth_context`。

3. 使用 `DefaultRunner` 批量运行 RAG  
   内部使用约定好的 NormalRag 工作流（可自行替换实现）。

4. 使用 RAGAS 评估  
   在控制台输出整体指标，并按 `evaluation.output_csv` 写出逐样本评分 CSV。

## 使用方式概览

### 一、改写默认 RAG workflow

1. 编辑 `qwen_rag_eval/runner/normal_rag.py` 的 `_build_graph`，在其中增删 LangGraph 节点（例如 rewrite、rerank、多跳检索等），控制整条 RAG 工作流。

2. 保持以下约定不变，即可继续复用评估组件：  
   1）构造函数签名：`__init__(retriever, answer_generator)`  
   2）输出字段：`question / contexts / generation`  
   只要 Runner 的 `invoke` 返回结构满足协议，评估侧不需要修改。

### 二、自定义 Runner 并接入评估

1. 实现满足协议的 Runner：

   ```python
   def invoke(self, question: str) -> dict:
       return {
           "question": question,
           "generation": "...",        # 模型最终回答
           "contexts": [...],          # str 或带 page_content 的对象
       }
   ```

2. 在评估侧使用统一接口：

   ```python
   from qwen_rag_eval.evaluation import RagBatchRunner, RagasEvaluator

   runner = MyRunner()
   records = RagBatchRunner(runner, mode="my_runner").run_batch(eval_samples)
   result = RagasEvaluator("config/application.yaml").evaluate(records)
   ```

   其中 `records` 是规范化后的 `RagEvalRecord` 列表，可直接传给 RAGAS。

### 三、替换数据集并重建向量库

1. 按 `datasets/DATASET_README.md` 准备样本文件 `samples.json`，至少包含字段：`id / question / ground_truth / ground_truth_context`。

2. 通过样本路径直接构建向量库：

   ```python
   from qwen_rag_eval.vector import build_vector_store_from_samples

   build_vector_store_from_samples(
       "path/to/samples.json",
       collection_name="my_collection",
       overwrite=True,
   )
   ```

   函数会自动切分 `ground_truth_context` 为 chunks，并写入 Chroma 向量库。

3. 也可以通过配置驱动构建：

   在 `config/application.yaml` 中修改 `dataset.*` 与 `vector_store.*` 后，调用：

   ```python
   from qwen_rag_eval import build_cmrc_dataset_and_vector_store

   build_cmrc_dataset_and_vector_store("config/application.yaml")
   ```

   实际流程为：raw → samples → chunks → VectorStore。

### 四、Streamlit 控制台

安装 Streamlit 后，在项目根目录运行：

```bash
streamlit run app/streamlit_app.py --server.address 0.0.0.0 --server.port 8501
```

控制台提供三类能力：

1. 批量评估  
   1）在侧栏指定 `application.yaml` 与评估样本数。  
   2）一键构建或刷新 CMRC 向量库。  
   3）点击“运行评估”，执行数据构建 → 批量生成答案 → RAGAS 评分。  
   4）页面展示整体指标、逐样本评分表以及 CSV 路径。

2. 单条结果查看  
   在已有评估结果的前提下，根据索引查看单条样本的 question / answer / ground_truth / contexts 与各项指标。

3. 在线 RAG 问答  
   复用当前配置下的 `DefaultRunner`，输入任意问题，查看生成答案与检索到的上下文，用于直观感受 RAG 工作流行为。

## 配置要点（application.yaml）

1. 数据相关 `dataset.*`  
   1）`raw_path`：原始数据路径。  
   2）`samples_path`：评估样本文件（json）。  
   3）`chunks_path`：切分后的 chunks jsonl 路径。  
   4）`sample_limit`：抽样数量（Quick Start 中使用）。

2. 向量库相关 `vector_store.*`  
   1）`embedding_model`：向量模型名称，默认 `text-embedding-v4`。  
   2）`persist_directory`：Chroma 持久化目录。  
   3）`collection_name`：集合名，用于检索和构建。

3. 检索相关  
   1）`retrieval.top_k`：召回条数。

4. 评估相关 `evaluation.*`  
   1）`output_csv`：逐样本结果 CSV 输出路径。  
   2）可选字段用于覆盖评估阶段使用的 `llm_model`、`embedding_model`。

5. agents 配置 `config/agents.yaml`  
   1）`default_answer_generator`：默认回答生成模型、温度、解析方式与 Prompt。  
   2）可按需增加其他 agent（例如重写、排序、总结等）。

## 关键目录结构

1. `datasets/`  
   原始与处理后数据，以及数据格式说明文档。

2. `qwen_rag_eval/dataset_tools/`  
   样本抽样、文本切分、数据加载等工具函数。

3. `qwen_rag_eval/vector/`  
   向量库管理器、构建脚本，以及与配置文件的对接逻辑。

4. `qwen_rag_eval/runner/`  
   默认 Runner、NormalRag 工作流以及可扩展的 Runner 协议实现。

5. `qwen_rag_eval/evaluation/`  
   批量执行器 `RagBatchRunner`、RAGAS 封装 `RagasEvaluator` 与 `EvalResult`。

6. `app/streamlit_app.py`  
   用于本地调试与展示的前端控制台入口。

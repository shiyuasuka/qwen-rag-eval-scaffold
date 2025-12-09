English | [中文说明](README_zh.md)
# rag-eval-scaffold

A lightweight RAG evaluation scaffold that runs the full pipeline — “dataset → vector store → RAG workflow → evaluation” — with a single command. It comes with a Streamlit-based lightweight visualization console, and exposes a decoupled vector-store management layer (VectorManager), RAG Runner layer, and evaluation layer, all connected via a unified `invoke` interface.

The default example ships with a Chinese QA dataset and a basic RAG workflow, plus configuration examples for model services (Qwen by default). The overall design is not tied to any single vendor; you can plug in any compatible chat and embedding service via configuration.

## 1. Project Positioning and Target Users

1. As a RAG / LLM engineering scaffold  
   For users with experimentation or engineering needs, this project helps you, under a unified protocol, to:  
   1) build and manage vector stores;  
   2) orchestrate one or more RAG workflows that output a unified structure;  
   3) call the evaluation engine to compare different workflows, datasets, and metrics.

2. As a 0-to-1 learning project for RAG  
   For users who lack end-to-end engineering experience, this project already stitches together the full pipeline “dataset → vector store → retrieval → generation → evaluation”.  
   You can start from the default implementation and gradually replace or extend modules, without constantly jumping between different component documentations.

## 2. Core Features (Current Version)

1. One-line vector-store construction and VectorManager retrieval  
   Through a unified entry point, you can in a single line:  
   1) load the benchmark dataset according to configuration;  
   2) build or update the vector store;  
   3) obtain a VectorManager instance for retrieval, document CRUD, and retriever construction.  
   Configuration is centralized in `config/application.yaml`, including data paths, chunking strategy, vector-store persistence path, and more.

2. Full decoupling of dataset/vector layer, RAG layer, and evaluation layer  
   1) The dataset/vector layer only handles “samples → chunks → vector store” and exposes VectorManager upwards.  
   2) The RAG layer only cares about “how to use VectorManager for retrieval and call the LLM to form a workflow”, and exposes results via a Runner’s `invoke`.  
   3) The evaluation layer depends only on the Runner protocol and sample format, and is agnostic to the underlying vector store and workflow implementation, making it easy to swap datasets, models, and pipelines.

3. Unified `invoke` protocol  
   1) The vector-store management tool provides retrieval via a unified interface (for example, `VectorManager().invoke(query: str, k: int = 5) -> List[Document]`), focusing solely on “given a query, which documents are returned”;  
   2) All Runners implement a unified `Runner().invoke(question: str) -> dict` interface and return an agreed structure (question, generation, contexts, etc.);  
   3) The evaluation engine always starts evaluation via `EvalEngine().invoke(runner)`, which internally handles batch execution and concrete evaluation methods.

4. Low entry barrier and high extensibility  
   The default chunking logic and workflow structure are kept as straightforward as possible, so beginners can read and modify the code directly:  
   1) Beginners can start from the default chunking and workflow, tweak prompts, retrieval strategies, or add/remove simple nodes;  
   2) Advanced users can completely ignore the default implementation, take a retriever from VectorManager, implement a protocol-compliant Runner (keeping the `invoke` signature and output structure), and plug it directly into the evaluation layer to compare different RAG schemes.

## 3. Installation and Environment

1. Python version  
   Python 3.10 or above is recommended.

2. Install dependencies  

   In the project root:

   ```bash
   pip install -r requirements.txt
   ```

3. Configure models and API keys  

   1) In the configuration file, select or declare the model provider and model names (chat model, embedding model, etc.);  
   2) Set the corresponding API key in your system environment variables, for example:

   On Windows (current session):

   ```bat
   set API_KEY_XXX=your-api-key
   python quickstart.py
   ```

   On Linux or macOS:

   ```bash
   export API_KEY_XXX="your-api-key"
   python quickstart.py
   ```

   The exact environment variable names and configuration examples are defined in `config/application.yaml` and related docs.

## 4. Quick Start: Run the Full Pipeline with One Command

In the project root:

```bash
python quickstart.py
```

With the web frontend:

```bash
streamlit run streamlit_app.py
```

The default flow includes:

1. Build the benchmark dataset and vector store  
   1) Build evaluation samples from raw data;  
   2) Chunk the context fields according to configuration to produce chunks;  
   3) Build or update the local vector store and persist it to the configured directory.

2. Load evaluation samples  
   Load normalized samples from `dataset.samples_path`.

3. Run the default Runner in batch RAG mode  
   The default Runner uses VectorManager internally for context retrieval and the configured model for answer generation,  
   and outputs fields like `question`, `generation`, and `contexts` according to the agreed schema.

4. Run the evaluation engine  
   1) Normalize Runner outputs into standard record structures;  
   2) Use RAGAS and other evaluation methods to compute global and per-sample metrics;  
   3) Print results to the console and optionally export CSV files as configured.

With dependencies, configuration, and API keys set correctly, this single command runs the full loop from data to evaluation report.

## 5. Advanced Usage Overview

This section focuses on concepts and interface contracts. See the example scripts in the repository for concrete usage.

### 5.1 Using VectorManager for Custom Retrieval and Management

A typical flow:

1. Use the unified entry `VectorDatabaseBuilder().invoke()` to construct the vector store and obtain a VectorManager (internally loading configuration from `config/application.yaml` by convention).  

2. Use VectorManager’s methods to:  
   1) append new documents under conditions (`add_documents(...)`);  
   2) delete or rebuild collections (`delete_collection(...)`);  
   3) get a retriever or directly retrieve a list of documents (`get_retriever(...)` or `invoke(...)`);  
   4) combine with your own models to perform QA or other analysis.

VectorManager hides the underlying vector-store implementation details, so RAG workflows only depend on a unified retrieval interface.

Example:

```python
from rag_eval import VectorDatabaseBuilder

# Build the vector store according to configuration and get its manager (VectorManager)
vector_manager = VectorDatabaseBuilder().invoke()

# Use invoke for similarity search; returns a list of documents
docs = vector_manager.invoke("Which two companies co-developed Sengoku Musou 3?", k=5)
```

If you want to use a custom dataset, please refer to `rag_eval/dataset_tools/__init__.py`.

### 5.2 Implementing Custom Runners and Plugging into the Evaluation Layer

Runner protocol (simplified):

```python
class MyRunner:
    def __init__(self, vector_manager, ...):
        self.vector_manager = vector_manager
        # other model, prompt, and workflow configs

    def invoke(self, question: str) -> dict:
        """Return a structured result for the evaluation layer:

        question:   original question
        generation: final model answer (string)
        contexts:   list of retrieved contexts used to answer this question
        """
        ...
        return {
            "question": question,
            "generation": answer_text,
            "contexts": context_list,
        }
```

The evaluation layer depends only on the input–output protocol of `invoke`, so:  
1) as long as the returned structure matches the contract, it can be evaluated;  
2) model choice, prompt composition, and whether you use multi-step workflows are all opaque to the evaluation layer.

### 5.3 Evaluating with the Unified Evaluation Engine

Typical entry point (simplified):

```python
from rag_eval import EvalEngine

runner = MyRunner(...)
eval_result = EvalEngine().invoke(runner)
eval_result.show_console(top_n=5)
```

Internally, EvalEngine:

1. Loads evaluation samples (optionally controlled by configuration, such as sample limits);  
2. Calls the Runner’s `invoke` in batch;  
3. Invokes RAGAS and other metrics;  
4. Aggregates global and per-sample metrics and provides console and frontend display helpers.

### 5.4 Using the Streamlit Frontend for Debugging and Demo (Optional)

To use the graphical interface for debugging or demo:

1. In the project root:

   ```bash
   streamlit run streamlit_app.py
   ```

2. The frontend typically provides two modes:  
   1) Evaluation mode  
      1) Runs the evaluation flow under the current configuration and shows global metrics and per-sample scores;  
      2) Lets you choose the number of samples and trigger evaluation, with progress and estimated time displayed on the page;  
      3) Supports inspecting a single sample’s question, ground_truth, generation, and contexts.  
   2) Chat mode  
      1) Reuses the current Runner configuration for interactive RAG QA;  
      2) Helps you qualitatively inspect retrieval and answer quality alongside quantitative evaluation results.

The frontend does not change the underlying interface contracts. It simply wraps the existing Runner, evaluation engine, and results into an easy-to-use control panel.

## 6. Configuration and Directory Structure (Overview)

The actual layout may evolve across versions; always refer to the repository for the latest structure. A typical layout:

```text
config/
  application.yaml          # Global config: datasets, vector store, models, evaluation, etc.
  agents.yaml               # Model roles, prompts, agent configs, etc.

datasets/
  raw/                      # Raw datasets
  processed/                # Processed samples / chunks

rag_eval/
  __init__.py
  core/                     # Core types and interface definitions
    __init__.py
    interfaces.py
    types.py
  dataset_tools/            # Tools from raw to samples and chunks
    __init__.py
    cmrc2018/
      ...                   # CMRC2018-related scripts
  embeddings/
    __init__.py
    factory.py              # Embedding factory and wrappers
  eval_engine/              # Batch execution and evaluation engine (EvalEngine, etc.)
    __init__.py
    engine.py
    eval_result.py
    rag_batch_runner.py
    ragas_eval.py
  rag/                      # Default RAG workflow and Runner
    __init__.py
    normal_rag.py
    runner.py
  vector/                   # Vector-store builder and VectorManager
    __init__.py
    vector_builder.py
    vector_store_manager.py

utils/
  ...                       # Common utilities

quickstart.py               # One-command end-to-end example
quickstart.ipynb            # Corresponding notebook example (optional)
streamlit_app.py            # Streamlit console entry
```

Main configuration lives in `config/application.yaml`, typically including:

1. Dataset paths and sample size limits;  
   2. Vector-store backend config (persistence directory, collection name, embedding model, etc.);  
   3. Retrieval parameters (for example, top_k);  
   4. Model and API settings;  
   5. Evaluation output paths and evaluation parameters.

## 7. Current Status and Roadmap (Overview)

1. The current version already provides:  
   1) full decoupling of dataset/vector layer, RAG layer, and evaluation layer;  
   2) one-line vector-store construction and VectorManager-based management;  
   3) a unified `invoke` protocol shared by Runners and the evaluation engine;  
   4) a one-command end-to-end run plus advanced customization entry points and improved frontend progress feedback;  
   5) a Streamlit console for debugging and demo.

2. Planned extensions include:  
   1) additional benchmark datasets and example Runners;  
   2) more evaluation metrics and richer configuration for evaluation;  
   3) more complete configuration templates and examples for different model services.

If you only want to quickly run an end-to-end RAG-plus-evaluation pipeline, start from `quickstart.py`.  
If you care more about engineering and extensibility, start from VectorManager, the Runner protocol, and the Streamlit frontend, and gradually replace or extend the modules.

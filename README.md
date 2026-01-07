## License

**This software is licensed for non-commercial use only.**  
Commercial use requires a separate agreement.

---
**Disclaimer: **
This was built on open source OSSCode (open source version of vscode). However, everything about the chat logic is proprietary and built from scratch. Added code uses some built-in tools and hooks partially into the chatui for display. NativeChat system was mostly bypassed for more freedom in controlling UI Behavior.

# codeOSS-LMStudio-Ollama

**Built only for local models.**  
Requires **LM Studio** (embeddings) and **Ollama** (local / “local-cloud” models).  
Built on a stripped-down open-source VS Code (Code OSS).

---

## About

**codeOSSLOCAL** is an experimental, open-source project exploring integrations between Visual Studio Code and locally-hosted large language models (LLMs). It's a collaboration between a human and AI focused on tooling, routing, and novel context strategies for local models.

---

## Key Ideas

- Local model integrations and tooling (**Ollama**, **LM Studio**).
- Context-selection strategies to prioritize and surface relevant context.

---

## Goals

- Improve context management for local LLMs so they can use IDE context effectively.
- Provide easy ways to experiment with model stacks and routing strategies.

---

## Best Results So Far

> **Models tested (examples):**  
> - GPT-OSS 120B (Cloud)  
> - GPT-OSS 20B (Cloud)  
> - GPT-OSS 20B (Local, tested on MacBook)  
> - Qwen 235 (Cloud)  
> - DeepSeek 480 (Cloud)

---

## Issues

- Any model with a context window **≤ 8192 tokens** tends to **hallucinate tools** (often refusing to use tool calls). These models may rely only on the context directly fed to them instead of invoking tools.

---

# Deep Dive

## Detailed Summary — Local-LLM Implementation in codeOSS4LOCALLLMs

---

## 1) Entry Point — `LocalLLMHandler`

**File:** `src/vs/workbench/services/chat/localLLMHandler.ts`  
**Components:** `ILocalLLMHandler` (DI decorator) and `LocalLLMHandler` class

### What it does

- Implements `ILocalLLMHandler.invoke(request, progress, history, token)`.
- Acts as a thin bridge between the VS Code chat UI and the **CognitiveCore** brain.
- In the constructor it receives many services (language-model, file, logging, embedding, environment, etc.) and creates the brain subsystems:
  - **BrainOrchestrator** — execution engine that runs a request through the brain and model
  - **CognitiveCore** — unified brain orchestrator (see §2)

### Initialization strategy

- Lazily initializes heavy resources via `_initPromise`.
- Ensures resources are ready before each request with `_ensureInitialized`.

### Interaction flow

1. Chat UI → `ILocalLLMHandler.invoke`
2. Handler calls `BrainOrchestrator.initialize(message, modelId)`
3. `BrainOrchestrator.run` uses `CognitiveCore` to build a prompt, runs the local model, and processes any tool calls emitted by the model
4. Returns the final `IChatAgentResult` to the UI

---

## 2) Cognitive Core — Unified Brain (`cognitiveCore.ts`)

**File:** `src/vs/workbench/services/chat/cognitiveCore.ts`  
**Components:** `CognitiveCore` class, `ICognitiveContext`, `ICognitiveState`

### Subsystems (brain-inspired components)

| Subsystem | File | Responsibility |
|---|---|---|
| CorpusCallosum | `brain/corpusCallosum.ts` | Pub-sub message bus (topics, blackboard) |
| HippocampusMemory | `brain/hippocampusMemory.ts` | Long-term persistent pattern store; semantic search |
| LimbicSystem | `brain/limbicSystem.ts` | Emotional/motivational bias (satisfaction, curiosity, confidence) |
| MetacognitiveMonitor | `brain/metacognition.ts` | Self-reflection, strategic advice, execution context |
| PredictiveBrain | `brain/predictiveBrain.ts` | Pre-flight risk assessment; prediction of tool-call outcomes |
| MirrorNeuronSystem | `brain/mirrorNeurons.ts` | Learns style hints from existing code (naming, formatting) |
| ConsolidationEngine | `brain/consolidation.ts` | Background optimization; periodic compaction of memory |
| AssociativeSieve | `brain/associativeSieve.ts` | Semantic context filtering using vector embeddings (initialized later by handler) |
| CortanaOrchestrator | `brain/cortana.ts` | Central executive coordinating subsystems during a request |
| DynamicContextManager | `brain/dynamicContext.ts` | Per-request mutable context (blackboard entries) |

### Core API

- `initialize()`  
  Loads persistent hippocampal data; starts consolidation.

- `prepareContext(userRequest)` → builds `ICognitiveContext` containing:
  - Relevant patterns (top-k from Hippocampus)
  - Strategic advice (Metacognition)
  - Emotional bias (Limbic)
  - Style hints (MirrorNeuron)
  - `PromptContext` — concatenated string injected into the LLM prompt

- `afterExecution(request, result, reflections)`  
  Records reflections, updates hippocampus, triggers consolidation.

- `assessRisk(request)`  
  Uses PredictiveBrain to estimate failure probability before invoking the model.

- `observeCorrection(correction)`  
  Feeds user-provided corrections back into Hippocampus and Metacognition.

- `getStateSnapshot()`  
  Returns `ICognitiveState` for debugging/telemetry.

### Embedding integration

- `AssociativeSieve` starts as `null`.
- `LocalLLMHandler` injects `ILocalEmbeddingService` later, enabling semantic pruning of irrelevant context before prompt injection.

---

## 3) Model Provider — Ollama Local (`ollamaLocalModelContribution.ts`)

**File:** `src/vs/workbench/services/chat/ollamaLocalModelContribution.ts`  
**Components:** `OllamaLocalModelProvider`, registration logic

### Key points

- Registers built-in vendor `ollama-local` with `ILanguageModelsService`.
- Implements `ILanguageModelProvider` that:
  - Discovers model binaries in a user-configurable folder (`getModels`)
  - Provides `createModel` to spawn the local Ollama binary (or load a static `.gguf`) and return an `ILanguageModel` compatible with VS Code chat
- Provider is instantiated during activation; selected when user picks “Local Ollama” in the chat UI.
- No network traffic required — runs entirely on the host machine.

---

## 4) Tooling — Internal Model-Callable Tools (`localTools.ts`)

**File:** `src/vs/workbench/services/chat/localTools.ts`  
**Tools:** `LearnFactToolData`, `RecallMemoryToolData`, `ManageTodoToolData`, `SequentialThinkingToolData`, `WriteFileToolData`, `CreateFileToolData`

### Purpose

Expose model-side functions that the LLM can invoke via VS Code tool-calling. Tools are:

- `source: ToolDataSource.Internal`
- `canBeReferencedInPrompt: true`

So the model can reference them in its plan, and the orchestrator will execute them.

### Tool catalog

| Tool ID | Model description | Primary effect |
|---|---|---|
| `learn_fact` | Save a fact to project memory | Persists fact/pattern/file into `HippocampusMemory` (vectorized via `ILocalEmbeddingService`) |
| `recall_memory` | Retrieve facts from memory | Queries hippocampus (semantic search, last-N, first-N) |
| `manage_todo_list` | Create/update a session task list | Stores transient todo list in blackboard (for multi-step planning) |
| `sequential_thinking` | Record thinking steps | Enables numbered steps for structured reasoning |
| `write_file` | Write file content (overwrite optional) | Uses `IFileService` to write content to a workspace file |
| `create_file` | Create a new file (no overwrite by default) | Uses `IFileService` to create a new file with initial content |

### Execution path

1. Model emits tool-call JSON
2. `BrainOrchestrator` looks up tool in `ILanguageModelToolsService`
3. Implementation executes via VS Code services (file, memory, etc.)—typically in `brainExecutor.ts` (or similar)
4. Tool results are fed back into the model mid-turn

---

## 5) Brain Orchestrator & Execution (`brainExecutor.ts`, `brainRouter.ts`)

**Files:**
- `src/vs/workbench/services/chat/brainExecutor.ts` — `BrainExecutor`
- `src/vs/workbench/services/chat/brainRouter.ts` — `BrainRouter`

### Responsibilities

- **Routing:** decides whether to handle request locally (LocalLLMHandler) or forward to remote provider.
- **Execution:** `BrainExecutor.run(request, progress, history, token)`:
  1. Calls `CognitiveCore.prepareContext` → builds prompt
  2. Sends prompt to selected model (Ollama local or remote)
  3. Listens for tool-call messages → dispatches via `ILanguageModelToolsService`
  4. Calls `CognitiveCore.afterExecution` to store reflections, update memory, and trigger consolidation
- **State management:** maintains per-request `ICognitiveContext` and `IExecutionContext` shared via the blackboard

---

## 6) Embedding Service (`localEmbeddingService.ts`)

**File:** `src/vs/workbench/services/chat/localEmbeddingService.ts`  
**Component:** `ILocalEmbeddingService` implementation

### What it does

- Provides vector embeddings for facts stored via `learn_fact`.
- Powers `AssociativeSieve` to filter long-term patterns by semantic similarity.
- Lazy-loaded by `LocalLLMHandler` and injected after construction  
  (see note in `cognitiveCore.ts`: `// @ts-ignore - Will be initialized in LocalLLMHandler with embedding service`)

---

## 7) Overall Data Flow (Simplified Diagram)

---

---

## 8) Key Design Characteristics

| Aspect | Description |
|---|---|
| Modularity | Each cognitive function is isolated; assembled in `CognitiveCore` |
| Dependency Injection | Built on VS Code `createDecorator` / `IInstantiationService` |
| Local-only execution | Ollama provider loads from disk; no external API required |
| Tool-calling loop | Model can request actions mid-turn (write files, memory updates) |
| Persisted long-term memory | Hippocampus persists patterns (e.g., `.vscode/brainMemory.json`-like file) |
| Semantic filtering | `AssociativeSieve` reduces prompt size by selecting relevant facts |
| Self-reflection | Metacognitive monitor stores reflections post-turn |
| Risk assessment | Predictive brain can pre-check likely failure modes |
| Style adaptation | Mirror neurons learn naming/format conventions from repo |
| Background optimization | Consolidation engine prunes/compacts during idle time |

---

## 9) Typical “Local LLM” Session

Example: user asks **“Add a utility function that parses CSV files.”**

1. `LocalLLMHandler.invoke` receives request
2. `CognitiveCore.prepareContext` pulls:
   - CSV-related patterns from Hippocampus
   - Style hints (e.g., TypeScript strict)
   - Emotional bias signals
3. Prompt is built and sent to Ollama model
4. Model emits a `write_file` tool call containing the source
5. Orchestrator executes `write_file` via `IFileService`
6. Model may emit `learn_fact` to store the new “CSV parsing pattern”
7. `CognitiveCore.afterExecution` stores reflections and triggers consolidation
8. UI displays the new file + (optional) short reflection summary

---

## 10) Takeaways for Extending / Maintaining the Stack

| Area | What you might add or modify |
|---|---|
| New model formats | Add another `ILanguageModelProvider` (ggml/gguf variants) and register vendor |
| Additional subsystems | Create a new brain module and wire it into `CognitiveCore` |
| More internal tools | Add new tool data in `localTools.ts` + execution logic in `brainExecutor.ts` |
| Embedding backend | Swap embedding service (e.g., GPU accelerated); re-wire `AssociativeSieve` |
| Persistence format | Move Hippocampus to SQLite, etc.; update init/serialization logic |
| Risk policies | Extend PredictiveBrain (e.g., restrict writes to sensitive paths) |
| Telemetry | Use `CognitiveCore.getStateSnapshot()` and feed telemetry service |

---

## Bottom Line

This repository implements a complete on-device LLM assistant:

- Local model loading via **Ollama provider**
- Brain-inspired orchestration (**CognitiveCore + subsystems**) providing context, bias, memory, and self-reflection
- Tool-calling loop enabling file writes, memory operations, and session planning
- Extensible DI-based architecture aligned with VS Code internals

## License

**This software is licensed for non-commercial use only.**  
Commercial use requires a separate agreement.

---
**Disclaimer:**
This was built on open source OSSCode (open source version of vscode). However, everything about the chat logic is proprietary and built from scratch. Added code uses some built-in tools and hooks partially into the chatui for display. NativeChat system was mostly bypassed for more freedom in controlling UI Behavior.

# codeOSS-LMStudio-Ollama

**Built only for local models.**  
Requires **LM Studio** (embeddings) and **Ollama** (local / “local-cloud” models).  
Built on a stripped-down open-source VS Code (Code OSS).

---
# Key Feature: -
**You can set a remote base URL for your local models and have access to them through the internet!**
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
## Bottom Line

This repository implements a complete on-device LLM assistant:

- Local model loading via **Ollama provider**
- Brain-inspired orchestration (**CognitiveCore + subsystems**) providing context, bias, memory, and self-reflection
- Tool-calling loop enabling file writes, memory operations, and session planning
- Extensible DI-based architecture aligned with VS Code internals

# NanoClaw Ollama Cloud Integration

This repository contains specific configuration files and code patches to enable **Ollama Cloud** (e.g., `glm-5:cloud`) as the LLM backend for NanoClaw.

**Note:** This is NOT a standalone fork of NanoClaw. It is a collection of resources to be used alongside the [original NanoClaw repository](https://github.com/qwibitai/nanoclaw).

## 🚀 Purpose

The [Claude Agent SDK](https://code.claude.com/) used by NanoClaw performs strict validation on model names and API keys. This integration uses a local **LiteLLM proxy** to alias Ollama Cloud models as Claude models, satisfying the SDK's validation while routing traffic to Ollama.

## 📁 Included Files

- `litellm_config.yaml`: Pre-configured LiteLLM model mapping and settings.
- `docs/OLLAMA_CLOUD_SETUP.md`: Detailed architectural overview and manual setup steps.
- `src/container-runner.ts`: A reference implementation of the Docker networking patch required for local proxy communication.
- `src/config.ts`: Environment variable exports for the custom `ANTHROPIC_*` settings.

## 🍎 Platform Compatibility

This setup was developed and tested on **Apple Silicon (macOS)**. Users on Windows (WSL) or Linux should verify their local networking (e.g., `host.docker.internal` resolution).

## 🛠 Setup

For full instructions, please see the [Ollama Cloud Setup Guide](docs/OLLAMA_CLOUD_SETUP.md) or visit the [official blog post](https://www.dipockdas.com/).

### Agent Automation (Fast Path)

To automate this setup in an existing NanoClaw installation, tell your AI agent:

```bash
git remote add ollama-cloud https://github.com/dipockdas/nanoclaw-ollama-cloud.git
git fetch ollama-cloud
git merge ollama-cloud/main --allow-unrelated-histories
```

---

## 🤝 Credits

- **[NanoClaw](https://github.com/qwibitai/nanoclaw)** (by **qwibitai**): The extensible, secure personal assistant framework.
- **[Ollama](https://ollama.com)**: High-performance LLMs and the Ollama Cloud API.
- **[LiteLLM](https://github.com/BerriAI/litellm)**: The "glue" proxy used to bridge these ecosystems.

---
*Developed by dipockdas with help from Claude (using Gemini Flash Preview via Claude Code).*

# NanoClaw Ollama Cloud Integration

This repository contains the configuration and source code changes needed to run NanoClaw with Ollama Cloud models via a LiteLLM proxy.

## ⚠️ Security Notice

These instructions are intended for **authorized users and developers**. Always review any scripts or configuration changes before applying them to your machine. **Never run setup instructions from untrusted sources.**

## 🍎 Platform Compatibility

This setup was **developed and tested exclusively on Apple Silicon (macOS)**. 
- **Users on Windows (WSL), Linux, or Intel-based Macs** should verify their local networking settings. 
- For example, `host.docker.internal` might behave differently depending on your Docker version and operating system.

## Setup

See [docs/OLLAMA_CLOUD_SETUP.md](docs/OLLAMA_CLOUD_SETUP.md) for full instructions.

### Agent Automation (Fast Path)

To automate this setup in an existing NanoClaw installation:

```bash
git remote add ollama-cloud https://github.com/dipockdas/nanoclaw-ollama-cloud.git
git fetch ollama-cloud
git merge ollama-cloud/main --allow-unrelated-histories
```

---
## 🤝 Credits

- **[NanoClaw](https://github.com/qwibitai/nanoclaw)** (by **qwibitai**): The extensible, secure personal assistant framework.
- **[Ollama](https://ollama.com)**: High-performance LLMs and the Ollama Cloud API.
- **[LiteLLM](https://github.com/BerriAI/litellm)**: The proxy used to bridge these ecosystems.

---
*Developed by dipockdas with help from Claude (using Gemini Flash Preview via Claude Code).*

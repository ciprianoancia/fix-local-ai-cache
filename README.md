# fix-local-ai-cache

Diagnose and fix slow local LLM and coding agent performance caused by KV cache misses, context overflow compaction, and full prompt reprocessing across any OS, inference engine, or agent client.

## Install

Install with the open-source `skills` CLI (works with Claude Code, Cursor, Codex, OpenCode, Gemini CLI, Cline, and more):

```bash
npx skills add ciprianoancia/fix-local-ai-cache
```

## What It Fixes

- **Claude Code:** Disables dynamic attribution header cache invalidation (`CLAUDE_CODE_ATTRIBUTION_HEADER=0` or updates to `2.1.181`+).
- **Ollama:** Prevents 32K context overflow compaction on 24–64 GB machines by setting `OLLAMA_CONTEXT_LENGTH=65536`, `OLLAMA_FLASH_ATTENTION=1`, and `OLLAMA_KV_CACHE_TYPE=q8_0`.
- **llama.cpp / LM Studio:** Enables verbose cache logging (`forcing full prompt reprocessing`), Flash Attention, `q8_0` KV cache quantization, and single-slot context allocation (`-np 1`).
- **vLLM / MLX:** Verifies prefix caching flags and prompt-cache retention.
- **Agent Frontends (OpenClaw, Hermes, Cline, Continue):** Ensures static system prefixes and preserves `<think>` blocks in multi-turn history.
- **Hybrid / Reasoning Models (e.g., Qwen3.8):** Lowers reasoning effort from `xhigh` to `medium` to avoid context bloat and recurrent checkpoint misses.

## License

MIT
# fix-local-ai-cache

Diagnose and fix slow local LLM and coding agent performance caused by KV cache misses, context overflow compaction, and full prompt reprocessing across any OS, inference engine, or agent client.

## Install

Install with the open-source `skills` CLI (works with Claude Code, Cursor, Codex, OpenCode, Gemini CLI, Cline, and more):

```bash
npx skills add ciprianoancia/fix-local-ai-cache
```

## What It Fixes

- **Claude Code:** Disables dynamic attribution header cache invalidation (`CLAUDE_CODE_ATTRIBUTION_HEADER=0` or updates to `2.1.181`+).
- **Ollama:** Prevents 32K context overflow compaction on 24–64 GB machines by setting `OLLAMA_CONTEXT_LENGTH=65536`, `OLLAMA_FLASH_ATTENTION=1`, and `OLLAMA_KV_CACHE_TYPE=q8_0`.
- **llama.cpp / LM Studio:** Enables verbose cache logging (`forcing full prompt reprocessing`), Flash Attention, `q8_0` KV cache quantization, and single-slot context allocation (`-np 1`).
- **vLLM / MLX:** Verifies prefix caching flags and prompt-cache retention.
- **Agent Frontends (OpenClaw, Hermes, Cline, Continue):** Ensures static system prefixes and preserves `<think>` blocks in multi-turn history.
- **Hybrid / Reasoning Models (e.g., Qwen3.8):** Lowers reasoning effort from `xhigh` to `medium` to avoid context bloat and recurrent checkpoint misses.

## License

MIT

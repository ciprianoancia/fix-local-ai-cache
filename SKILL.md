---
name: fix-local-ai-cache
description: Diagnose and fix slow local LLM and coding agent performance caused by KV cache misses, context overflow compaction, and full prompt reprocessing across any OS, inference engine, or agent client. Use when local AI, Ollama, llama.cpp, LM Studio, vLLM, MLX, or coding agents (Claude Code, OpenClaw, Hermes, Cline, Continue) feel sluggish, stall before generating the first token, repeatedly compact context, or reprocess prompts from scratch across turns.
license: MIT
---

# Fix Local AI Cache (Universal)

Diagnose and resolve slow prompt-reading (prefill) bottlenecks across any local LLM stack (macOS, Linux, Windows; Ollama, llama.cpp, LM Studio, vLLM, MLX; Claude Code, OpenClaw, Hermes, Cline, Continue, or custom clients). High tokens-per-second counters only measure output generation (decode), whereas multi-turn agents stall when prefix caching breaks and forces full prompt reprocessing on every turn.

## Core Principles

1. **Detect before changing:** Inspect the active **client/agent**, **inference backend**, and **model architecture** first. Apply only the fixes that match the detected stack.
2. **Preserve prefix immutability:** KV prefix caching requires the beginning of the prompt (system prompt, tool schemas, prior turns, and reasoning blocks) to stay byte-for-byte identical across turns.
3. **Prevent context overflow compaction:** When an agent's initial prompt exceeds the server's active context window, the client summarizes/compacts history into a brand-new prompt, destroying the cache on every cycle.
4. **Fit the KV cache in VRAM:** Enlarging context windows to 64K+ increases KV cache RAM/VRAM usage linearly; if the cache spills to system RAM/swap, prefill and decode collapse.

## Step 1: Quick Triage (Detect Stack & Bottleneck)

Run a quick check to identify which client and server are active and whether cache misses are occurring:

- **Check active processes & ports:**

```bash
ps aux | grep -E "ollama|llama-server|lms|vllm|mlx"
```

- **Check Ollama loaded models (if installed):**

```bash
ollama ps
```

- **Check Claude Code version & config (if installed):**

```bash
claude --version && cat ~/.claude/settings.json 2>/dev/null
```

Use the matrix below to jump directly to the rules matching the user's stack:

| Component Detected | Apply Rule | Primary Fix |
| :--- | :--- | :--- |
| **Claude Code** | Rule 1 | Update to `2.1.181`+ or set `"CLAUDE_CODE_ATTRIBUTION_HEADER": "0"` in `~/.claude/settings.json` |
| **Ollama** | Rule 2 | Set `OLLAMA_CONTEXT_LENGTH=65536` + `OLLAMA_KV_CACHE_TYPE=q8_0` + `OLLAMA_FLASH_ATTENTION=1` |
| **llama.cpp / LM Studio** | Rule 3 | Enable `--verbose` / Developer Logs to inspect `forcing full prompt reprocessing`; tune `-fa on` & cache quant |
| **vLLM / MLX** | Rule 4 | Ensure `--enable-prefix-caching` (vLLM) or prompt-cache reuse (MLX) is active and slot count fits VRAM |
| **OpenClaw / Hermes / Cline / Continue** | Rule 5 | Disable dynamic system prompt timestamps/injections and preserve `<think>` blocks in history |
| **Hybrid / Reasoning Models (e.g., Qwen3.8)** | Rule 6 | Drop reasoning effort from `xhigh` to `medium` and avoid mid-history message edits |

## Step 2: Stack-Specific Rules (If Using X -> Fix Y)

### Rule 1: If Using Claude Code

- **Issue:** Versions prior to `2.1.181` prepend a dynamic attribution header to the system prompt on every request, invalidating the KV cache prefix on every turn. Additionally, Claude Code's default system prompt + tool definitions consume ~33,000 tokens on turn 1.
- **Check:** Run `claude --version` and inspect `~/.claude/settings.json`.
- **Fix:**
  1. Update Claude Code to `2.1.181` or newer.
  2. For older builds or guaranteed persistence across GUI/IDE launches, add `"CLAUDE_CODE_ATTRIBUTION_HEADER": "0"` inside the `"env"` block of `~/.claude/settings.json` (in addition to `export CLAUDE_CODE_ATTRIBUTION_HEADER=0` in your shell profile):

```json
{
  "env": {
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
  }
}
```

  3. Optional local latency reduction: launch with `--exclude-dynamic-system-prompt-sections` (or `--bare` when full Claude Code scaffolding is unnecessary) to keep the system prompt prefix compact and static.
- **Verification:** Confirm `claude --version` is `2.1.181`+ or `~/.claude/settings.json` contains `"CLAUDE_CODE_ATTRIBUTION_HEADER": "0"`.

### Rule 2: If Using Ollama

- **Issue:** Ollama sizes default context windows by visible GPU memory (4K below 24 GB, 32K for 24-48 GB, 256K above 48 GB). On 48 GB unified-memory Macs (~36 GB visible to GPU) or 24-48 GB GPUs, the 32K default overflows immediately on ~33K-token coding agent prompts, causing endless context compaction and cold reads.
- **Check:** Run `ollama ps` while a model is loaded and inspect the `CONTEXT` column.
- **Fix:**
  1. Set `OLLAMA_CONTEXT_LENGTH=65536` (or higher), enable Flash Attention, and quantize the KV cache to `q8_0` (cuts KV cache VRAM roughly in half with negligible quality loss so 64K+ fits comfortably in VRAM):
     - **macOS (`launchd`):**

```bash
launchctl setenv OLLAMA_CONTEXT_LENGTH 65536
launchctl setenv OLLAMA_FLASH_ATTENTION 1
launchctl setenv OLLAMA_KV_CACHE_TYPE q8_0
```

     - **Linux (`systemd`):** Add `Environment="OLLAMA_CONTEXT_LENGTH=65536"`, `Environment="OLLAMA_FLASH_ATTENTION=1"`, and `Environment="OLLAMA_KV_CACHE_TYPE=q8_0"` under `[Service]` in `/etc/systemd/system/ollama.service.d/override.conf`, then run `sudo systemctl daemon-reload && sudo systemctl restart ollama`.
     - **Windows:** Add `OLLAMA_CONTEXT_LENGTH=65536`, `OLLAMA_FLASH_ATTENTION=1`, and `OLLAMA_KV_CACHE_TYPE=q8_0` in User/System Environment Variables and restart Ollama from the system tray.
  2. Alternatively, bake `PARAMETER num_ctx 65536` into a custom Modelfile if you only want a large context on specific coding models.
- **Verification:** Restart Ollama, send a prompt, and run `ollama ps` to confirm `CONTEXT` displays `65536` or higher and `PROCESSOR` shows `100% GPU`.

### Rule 3: If Using llama.cpp (`llama-server`) or LM Studio

- **Issue:** Default log levels hide KV cache misses, making full prompt reprocessing invisible while prefill stalls for minutes. Multiple parallel slots (`-np`) also divide the available context window across slots.
- **Check:** Enable verbose/debug logging and inspect server logs on turn 2+ of a session:
  - **llama.cpp:** Start `llama-server` with `--verbose` (or log level 4).
  - **LM Studio:** Open **Developer / Server Logs** and set log verbosity to verbose/debug.
- **Fix:**
  1. Search logs for `forcing full prompt reprocessing due to lack of cache data` (or `0` cached prompt tokens).
  2. For `llama-server`, enable Flash Attention (`-fa on`), KV cache quantization (`--cache-type-k q8_0 --cache-type-v q8_0`), and keep parallel slots at `-np 1` for single-user agents so the full context budget (`-c 65536`) stays dedicated to one active prefix:

```bash
llama-server -m model.gguf -c 65536 -np 1 -fa on --cache-type-k q8_0 --cache-type-v q8_0 --verbose
```

  3. In **LM Studio**, enable **Flash Attention** and **K/V Cache Quantization** (`q8_0`) in model load settings, and verify context length is set to `65536`+.
- **Verification:** Follow-up turns log prompt cache reuse (`prompt eval time` processes only new tokens) rather than `forcing full prompt reprocessing`.

### Rule 4: If Using vLLM or MLX (`mlx_lm.server`)

- **Issue:** Prefix caching may be disabled, evicted prematurely due to memory pressure, or bypassed by oversized concurrent batch settings.
- **Fix:**
  - **vLLM:** Ensure `--enable-prefix-caching` is active, set `--max-model-len 65536` (or matching your agent's needs), and use `--kv-cache-dtype fp8` if VRAM is tight.
  - **MLX (`mlx_lm.server`):** Ensure the client preserves exact message prefixes and avoid restarting or switching models mid-session, which clears the in-memory prompt cache.
- **Verification:** Server metrics/logs show high prefix cache hit rates (`cached_tokens` > 0) on turn 2+.

### Rule 5: If Using Other Agent Frontends (OpenClaw, Hermes, Cline, Continue, Custom Scripts)

- **Issue:** Some clients mutate system prompts, inject live per-turn timestamps/git status into the system prefix, reorder tool definitions, or strip assistant `<think>` blocks between turns, forcing `0` cached tokens even when the server is properly configured.
- **Check:** Inspect per-turn server API usage fields (`cached_tokens`, `cache_read_input_tokens`, or prompt eval token count) on turns 2 and 3.
- **Fix:**
  - Disable dynamic system-prompt injections (such as per-turn timestamps or changing metadata at the top of the prompt).
  - For reasoning models, configure the client to **preserve `<think>` / reasoning blocks** in assistant conversation history rather than stripping them between turns.
  - Keep tool schemas in a deterministic, fixed order across requests.
- **Verification:** On turn 2+, server metrics report `cached_tokens` approximately equal to the prior turn's total prompt tokens.

### Rule 6: If Using Hybrid or Reasoning Models (e.g., Qwen3.8, Linear-Attention Hybrids)

- **Issue:** Hybrid architectures combine standard attention layers with recurrent/linear summary layers that cannot rewind to arbitrary positions—they only save periodic cache checkpoints thousands of tokens apart. Maximum reasoning effort (`xhigh`) generates 5x-6x more thinking tokens, bloating context and causing checkpoint misses.
- **Check:** Inspect model reasoning effort settings and per-turn thinking token counts.
- **Fix:**
  - Lower reasoning/thinking effort from `xhigh` (or max) to `medium`.
  - Avoid editing, deleting, or rewriting mid-conversation messages so the prompt prefix stays aligned with saved checkpoints.
- **Verification:** Thinking token counts drop to moderate levels and follow-up turns prefill in seconds instead of minutes.

## Step 3: Restart & End-to-End Verification

1. **Restart affected processes:** Environment variables (`OLLAMA_CONTEXT_LENGTH`, `CLAUDE_CODE_ATTRIBUTION_HEADER`) only take effect after restarting the Ollama daemon/app and opening a fresh terminal session.
2. **Run a 2-turn cache test:**
   - **Turn 1 (Cold Read):** Send an initial agent prompt and note the time-to-first-token (TTFT).
   - **Turn 2 (Warm Read):** Send a short follow-up (`"Reply with ok"`). Confirm TTFT drops to a few seconds and server logs/usage show `cached_tokens` matching Turn 1's prompt length.

## Gotchas

- **Decode TPS is deceptive:** A UI showing 50+ tokens/sec only measures generation speed; if prefill re-reads 40K tokens every turn, effective agent speed drops below 3 tokens/sec.
- **Shell exports vs. GUI/daemon processes:** `export VAR=...` in `~/.zshrc` does not apply to background daemons (`launchd`/`systemd`) or GUI apps unless set via `launchctl setenv`, `systemd` unit overrides, or `~/.claude/settings.json`.
- **Context size vs. VRAM spillover:** Raising `num_ctx` to 65536+ without `q8_0` KV cache quantization can push a borderline model onto CPU/swap—always check `ollama ps` (`100% GPU`) or VRAM usage after raising context length.

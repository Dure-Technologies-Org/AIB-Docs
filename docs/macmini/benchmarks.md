# Benchmarks

Benchmarking AI models in standalone and in-app (ie. embedded in intellicare) modes.

## Standalone


### Whisperlivekit 

For a ~1min live-audio:  

```bash
wlk --backend mlx-whisper --model large-v3-turbo --model_cache_dir /root/.cache/huggingface/hub
```

| Backend | Model | Lang | CPU% | unified mem | Lag |
|---|---|---|---|---|---|
| faster-whisper(cpu) | large-v3-turbo | En | ~40% | 3.3 | ~3 min |
| mlx-whisper(gpu) | large-v3-turbo | En | ~2% | 1.8GB | ~1 sec |

* all experiments were in `float16` datatype using `wlk` cli. 
* Even though `wlk` process took `~1.8GB` of unified memory, it showed an unexplained jump of `~12GB` on `btop`. 

### MedASR

* Tested [medasr-mlx](https://github.com/ainergiz/medasr-mlx), [medasr-patient-summary](https://github.com/ainergiz/medasr-patient-summary), [medasr colab](https://github.com/google-health/medasr/blob/main/notebooks/quick_start_with_hugging_face.ipynb)
* Used UAT recordings from hospital and local team's voices.
* Fails to transcribe in foreign languages other than English. 
* French accent english recording tranascriptions were on par with whisperlivekit.
* MedASR is trained with lot of clinical jargon words, maybe not in practice with day-to-day doctor-patient interactions.

### vllm-metal vs llama.cpp

* MedGemma-4B-it & Qwen3.5-4B serving backend evaluation with 16K context

#### Backend comparison (discharge_clinical_triage & other scenarios)

| Scenario | Engine (quant) | TTFT p50 | Total p50 | Decode tok/s | out tok p50 | Result |
|---|---|---|---|---|---|---|
| discharge_clinical_triage | vLLM-metal (MLX 4-bit) | 147ms | 120,183ms | 34 | 4,096 | ❌ garbage loop (repetition_score ~1.0) |
| discharge_clinical_triage | vLLM-metal (MLX 8-bit) | — | 151,900ms | ~27 | 4,096 | ❌ garbage loop (same failure, precision didn't help) |
| discharge_clinical_triage | **llama.cpp (MedGemma GGUF Q4_K_M)** | **25ms** | **13,436ms** | **69** | 923 | ✅ valid, natural stop |
| discharge_clinical_triage | **llama.cpp (Qwen3.5-4B GGUF, thinking off)** | 69ms | 30,746ms | 50 | 1,537 | ✅ valid, natural stop |
| realtime_visit_summary | vLLM-metal (4-bit) | 122ms | 22,010ms | 37 | 800 | ❌ garbage loop |
| realtime_visit_summary | **llama.cpp (MedGemma)** | **23ms** | **2,223ms** | **70** | 154 | ✅ coherent |
| realtime_visit_summary | **llama.cpp (Qwen3.5-4B, thinking off)** | 3,479ms | 6,888ms | 51 | 149 | ✅ coherent |
| chatbot_qa | vLLM-metal (4-bit) | 102ms | 437ms | 42 | 14 | ✅ ok (short prompt) |
| chatbot_qa | **llama.cpp (MedGemma)** | 60ms | **241ms** | **78** | 14 | ✅ ok |
| chatbot_qa | **llama.cpp (Qwen3.5-4B, thinking off)** | 65ms | 1,252ms | 52 | 62 | ✅ ok |
| opd_extraction | vLLM-metal (4-bit) | 276ms | 31,956ms | 32 | 1,024 | ❌ garbage loop |
| opd_extraction | **llama.cpp (MedGemma)** | **38ms** | **15,447ms** | **66** | 1,024 | ⚠️ valid JSON, cut mid-object (undersized max_tokens) |
| opd_extraction | **llama.cpp (Qwen3.5-4B, thinking off)** | 16,830ms | 38,333ms | 48 | 1,024 | ⚠️ valid JSON, cut mid-object (same cause) |
| config_extraction_medium | vLLM-metal (4-bit) | 276ms | 39,040ms | 31 | 1,200 | ❌ garbage loop |
| config_extraction_medium | **llama.cpp (MedGemma)** | **38ms** | **18,167ms** | **66** | 1,200 | ⚠️ valid JSON, cut mid-object |
| config_extraction_medium | **llama.cpp (Qwen3.5-4B, thinking off)** | 11,868ms | 27,874ms | 48 | 1,200 | ⚠️ valid JSON, cut mid-object |
| config_extraction_stress | **llama.cpp (Qwen3.5-4B)** | — | — | — | — | ❌ 400 error: real DHIS2 schema is 23-24K tokens, exceeds 16K context (model-independent) |

#### Root cause
- vLLM-metal (both 4-bit and 8-bit MLX quantization of `mlx-community/medgemma-4b-it-*`) produces deterministic repetition-loop garbage on any prompt above ~800 input tokens, across 6 of 7 real production scenarios. Confirmed independent of `temperature` (0 and 0.2) and independent of JSON-mode grammar constraints.
- vllm-metal's own docs list Gemma 3 as supported, but the only validated example checkpoint is a **QAT** 4-bit build — MedGemma has no QAT variant on Hugging Face (`mlx-community` only ships plain `-4bit/-6bit/-8bit/-bf16`). Untested fine-tune × untested quantization method.
- **llama.cpp with GGUF `Q4_K_M` fully resolves the degeneration** on the identical prompts, and is ~2x faster on decode with far lower TTFT.

#### Remaining issue: opd_extraction / config_extraction_medium truncation (llama.cpp)
- Both scenarios still hit `max_tokens` on llama.cpp — but this is **legitimate truncation**, not degeneration: the cut-off content is valid, well-formed JSON, and the field names (`symptom_blurred_vision`, etc.) are real fields from `opd_form_schema.yaml` / `models.py` — the model is genuinely filling the production schema. Same behavior on both MedGemma and Qwen3.5-4B, confirming it's a prompt/budget issue, not a model quirk.
- Tested unbounded (`max_tokens=8000`, MedGemma): **the model still does not terminate naturally** — it hits length cutoff at 7,347 tokens having emitted 111/104 field-objects for a 90-field schema (i.e., re-extracting duplicates). No clean "full" output length exists to size `max_tokens` to.
- **Bumping max_tokens is not a safe fix on its own**:
  - Production Ollama/Jetson timeout is hardcoded to 120s (`llm_extractor.py:240`), sized for the current 1024-token default at Jetson's ~12 tok/s. A larger budget (thousands of tokens) would exceed 120s by 5x+.
  - On timeout, `_call_ollama` silently returns `"{}"` (empty result) rather than erroring — so on real hardware this would silently zero out extractions, not just truncate them.
  - `extract_json_from_text()` has no partial-JSON recovery — truncation at any token count is a hard parse failure today.
  - `extract_fields` is invoked from a WebSocket handler (`transcription_ws.py`) — live ambient extraction during consultation; longer generations risk stale/lagging auto-filled fields.
- **Recommendation**: fix via prompt design (explicit stop-after-N-fields instruction, or split the 90-field schema into smaller per-section extraction calls) rather than a `max_tokens` increase, and if `max_tokens` is raised at all, pair it with a proportional timeout increase and re-test on Jetson-class hardware (not just this Mac).

#### Qwen3.5-4B-GGUF (llama.cpp) — reasoning-mode requirement
- Qwen3.5-4B is a thinking/reasoning model. With default settings, it spends the entire `max_tokens` budget on invisible `reasoning_content` and returns **empty** `content` on 6 of 8 scenarios (TTFT up to 75s, total latency up to 98s).
- Fix: pass `chat_template_kwargs: {"enable_thinking": false}` per-request (no server restart needed — confirmed via llama.cpp's `-rea`/`--reasoning` flag exists as a startup-time alternative, but the per-request override takes precedence). All numbers above are with thinking disabled.
- Decode speed is flat at ~48-52 tok/s regardless of scenario or thinking mode — thinking doesn't change generation speed, only how many tokens get burned (and how long TTFT is) before anything usable comes back. Notably slower raw decode than MedGemma (66-78 tok/s) on identical hardware/backend.
- `config_extraction_stress` fails independent of model/thinking mode — the real production DHIS2 config schema is 23-24K tokens, exceeding the 16K context window outright.

# Benchmarks

Benchmarking AI models in standalone and in-app (ie. embedded in intellicare) modes.

## Standalone

### MedGemma (Ollama Native)

```bash
jetson@yahboom:~$ ollama run --verbose medgemma:4b-it-q4_K_M
>>> hi
Hello! How can I help you today?


total duration:       1.642398878s
load duration:        290.279712ms
prompt eval count:    9 token(s)
prompt eval duration: 543.988279ms
prompt eval rate:     16.54 tokens/s
eval count:           11 token(s)
eval duration:        805.393013ms
eval rate:            13.66 tokens/s
```

### MedGemma (Ollama Docker)

```bash
jetson@yahboom:~$ docker run --runtime nvidia -it --rm --network host -v ~/.ollama:/.ollama  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility -e NVIDIA_VISIBLE_DEVICES=all dustynv/ollama:r36.2.0
root@yahboom:/# ollama run --verbose medgemma:4b-it-q4_K_M
>>> hi
Hi there! How can I help you today?


total duration:       1.27903489s
load duration:        284.528683ms
prompt eval count:    9 token(s)
prompt eval duration: 127.23802ms
prompt eval rate:     70.73 tokens/s
eval count:           12 token(s)
eval duration:        843.226315ms
eval rate:            14.23 tokens/s
```
NOTE: ollama is not concerned with graphics driver files by nvidia and so `-e NVIDIA_DRIVER_CAPABILITIES=compute,utility` might be needed, or else `--runtime nvidia` will give graphics driver files to ollama too.


```bash
jetson@yahboom:~$ docker run --runtime nvidia -it --rm --network host -v ~/.ollama:/.ollama -e NVIDIA_DRIVER_CAPABILITIES=compute,utility -e NVIDIA_VISIBLE_DEVICES=all dustynv/ollama:r36.4.3
root@yahboom:/# ollama run --verbose medgemma:4b-it-q4_K_M
>>> hi
Hi there! How can I help you today?


total duration:       1.430539752s
load duration:        262.177672ms
prompt eval count:    9 token(s)
prompt eval duration: 419.962748ms
prompt eval rate:     21.43 tokens/s
eval count:           12 token(s)
eval duration:        746.435849ms
eval rate:            16.08 tokens/s
```

```bash
jetson@yahboom:~$ docker run --runtime nvidia -it --rm --network host -v ~/.ollama:/.ollama  ghcr.io/nvidia-ai-iot/ollama:r38.2.arm64-sbsa-cu130-24.04
root@yahboom:/# ollama run --verbose medgemma:4b-it-q4_K_M
>>> hi
Hi there! How can I help you today?


total duration:       1.425674348s
load duration:        200.226633ms
prompt eval count:    9 token(s)
prompt eval duration: 464.847896ms
prompt eval rate:     19.36 tokens/s
eval count:           12 token(s)
eval duration:        759.126239ms
eval rate:            15.81 tokens/s
```

### Whisperlivekit 

For a ~1min live-audio:  

```bash
docker run --gpus all \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility \
  -e HF_HUB_OFFLINE=1 \
  -v /idata/.cache/huggingface:/root/.cache/huggingface \
  -v /idata/.cache/wlk:/root/.cache/wlk \
  -p 8000:8000 \
  --name wlk -d wlk \
  --backend whisper --model large-v3-turbo --model_cache_dir /root/.cache/wlk
```

| Backend | Model | Lang | CPU% | CPU mem | GPU mem | Lag |
|---|---|---|---|---|---|---|
| whisper | large-v3-turbo | En | ~50% | 500MB | 5.3GB | ~10 sec |
| whisper | tiny | En | ~50% | 450MB | 1GB | ~0.5 sec |
| faster-whisper(cpu) | large-v3-turbo | En | ~200% | 800MB | 1.5GB | ~4 min |
| faster-whisper(cpu) | tiny | En | ~150% | 350MB | 1.3GB | ~2 sec |
| faster-whisper(gpu) | large-v3-turbo | En | ~15% | 800MB | 2GB | ~10 sec |
| faster-whisper(gpu) | tiny | En | ~15% | 350MB | 350MB | ~0.5 sec |

* `Faster-whisper` uses `ctranslate2` which does not have gpu version compatible with jetson   
ie. aarch64, py3.10, so it falls back to cpu for `ctranslate2` tasks. Unless we build `ctranslate2` from source for jetson.
* if `backend==whisper`: `--model_cache_dir /root/.cache/wlk`,  
if `backend==faster-whisper`: `--model_cache_dir /root/.cache/huggingface/hub`
* `tiny` model, even if with least lag, barely transcribes correctly.
* Cannot change `faster-whisper` `compute-type` to `int8` or `int8_float16`, with `wlk` cli, but can be set via wlk API.


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

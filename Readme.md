# ASR Benchmarking: Voxtral vs Faster Whisper

## 📌 Overview
This project benchmarks **Automatic Speech Recognition (ASR)** systems across **streaming and offline modes** using:

- Voxtral Mini 4B Realtime (Streaming + Offline)
- Faster Whisper (large-v3-turbo) (Streaming + Offline baseline)

# Metrics

### Latency and Time To First TokeTn (TTFT)
- **Streaming:**  
  UPL = Time from audio end - Final transcription available
  TTFT = First token emitted - Time from first chunk sent 
- **Offline:**  
  UPL(Total Latency) = Full Transcription Available − Start time of model after sending full audio 
  TTFT = First Token emmited - Full audio sent 

---

# ⚙️ Models

## Voxtral Mini 4B
- Native streaming model
- Token-level decoding
- Supports real-time inference

## Faster Whisper (large-v3-turbo)
- Optimized Whisper inference
- No true token streaming (segment-based output)
- Used as strong baseline

---

# Methodology

## Hardware
- GPU: NVIDIA A100
- Precision: FP16 / BF16

## Dataset
- 24 Hinglish audio clips
- 8 clips per Bucket
- Source: ai4Bharat/IndicVoices
- Conversational/Extempore Hinglish audio clips
- Buckets:
  - 5s
  - 15s
  - 30s

## Inference Setup
- 3 runs per clip for averaging
- Warmup runs are discarded
- Same dataset across all models

## WER Calculation
- Word Error Rate (WER) was evaluated using the `jiwer` library.
- Both reference and hypothesis transcripts were normalized (lowercasing, punctuation removal, and space normalization) before computing WER.

## Model Loading
- Voxtral for Realtime Streaming was Inferenced using vLLM engine for various delay settings.
- Voxtral and Wisper for Offline Transcrption was Locally loaded on A100 GPU and then Inferenced.

---

# 📊 Results

## 1. Voxtral Realtime Streaming Results


| Bucket | Delay (ms) | p50 TTFT (ms) | p95 TTFT (ms) | p50 UPL (ms) | p95 UPL (ms) | p50 WER | p95 WER |
|--------|------------|---------------|---------------|--------------|--------------|---------|---------|
| 5s     | 480        | 509.31        | 510.16        | 272.36       | 276.58       | 0.2385  | 0.335   |
|        | 960        | 27.17         | 28.94         | 279.58       | 285.88       | 0.2385  | 0.335   |
|        | 2400       | 27.50         | 28.31         | 278.49       | 282.12       | 0.2385  | 0.335   |
| 15s    | 480        | 513.10        | 514.48        | 285.64       | 293.38       | 0.2165  | 0.330   |
|        | 960        | 31.06         | 32.22         | 280.94       | 292.05       | 0.2165  | 0.330   |
|        | 2400       | 30.36         | 34.19         | 280.66       | 296.17       | 0.2165  | 0.330   |
| 30s    | 480        | 515.93        | 516.74        | 284.08       | 292.84       | 0.2230  | 0.376   |
|        | 960        | 33.73         | 34.92         | 287.54       | 297.37       | 0.2230  | 0.376   |
|        | 2400       | 32.39         | 35.38         | 272.32       | 277.17       | 0.2230  | 0.376   |

###  Observations
- **480 ms delay** → Higher observed TTFT (~500 ms) occurs because the model receives the first chunk at *t = 0*, but lacks sufficient context to generate a token, so it waits for the next chunk.

- In real-world streaming, the **effective TTFT** should include the chunk delay:
  - For **960 ms / 2400 ms delay** →  
    `TTFT ≈ delay time + model TTFT`
  - For **480 ms delay** →  
    `TTFT ≈ 480 ms + model TTFT`

- Therefore, despite the higher measured TTFT, **480 ms delay actually results in lower real-world TTFT latency** compared to 960 ms and 2400 ms, since the initial buffering time is smaller.
- In real world we don,t have initial chunk at t=0.
-**WINNER** = 480ms

- **WER Comparision** -> Wer is approximately same for all three delay settings
- **WINNER** = All three same

- **UPL** -> For 30s clips , 2400ms is giving low UPL but for short clips 480ms is good
-**WINNER** = 2400ms(Long Clips) and 480ms(Short Clips)

## 2. Offline Comparison (Voxtral vs Faster Whisper)

| Model   | Bucket | p50 TTFT (s) | p95 TTFT (s) | p50 UPL (s) | p95 UPL (s) | p50 WER | p95 WER | 
|---------|--------|-------------|-------------|-------------|-------------|---------|---------|
| Voxtral | 5s     | 0.0138      | 0.0141      | 6.054       | 7.575       | 0.250   | 0.311   | 
| Whisper | 5s     | 1.023       | 1.157       | 1.023       | 1.157       | 0.275   | 1.065   |
------------------------------------------------------------------------------------------------ 
| Voxtral | 15s    | 0.0142      | 0.0151      | 14.879      | 19.059      | 0.216   | 0.764   | 
| Whisper | 15s    | 1.562       | 4.857       | 1.562       | 4.859       | 0.359   | 0.825   | 
-----------------------------------------------------------------------------------------------
| Voxtral | 30s    | 0.0138      | 0.0148      | 21.226      | 22.453      | 0.223   | 0.373   | 
| Whisper | 30s    | 5.120       | 5.826       | 5.714       | 6.585       | 0.389   | 1.000   |


### Key Observation
- Whisper is generally **faster offline**(Lower UPL Value)
- TTFT is excluded from comparision since it represents **first segment emission**, not true token latency for Wisper. Wisper architecture doesnt stream tokens one by one.
- WER is better for Voxtral

-**WINNER** -> Voxtral(In terms of Accuracy)
               Wisper(In terms of Latency)




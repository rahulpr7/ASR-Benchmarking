# ASR Benchmarking: Voxtral vs Faster Whisper

## 📌 Overview
This project benchmarks **Automatic Speech Recognition (ASR)** systems across **streaming and offline modes** using:

- Voxtral Mini 4B Realtime (Streaming + Offline)
- Faster Whisper (large-v3-turbo) (Offline baseline)

# 📈 Metrics


## 1. User Perceived Lag(UPL)
## 2. Time To First Token(TTFT)
- **Streaming:**  
-  UPL = Final transcription available - Time from audio end(Stop sending Chunk)
-  TTFT = First token emitted - Time from first chunk sent 
- **Offline:**  
-  UPL = Full Transcription Available − Start time of model after sending full audio 
-  TTFT = First Token emmited - Full audio sent 

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

# 📝 Methodology

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
- Word Error Rate (WER) was evaluated using the standard WER formula.
- Both reference and hypothesis transcripts were normalized (lowercasing, punctuation removal, and space normalization) before computing WER.
### The Formula
The error rate is calculated using the following equation:

$$WER = \frac{S + D + I}{N}$$

| Variable | Name | Description |
| :--- | :--- | :--- |
| **S** | **Substitutions** | Words that were replaced (e.g., "cat" instead of "can"). |
| **D** | **Deletions** | Words that were omitted from the transcript. |
| **I** | **Insertions** | Extra words added that were not in the reference. |
| **N** | **Reference Words** | The total count of words in the original ground-truth text. |

## Model Loading
- Voxtral for Realtime Streaming was Inferenced using vLLM engine for various delay settings.
- Voxtral and Wisper for Offline Transcrption was Locally loaded on A100 GPU and then Inferenced.

---

# 📊 Results

## Voxtral Realtime Streaming Results


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

# 📊 Sreaming Performance Observations: Delay Configuration Analysis

This section outlines the performance characteristics across different delay configurations (480ms, 960ms, and 2400ms) based on Time To First Token (TTFT), Word Error Rate (WER), and UPL metrics.

## ⏱️ Time To First Token (TTFT)

- **Measured vs. Real-World:** A **480ms delay** shows a higher *measured* model TTFT (~500ms) in testing. This happens because the model receives the first chunk at `t = 0` but waits for the next chunk to gain sufficient context before generating a token.
- **Real-World Streaming Impact:** In real-world scenarios, the initial chunk is not instantly available at `t = 0`. Therefore, the effective TTFT must include the chunk buffering delay:
  - **For 960ms / 2400ms delay:** `Effective TTFT ≈ delay time + model TTFT`
  - **For 480ms delay:** `Effective TTFT ≈ 480ms + model TTFT`
- **Conclusion:** Despite the higher *measured* model TTFT, a **480ms delay** actually results in the **lowest real-world TTFT latency** because the initial buffering time is significantly smaller.

> 🏆 **Winner:** `480ms`

---

## 🎯 Word Error Rate (WER)

- **Comparison:** The Word Error Rate remains stable and is approximately the same across all three delay settings. Changing the chunk delay within this range does not significantly impact overall accuracy.

> 🏆 **Winner:** `Tie` (All three settings perform equally)

---

## 📉 UPL Performance

- **Duration Dependency:** The optimal delay setting for UPL depends on the length of the processed clip.
  - **Long Clips (~30s):** `2400ms` provides a lower UPL.
  - **Short Clips:** `480ms` yields better UPL results.

> 🏆 **Winner:** `2400ms` (Long Clips) | `480ms` (Short Clips)


## Offline Transcription Comparison: Voxtral vs. Faster Whisper

The following table details the offline performance comparison between Voxtral and Faster Whisper across 5s, 15s, and 30s audio buckets.

| Model | Bucket | p50 TTFT (s) | p95 TTFT (s) | p50 UPL (s) | p95 UPL (s) | p50 WER | p95 WER |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Voxtral** | 5s | 0.0138 | 0.0141 | 6.054 | 7.575 | 0.250 | 0.311 |
| **Whisper** | 5s | 1.023 | 1.157 | 1.023 | 1.157 | 0.275 | 1.065 |
| **Voxtral** | 15s | 0.0142 | 0.0151 | 14.879 | 19.059 | 0.216 | 0.764 |
| **Whisper** | 15s | 1.562 | 4.857 | 1.562 | 4.859 | 0.359 | 0.825 |
| **Voxtral** | 30s | 0.0138 | 0.0148 | 21.226 | 22.453 | 0.223 | 0.373 |
| **Whisper** | 30s | 5.120 | 5.826 | 5.714 | 6.585 | 0.389 | 1.000 |

### 🔍 Key Observations

* **Latency Advantage (UPL):** Faster Whisper is generally much faster for offline processing, consistently showing significantly lower UPL (User Perceived Latency) values across all tested audio lengths.
* **Accuracy Advantage (WER):** Voxtral significantly outperforms Whisper in accuracy. It maintains a lower and more stable Word Error Rate (WER), avoiding the high p95 WER spikes observed in Whisper.
* **The TTFT Caveat:** TTFT (Time To First Token) should be excluded from direct comparison here. Faster Whisper's architecture does not stream tokens one by one; instead, it emits full segments. Therefore, its recorded TTFT represents the *first segment emission* rather than a true token latency measurement.

### 🏆 Final Verdict

> 🎯 **Winner for Accuracy:** `Voxtral` (Consistently lower WER)  
> ⚡ **Winner for Latency:** `Faster Whisper` (Significantly lower UPL)

## 🚀 Future Roadmap: Optimizing for Banking Voice Pipelines

The core challenge for a banking-grade ASR pipeline is the **WER-Latency Trade-off**. In financial services, accuracy is non-negotiable for compliance, yet high latency leads to "caller overlap" and poor UX. 

To bridge the gap between Voxtral's accuracy and Whisper's latency, the following research steps are proposed:

### 1. Hinglish-Specific Pretraining and Fine-Tuning (Top-Layer Adaptation)
Current observations suggest Voxtral struggles with nuances in Hinglish audio. We plan to:
* **Language-Adaptive Pretraining:** Briefly pretrain the encoder on a larger Hinglish corpus before fine-tuning top layers. It helps with natural code-switching.
* **Layer-Wise Fine-Tuning:** Freeze the base LLM/Encoder and fine-tune only the adapter or top-level projection layers using a curated Hinglish dataset (e.g., Microsoft’s Speechocean or one i used here).
* **Domain-Specific Vocabulary:** Inject banking-specific terminology (e.g., "OTP," "CVV," "Beneficiary," "UPI") into the training corpus to reduce WER on critical keywords.
* **Language-Adaptive Pretraining:** Briefly pretrain the encoder on a larger Hinglish corpus before fine-tuning top layers. It helps with natural code-switching.

### 2. Model-Level Improvements

* **Adaptive Chunking:** Instead of fixed chunks, dynamically adjust based on speech rate or detected pauses. If Slower speech then we should take larger chunk which will result in better context and lower WER.
* **Confidence-Based Trigger:** Only output ASR if token confidence > threshold; otherwise wait a few more ms for context. This slightly increases latency but reduces errors on sensitive terms.

### 3. RAG-Augmented Correction
* Implement a post-processing **RAG (Retrieval-Augmented Generation)** layer. If the ASR output is ambiguous, the system can query a banking-term vector database to "auto-correct" the transcription before it hits the downstream NLU.



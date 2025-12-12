
# Technical Documentation: Web-Based Custom Paraphrase Generator (CPG)

## 1. Executive Summary
[cite_start]This project implements a **Custom Paraphrase Generator (CPG)** entirely within the browser, adhering to the assignment requirements[cite: 6, 7]. It utilizes **Transformers.js** to run a quantized T5-based model client-side via WebAssembly/WebGPU, eliminating the need for a Python backend.

[cite_start]The system is designed to preserve document structure (headers, dates, salutations) while paraphrasing substantial body paragraphs, and it includes a built-in benchmarking tool to compare performance against external LLMs[cite: 8].

---

## 2. System Architecture

The application follows a **Client-Side SPA (Single Page Application)** architecture with a decoupled compute layer.

### Core Technologies
* **Runtime**: Standard Web Browser (Chrome/Edge/Firefox).
* **ML Engine**: [Transformers.js](https://huggingface.co/docs/transformers.js/) (v2.16.0).
* **Model**: `Xenova/LaMini-Flan-T5-248M`.
* **Concurrency**: HTML5 Web Workers (for non-blocking inference).
* **UI Framework**: Tailwind CSS (via CDN).

### High-Level Data Flow
1.  **User Input** -> Main UI Thread.
2.  [cite_start]**Validation** -> Checks word count constraints (200-400 words)[cite: 6].
3.  **Dispatch** -> Text is sent to the **Web Worker** via `postMessage`.
4.  **Inference** -> Web Worker downloads model (first run) or uses cache, then processes text.
5.  **Response** -> Worker streams partial results back to Main Thread for real-time feedback.
6.  **Rendering** -> Main Thread updates the DOM and calculates metrics.

---

## 3. Technical Implementation Details

### 3.1 Model Selection
[cite_start]The assignment required a model capable of paraphrasing while being downloadable[cite: 35].
* **Model Choice**: `LaMini-Flan-T5-248M`.
* **Rationale**: This is a "Distilled" version of T5, fine-tuned specifically on instruction datasets.
    * *Size*: ~250MB (Quantized). Small enough for browser caching but large enough to understand complex instructions.
    * *Architecture*: Encoder-Decoder Transformer.
    * *Optimization*: The model is quantized to 8-bit integers to run efficiently in WASM.

### 3.2 The Web Worker Strategy
To prevent the browser interface from freezing (locking up) during the heavy matrix multiplications required by the Neural Network, the inference logic is isolated in a **Web Worker**.

* **Main Thread**: Handles clicks, scroll, progress bar updates, and timer ticks.
* **Worker Thread**: Loads the ONNX runtime and executes the `.generate()` function.

**Communication Protocol:**
```javascript
// Main -> Worker
{ type: 'init' }       // Load model
{ type: 'generate', data: { text: "..." } } // Start job

// Worker -> Main
{ type: 'status', status: 'ready' } // Model loaded
{ type: 'progress', progress: 50, currentText: "..." } // Live update
{ type: 'complete', text: "..." } // Job finished
````

### 3.3 Structure Preservation Algorithm (The "Split-Merge" Logic)

A raw LLM generation often squashes input text into a single block. To preserve the format of a cover letter (Header -\> Date -\> Salutation -\> Body), the application uses a heuristic segmentation algorithm:

1.  **Segmentation**: Split input text by newline character `\n`.
2.  **Analysis**: Iterate through every line.
3.  **Heuristic Filter**:
      * **IF** line length \< 15 words: Assume it is metadata (Date, Name, Address). **Action**: Keep original text.
      * **IF** line length \>= 15 words: Assume it is a body paragraph. **Action**: Send to LLM for paraphrasing.
4.  **Reconstruction**: Join all lines (original metadata + paraphrased bodies) back with `\n`.

[cite_start]**Why this matters**: This ensures the output visually resembles the input, making it a viable CPG for structured documents like cover letters[cite: 10, 11].

### 3.4 Generation Parameters

[cite_start]To meet the requirement of generating text with a minimum length of 80% of the input[cite: 7], we dynamically adjust the generation config per paragraph:

```javascript
const output = await generator(prompt, {
    max_new_tokens: 512,
    // Strict constraint: Minimum output length must be 90% of input line length
    min_length: Math.floor(inputWordCount * 0.9), 
    temperature: 0.6,      // Balance creativity vs accuracy
    repetition_penalty: 1.2 // Prevent looping phrases
});
```

-----

## 4\. Evaluation Metrics System

[cite_start]The application computes metrics locally in real-time to satisfy the evaluation requirements[cite: 8, 29].

| Metric | Definition | Implementation Detail |
| :--- | :--- | :--- |
| **System Latency** | Time taken from button click to final render. | `performance.now()` start vs end timestamps. |
| **Length Compliance** | Ratio of Output Words / Input Words. | [cite_start]Target is \> 80%[cite: 7]. Code validates `(outputLen / inputLen) * 100`. |
| **CPG Overlap Score** | A proxy for semantic preservation and vocabulary variation. | **Jaccard Similarity Index**: Intersection of unique words divided by Union of unique words. |
| **LLM Comparison** | Benchmarking against external models (ChatGPT/Gemini). | Compares Jaccard score of Local CPG vs User-pasted External LLM output. |

-----

## 5\. Setup & Usage Guide

### Prerequisites

  * Modern Browser (Chrome v80+, Edge, Firefox).
  * Python (for local hosting).

### Installation

No `npm install` is required as all dependencies are loaded via CDN (ES Modules).

1.  Save the provided code as `index.html`.
2.  Open a terminal in that folder.
3.  Run the Python HTTP server to bypass browser security restrictions on Web Workers loading local files:
    ```bash
    python -m http.server 8000
    ```
4.  Navigate to `http://localhost:8000`.

### First Run

Upon first load, the browser will fetch approximately **250MB** of model weights. This is cached in the browser storage API. Subsequent reloads will be instant.

-----

## 6\. Future Improvements (Error Analysis)

  * **Heuristic Limitations**: The "15-word" rule for detecting paragraphs might fail on very short sentences or very long addresses. A Named Entity Recognition (NER) model would be more robust.
  * **Model Hallucination**: Small T5 models occasionally repeat phrases. Increasing `repetition_penalty` mitigates this but can affect fluency.
  * **WebGPU Support**: Currently relies on WASM. Enabling explicit WebGPU support in Transformers.js would speed up inference by 10x on supported hardware.




# LLM-Orchestrated Document Understanding Pipeline

## 🧩 Core Idea
Text extraction from document CLI tool (main idea).  

- **Input**: Directory of document page images.  
- **images will follow naming pattern like:**:
  - `page_1`
  - `2_page`  
- **Page number extraction**: Using regex (default: `page_(\d+)`).  
- **User override**: CLI argument to set custom regex.  
- **Page sorting**: Based on extracted numbers.

---

## ⚙️ Config & Input
- **Global config file** defines defaults.  
- **CLI args** override config.  
- **No runtime user interaction** required.  
- **Parameters include**:
  - Regex pattern
  - Column count (default = 2, fixed)
  - Text orientation (default = RTL for Pashto)
  - Max retries
  - Retry delay
  - Output directories
  - Confidence threshold (for LLM outputs)
  - Delay ranges:
    - Per page: 1–2 minutes
    - Intermediate: 20–50 seconds
  - Fallback LLM enable/disable (Gemini → GPT)

---

## 🌐 LLM Interface (Multi-Mode)
- **Primary**: Gemini UI (non-logged-in)  
- **Fallback**: GPT interface (UI or API if possible)  
- **Strategy**:
  1. Use Gemini as default.
  2. If:
     - Response invalid
     - Confidence below threshold
     - Retries exhausted  
     → Fallback to GPT.  
- **Optional**: Use both Gemini + GPT for critical steps (e.g., text extraction) and compare outputs for higher accuracy.

---

## 🧠 Prompt System
- Each stage has its own **prompt pool** (10 prompts per stage):
  - Column detection
  - JSON validation
  - Crop validation
  - Overflow detection
  - Text extraction

---

## 🎯 Prompt Selection (Controlled Variation)
- Deterministic selection (no randomness):  
  ```text
  prompt_index = (page_number + attempt_number) % total_prompts
- Applied independently per stage.  
- Ensures:
  - Reproducibility
  - Controlled variation
  - Better debugging

---

## 📐 Column Detection
- Upload full page image  
- Ask LLM for column coordinates  
- Expected JSON format:
```json
{
  "col1": { "ymin": int, "xmin": int, "ymax": int, "xmax": int },
  "col2": { "ymin": int, "xmin": int, "ymax": int, "xmax": int }
}
```

---

## 🧪 Response Handling (Smart JSON Extraction)
- Extract HTML via automation.  
- Multi-step JSON extraction:
1. Regex extraction
2. Bracket-based extraction
3. Fallback: 
   - ask LLM: “extract only JSON from this text”  
- Auto-fix JSON:
  - Remove trailing commas
  - Fix bracket balancing
  - Enforce strict key order
  - Reject duplicates
- If still invalid: 
  - send correction request to LLM

---

## 📊 Confidence Scoring Layer
- After each LLM response:
  - Ask LLM for confidence (0–100)  
    **or**  
  - Derive heuristics (length, format, structure)  
- If confidence < threshold:
  - Retry  
  - Or fallback to GPT

---

## ✂️ Image Processing
- Store coordinates in `/coords_output`  
- Crop image into columns  
- Save cropped images

---

## ✅ Crop Validation
- Upload cropped image  
- Ask LLM if crop is correct  
- If not correct: 
  - redo column detection

---

## 🔀 Overflow Handling
- Upload first column (right column for RTL)  
- Ask if text overflows  
- If YES:
  - Process second column
  - Extract overflow coordinates
  - Crop overflow
  - Merge images  
- if fails:
  - retry until max
  - then skip + log for manual fix

---

## 📝 Text Extraction (Critical Step)
- Upload final processed image  
- Extract text using prompt pool  
- **Accuracy Strategy**:
  - **Option A**:
    - Gemini → confidence check  
    - If low → retry → fallback to GPT  
  - **Option B (high accuracy)**:
    - Extract using Gemini
    - Extract using GPT
    - Compare outputs:
      - If similar → accept
      - If different → retry or flag

---

## 💾 Output
- Save extracted text  
- **Formats**:
  - CSV / TSV (primary for dictionary use case)
  - Generic text support

---

## 📦 Output Post-Processing (Dictionary Mode)
- Parse extracted text into:
  - Word
  - Meaning
  - Grammar
  - Additional metadata  
- Export structured CSV/TSV

---

## 🧾 Structured Logging
- Logs stored as structured JSON, example:
```json
{
  "page": 1,
  "stage": "column_detection",
  "attempt": 2,
  "prompt_index": 4,
  "status": "failed",
  "reason": "invalid_json"
}
```

- Logs
  - stored in files,  
  - accessible via API, 
  - printed to console.

---

## 🧠 Pipeline State File (Checkpointing++)
- **Per page state file example:**
```json
{
  "page": 1,
  "stage": "overflow_done",
  "coords_valid": true,
  "text_extracted": false
}
```


**Allows:**
- Resume from failure
- Skip completed steps
- Crash recovery

---

## 🔁 Failure Handling
- Max retries enforced  
- On failure:
  - Skip page
  - Store in `/failed_pages/failed.txt`
  - Log reason

---

## 🔄 Reprocessing Strategy
- After full run completes:
  - Reprocess failed pages
  - Second pass:
    - Different prompts
    - Stricter rules
    - Possibly GPT-first instead of Gemini

---

## 🔔 Notifications
- Notify on:
  - Completion
  - Failure
  - Retry exhaustion

---

## 🌐 API (Flask)
- **Endpoints:**
  - `/status`
  - `/logs`
  - `/progress`
  - `/current_page`  
- Accessible remotely via port forwarding

---

## 🚀 Execution Environment
- Runs on local machine  
- Fully headless  
- Monitored remotely

---

## ⏱️ Anti-Bot Strategy
- **Delays:**
  - Per page: 1–2 minutes
  - Intermediate: 20–50 seconds  
- Configurable

---

## 🧠 Final Notes on Multi-LLM Strategy
- **Recommended Strategy (Balanced):**
  - Use Gemini first
  - Ask for confidence
  - If confidence < threshold → retry
  - If still low → use GPT
  - For critical steps, optionally compare both outputs
- **Tradeoff:**
  - Gemini only → faster
  - Gemini + GPT → slower but much higher accuracy
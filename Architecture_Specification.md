# cernodata-internal: Application Architecture Specification

This document outlines how `cernodata-internal` solves the frustrating guesswork of PDF parsing for RAG. Instead of blindly running a parser and hoping for usable chunks, we use guided decision trees, automated fallback loops, page slicing, and visual debugging to guarantee clean output. Above all, the architecture is engineered to dramatically shave developer iteration time by finding ways to inspect, debug, and validate data without having to execute heavy ML tools unnecessarily.

---

## 1. Preset Selection Decision Tree

Nobody wants to read 50 pages of docs just to pick an OCR engine. The setup wizard asks a few targeted questions and configures the right pipeline for you.

### Inputs & Weighting Logic
- **Document Taxonomy**: Financial reports, multi-column articles, scanned forms, tabular ledgers, mixed text/images.
- **Hardware Profile**: Low-spec PC (shared CPU/RAM), workstation with CUDA GPU, cluster/cloud node.
- **Quality vs. Speed Target**: Rapid approximate extraction vs. high-precision structural extraction.
- **Security Constraints**: Zero external API calls (air-gapped local model) vs. hosted vision APIs.

### Output
- Evaluates answers to assign initial confidence probabilities $P(\text{Preset}_i | \text{Answers})$ across candidate presets.
- Emits a declarative YAML pipeline configuration selecting the primary preset and fallback ranking queue.

---

## 2. Curated & Extensible Parsing Presets

Parsers are wrapped as isolated, parameter-configurable presets combining layout engines, OCR drivers, and post-processors.

### 2.1 Curated Presets
- **Docling + Fast OCR**: Docling layout parser with Tesseract / RapidOCR (optimized for standard PDFs, low compute).
- **Docling + Deep OCR**: Docling paired with Surya / EasyOCR (optimized for complex layouts, dense tables).
- **Vision LLM Direct**: Multi-modal vision model (e.g. Qwen2-VL, PaddleOCR-v4) for heavily distorted scans.

### 2.2 User-Defined Custom Presets
- Users can register custom presets via simple YAML configurations or Python classes.
- Allows substituting OCR engines, wiggling parameters (DPI, binarization threshold, table detection mode), or plugging in domain-specific post-processors.

---

## 3. Automated Confidence-Guided Iteration Loop

When a document is parsed, it undergoes automated quality evaluation against target confidence thresholds.

```
                    ┌─────────────────────────┐
                    │ Execute Selected Preset │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Run Quality Heuristics  │
                    └────────────┬────────────┘
                                 │
                   Confidence >= Target Threshold?
                        ┌────────┴────────┐
                       YES               NO
                        │                 │
                        ▼                 ▼
                   ┌──────────┐ ┌───────────────────┐
                   │ Accept   │ │  Execute Dual-Path│
                   │ Output   │ │   Fallback Loop   │
                   └──────────┘ └─────────┬─────────┘
                                          │
            ┌─────────────────────────────┴─────────────────────────────┐
            ▼                                                           ▼
┌─────────────────────────┐                                 ┌─────────────────────────┐
│ Path A: Switch Preset   │                                 │ Path B: Wiggle Parameters│
│ Re-order queue using    │                                 │ Adjust DPI, contrast,   │
│ decision tree weights   │                                 │ table detection mode    │
└─────────────────────────┘                                 └─────────────────────────┘
```

### 3.1 Tiered Quality Check Continuum
We don't want a black box deciding what counts as "good data". Instead, users get a clear ladder of checks they can inspect, tune, and turn on as needed:
1. **Generic Garbage Detection**: Measures ratio of non-printable characters, ungrammatical unicode symbols, and character repetition heuristics.
2. **Language & Dictionary Checks**: Optional user-provided language hints (e.g. `en`, `de`) to score dictionary word hit rate.
3. **Target Keyword / Term Spotting**: User-supplied required terms or regular expressions (e.g., invoice numbers, section headers).
4. **Structural Table Grid Checks**: Validates that detected table bounding boxes contain aligned cell boundaries rather than fragmented text blocks.
5. **RAG Vector Search Signal Check**: Downstream similarity scoring against test queries in a DuckDB vector sandbox.

### 3.2 Dual-Path Fallback Execution
If confidence falls below the target threshold:
- **Path A (Preset Switch)**: Pulls the next highest-weighted preset from the decision queue.
- **Path B (Parameter Wiggling)**: Retries the current preset with adjusted parameters (e.g. higher DPI rendering, contrast adjustments, enabling strict table structure mode).

---

## 4. Page-Level PDF Subdivision

Instead of re-running a 100-page document from scratch just because page 42 failed, we evaluate quality page-by-page.

### Workflow
1. **Page-Granularity Scoring**: Computes confidence scores per page $S_1, S_2, \dots, S_n$.
2. **Page Slicing**: High-confidence pages ($S_i \ge T$) are locked and stored. Unconfident pages ($S_j < T$) are sliced into temporary single-page PDFs.
3. **Targeted Re-Processing**: Sliced unconfident pages are routed through heavier fallback presets independently.
4. **Reassembly & Provenance Mapping**: Page DOM objects explicitly track both `global_page_index` (original document context) and `temp_slice_index` (slice context), guaranteeing exact geometric bounding-box mapping upon reassembly without page-number drift.

---

## 5. Structural Extraction, Bounding Box Overlays & Template Hints

### 5.1 Structural DOM Primitives
Extracted output is structured into geometric DOM objects: text blocks, multi-column regions, table grids, headers/footers, and figure captions.

### 5.2 Visual Provenance UI Overlay
- Side-by-side visual explorer rendering PDF pages alongside parsed DOM trees.
- Interactive bounding box masks highlight structural detection (e.g. displaying whether a 2-column layout was incorrectly merged as 1-column).
- Enables 1-click manual override to toggle layout hints (e.g., "Force Multi-Column Mode").

### 5.3 Standardized Document Template Hints
- **Template Selection Subflow**: During the "Add Documents" ingestion wizard, users select an applicable template (defaulting to the last template used so users can process recurring batches of financial statements seamlessly without passing CLI flags).
- **Page-Level Structural Hints**: For recurring structures, users can save page-level structural hints (e.g. *"Page 3 always contains a standardized balance sheet table"*). Bypasses structure discovery overhead and forces specialized table extraction parsers on designated pages.
- **Misapplication Detection**: Automated template misapplication detection flags when an ingested file deviates from its assigned template schema, prompting the user for resolution.

---

## 6. Downstream Evaluation Decision Tree & SLM Prompting

Knowing whether your parsed data is actually good requires testing. A simple decision tree guides you in setting up test cases for your vector DB, RAG pipeline, or AI agent.

### 6.1 Test Suite Types
- **Structural Queries**: Schema validation, key-value pair checks, and row/column count expectations.
- **Semantic Queries**: Domain expert Q&A requiring synthesis across multiple chunks.

### 6.2 SLM-Driven Test Prompt Generation
To speed up test suite creation, a local 1B model (via Ollama API, e.g., `qwen2.5:1.5b` or `llama3.2:1b`) scans your extracted text and automatically proposes relevant test prompts—giving you a ready-made test suite in seconds that you can review, edit, or override.

---

## 7. Downstream Packing & Target Use-Case Decision Tree

How you format and chunk your data depends entirely on what's consuming it downstream.

| Target Downstream Integration | Recommended Packing & Chunking Strategy |
| :--- | :--- |
| **Vector Search / Hybrid RAG** | Hierarchical parent-child chunking (300-token child, full section parent) with page bounding box metadata. |
| **Conversational Chatbot** | Conversational section summaries with inline table Markdown representations. |
| **Autonomous Action Agents** | Structured JSON schema extraction preserving explicit tool-calling fields and API payload shapes. |
| **SFT / Fine-Tuning Datasets** | Synthetic QA pair generation (Instruction/Input/Response) formatted for LLaMA / Alpaca fine-tuning. |

---

## 8. Resource Protection & Work Dispatching Subsystem

Running heavy OCR locally shouldn't freeze your machine while you're trying to browse the web, edit code, or run a local model in Ollama.

### 8.1 Machine Profiles & Conversational Tuning
Upon setup, a brief interactive prompt configures resource bounds:
- **Responsive Desktop Profile (Default)**: Concurrency capped at 1 heavy worker process, strict RSS memory ceilings (e.g., 4GB RAM cap), low process CPU priority.
- **Dedicated Workstation Profile**: Multi-worker parallel processing, moderate RAM allocations.
- **Cluster / Container Profile**: Full multi-core execution, Docker containerization, or gRPC worker dispatching.

### 8.2 Structure-Only Debug Passes vs. Full OCR
Running full OCR across 500 pages just to check if table borders were detected takes forever. The "Structure-Only" mode quickly draws layout boxes on your pages so you can visually verify detection in seconds before burning compute on full text extraction.
- **Engine Support Tiering**: Initially enabled for layout-friendly engines (e.g., Docling, PaddleLayout, Surya) that naturally expose structural layout bounding boxes without running full text OCR.
- **Deep Engine Interop Roadmap**: To ensure a homogeneous iteration experience across all engines, future iterations will establish deeper interoperation with engines like Tesseract to extract internal layout primitives and squeeze the maximum quality out of legacy drivers.

### 8.3 Heavy Component Isolation (Docling / PyTorch)
PyTorch memory leaks are infamous for quietly crashing long batch jobs. Python ML workers run inside isolated subprocesses managed by the Go core that automatically restart after a set number of pages to keep memory clean.

---

## 9. Interactive Visual Web App & Audit-Friendly Standalone Script Exporter

To lower adoption friction and build trust, initial development focuses on an interactive visual web application, curated benchmark sample documents, and transparent script generation.

### 9.1 Interactive Visual Flow App & Step-by-Step Manual Execution
- Web UI serving as a visual simulator of the end-to-end pipeline.
- In the initial phase, users run exported preset scripts manually step-by-step. This allows users to test presets on troublesome documents, visually inspect fallback paths, and evaluate the decision trees before an end-product engine automates execution behind the scenes.
- **Troublesome Sample Suite**: The app will bundle a benchmark suite of "hard-to-parse" sample documents (multi-column financial reports, dense tabular ledgers, scanned forms) that typically cause standard RAG pipelines to fail. This demonstrates how our fallback iteration provides qualitative improvement over conventional tools.

### 9.2 Standalone Python Script Exporter
- Every preset can be exported as a short, zero-dependency / minimal standalone Python script (e.g., `parse_preset_docling_tesseract.py`).
- If you don't trust a heavy background daemon running on your computer, you can grab the standalone Python script, inspect the code, and run it directly in your terminal. (This makes auditing effortless: security-minded users can read the raw Python code, verify what it does, and run it independently without installing our full runtime.)

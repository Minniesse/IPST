# Workshop Design: การเตรียมและจัดโครงสร้างข้อมูลสำหรับสอนโมเดล

**Date**: 2026-02-18
**Duration**: 3-4 hours (half day)
**Audience**: Very beginners
**Format**: Jupyter notebooks
**Language**: Thai-language source documents

## Goal

Teach participants how to extract data from Thai PDF/PPTX documents and structure it into datasets for both LLM fine-tuning (SFT) and RAG.

## Approach

**Docling + Ollama Manual Pipeline** — transparent, educational, minimal dependencies. Each step is explicit Python code in a notebook so beginners understand every transformation.

## Tech Stack

| Tool | Purpose | Version |
|---|---|---|
| `docling` | Document extraction (PDF, PPTX -> Markdown/JSON) | latest |
| `ollama` + `qwen2.5` (3B/7B) | Local LLM for Q&A generation | latest |
| `pythainlp` | Thai tokenization, normalization, language detection | latest |
| `sentence-transformers` | RAG embeddings (`intfloat/multilingual-e5-small`) | latest |
| `pandas`, `tqdm` | Data manipulation and progress bars | latest |

## Pipeline Architecture

```
Thai PDF/PPTX
    |
    v
[Notebook 1] Docling extraction -> DoclingDocument -> Markdown + JSON
    |
    v
[Notebook 2] pythainlp normalize -> chunk by section/size -> chunks.jsonl
    |
    v
[Notebook 3] Ollama (qwen2.5) -> Q&A pairs -> quality filter -> sft_dataset.jsonl
    |
    v
[Notebook 4] SFT format (Alpaca/ShareGPT) + RAG embeddings -> final datasets
```

## Notebook Details

### Notebook 1: Document Extraction with Docling (45 min)

**Learning objectives:**
- Install and configure Docling
- Load Thai PDF and PPTX files
- Extract content to Markdown and JSON formats
- Explore the DoclingDocument structure (text, tables, images, hierarchy)
- Handle Thai-specific OCR if needed (EasyOCR/Tesseract)

**Key code:**
```python
from docling.document_converter import DocumentConverter

converter = DocumentConverter()
result = converter.convert("thai_document.pdf")
markdown = result.document.export_to_markdown()
json_doc = result.document.export_to_dict()
```

### Notebook 2: Data Cleaning & Chunking (45 min)

**Learning objectives:**
- Clean extracted Thai text (encoding, whitespace, mixed scripts)
- Normalize Thai text with `pythainlp.util.normalize`
- Apply chunking strategies: fixed-size and semantic (by headers/sections)
- Add metadata to chunks (source file, page number, section title)
- Export chunks to JSONL format

**Thai-specific considerations:**
- Thai has no spaces between words — use `pythainlp.tokenize.word_tokenize` for word-level chunking
- Handle Thai Unicode normalization (tone marks, vowel forms)
- Detect and handle mixed Thai-English text

### Notebook 3: LLM-Assisted Data Structuring with Ollama (60 min)

**Learning objectives:**
- Setup Ollama with Qwen2.5 (3B for CPU, 7B for GPU)
- Design prompt templates for generating Q&A pairs from Thai text chunks
- Generate instruction/input/output triples for SFT
- Generate summaries and metadata
- Quality filtering: deduplication, length checks, language detection
- Export to Alpaca format JSONL

**Prompt template example:**
```
Given the following Thai text, generate 3 question-answer pairs in Thai.
The questions should test understanding of the content.

Text: {chunk}

Output format:
Q1: ...
A1: ...
```

### Notebook 4: Building Final Datasets (30 min)

**Learning objectives:**
- RAG path: create embeddings with `multilingual-e5-small`, store as JSONL with vectors
- SFT path: validate dataset format, check for completeness
- Compute dataset statistics: length distribution, topic coverage
- Visualize dataset quality
- Next steps: how to use with Hugging Face Hub, Unsloth, TRL

## Output Formats

### SFT Dataset (Alpaca format)
```json
{
  "instruction": "อธิบายแนวคิดหลักของ...",
  "input": "",
  "output": "แนวคิดหลักคือ..."
}
```

### RAG Dataset
```json
{
  "chunk_id": "doc1_chunk_003",
  "text": "เนื้อหาข้อความ...",
  "metadata": {"source": "thai_doc.pdf", "page": 3, "section": "บทที่ 1"},
  "embedding": [0.012, -0.034, ...]
}
```

## Important Considerations

1. **Data quality validation** — Filter bad LLM outputs (hallucinations, wrong language, too short/long)
2. **Deduplication** — Remove duplicate text across pages/sections
3. **Thai text normalization** — Normalize Unicode variants before processing
4. **Chunk overlap for RAG** — Use overlapping windows to prevent context loss at boundaries
5. **Prompt engineering** — Include well-tested prompt templates for Q&A generation
6. **Dataset statistics** — Show participants how to analyze their output
7. **Sample documents** — Provide pre-selected Thai PDFs/PPTXs for consistent workshop experience
8. **Hardware fallback** — Ensure Qwen2.5-3B runs on CPU for participants without GPU

## Prerequisites for Participants

- Python 3.10+
- `uv` package manager
- Ollama installed with `qwen2.5:3b` pulled
- Basic Python knowledge (variables, loops, functions)

## References

- [Docling](https://github.com/docling-project/docling) — Document extraction
- [Data Prep Kit](https://github.com/data-prep-kit/data-prep-kit) — IBM's data preparation toolkit
- [Easy Dataset](https://github.com/ConardLi/easy-dataset) — GUI-based dataset builder (alternative tool)
- [Distilabel](https://github.com/argilla-io/distilabel) — Synthetic data generation framework
- [pythainlp](https://github.com/PyThaiNLP/pythainlp) — Thai NLP toolkit
- [Granite-Docling VLM](https://huggingface.co/ibm-granite/granite-docling-258M) — Document understanding model

# Data Prep Workshop Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build 4 Jupyter notebooks for a beginner workshop on extracting Thai documents (PDF/PPTX) with Docling and structuring them into SFT and RAG datasets using Ollama (Qwen2.5).

**Architecture:** Linear notebook pipeline — each notebook produces output files consumed by the next. Notebook 1 extracts documents, Notebook 2 cleans/chunks text, Notebook 3 uses Ollama to generate Q&A pairs, Notebook 4 builds final datasets with embeddings and statistics.

**Tech Stack:** Docling, Ollama (qwen2.5:3b), pythainlp, sentence-transformers, pandas, tqdm, matplotlib. Package management via `uv`.

---

## Task 0: Project Setup

**Files:**
- Create: `pyproject.toml`
- Create: `data/samples/` (directory for sample Thai documents)
- Create: `output/` (directory for intermediate and final outputs)

**Step 1: Initialize project with uv**

```bash
cd /home/moonie/IPST
uv init workshop --no-readme
cd workshop
```

**Step 2: Add dependencies to pyproject.toml**

Replace the generated `pyproject.toml` with:

```toml
[project]
name = "data-prep-workshop"
version = "0.1.0"
description = "Workshop: การเตรียมและจัดโครงสร้างข้อมูลสำหรับสอนโมเดล"
requires-python = ">=3.10"
dependencies = [
    "docling>=2.70",
    "ollama>=0.4",
    "pythainlp>=5.0",
    "sentence-transformers>=3.0",
    "pandas>=2.1",
    "tqdm>=4.65",
    "matplotlib>=3.8",
    "ipykernel>=6.29",
    "tabulate>=0.9",
]
```

**Step 3: Install dependencies**

```bash
uv sync
```

**Step 4: Create directory structure**

```bash
mkdir -p data/samples output/extracted output/chunks output/datasets
```

**Step 5: Create a sample Thai PDF for testing**

We need at least one Thai PDF and one Thai PPTX as sample data. Either:
- Download a freely available Thai government document/textbook PDF
- Or create a minimal Thai PDF using Python for testing

For the workshop, place sample documents in `data/samples/`.

---

## Task 1: Notebook 1 — Document Extraction with Docling

**Files:**
- Create: `notebooks/01_document_extraction.ipynb`

This notebook has ~12 cells covering installation verification, single PDF conversion, PPTX conversion, batch conversion, table extraction, and saving outputs.

**Cell 1 (Markdown): Title and overview**

```markdown
# Notebook 1: การดึงข้อมูลจากเอกสารด้วย Docling
# (Document Extraction with Docling)

## สิ่งที่จะได้เรียนรู้ (What you'll learn)
- ติดตั้งและใช้งาน Docling สำหรับแปลงเอกสาร
- แปลงไฟล์ PDF และ PPTX ภาษาไทย
- สำรวจโครงสร้าง DoclingDocument
- ดึงตาราง รูปภาพ และลำดับชั้นของเอกสาร
- บันทึกผลลัพธ์เป็น Markdown และ JSON
```

**Cell 2 (Code): Verify installation**

```python
# ตรวจสอบการติดตั้ง (Verify installation)
import docling
print(f"Docling version: {docling.__version__}")

from docling.document_converter import DocumentConverter
print("DocumentConverter imported successfully!")
```

**Cell 3 (Markdown): Explain DocumentConverter**

```markdown
## DocumentConverter คืออะไร?

`DocumentConverter` เป็นตัวแปลงเอกสารหลักของ Docling ที่รองรับหลายรูปแบบ:
- **PDF** — รวมถึง scanned PDF ที่ต้องใช้ OCR
- **PPTX** — ไฟล์ PowerPoint
- **DOCX** — ไฟล์ Word
- **HTML, Images, และอื่นๆ**

ผลลัพธ์จะได้เป็น `DoclingDocument` ซึ่งเป็นโครงสร้างข้อมูลแบบรวม (unified representation)
```

**Cell 4 (Code): Convert a single Thai PDF**

```python
from docling.document_converter import DocumentConverter
from pathlib import Path

# กำหนดไฟล์ต้นทาง (Set source file)
pdf_path = Path("../data/samples/thai_sample.pdf")

# สร้าง converter และแปลงเอกสาร
converter = DocumentConverter()
result = converter.convert(str(pdf_path))

print(f"สถานะการแปลง: {result.status}")
print(f"ชื่อไฟล์: {result.input.file.name}")
```

**Cell 5 (Code): Export to Markdown**

```python
# ดูผลลัพธ์เป็น Markdown
markdown_text = result.document.export_to_markdown()

print("=" * 60)
print("Markdown Output (500 ตัวอักษรแรก):")
print("=" * 60)
print(markdown_text[:500])
print(f"\n... (ทั้งหมด {len(markdown_text)} ตัวอักษร)")
```

**Cell 6 (Code): Export to JSON and explore structure**

```python
import json

# ดูโครงสร้าง JSON
json_doc = result.document.export_to_dict()

# แสดง keys ระดับบนสุด
print("Keys ในเอกสาร:")
for key in json_doc.keys():
    print(f"  - {key}")

# บันทึกเป็นไฟล์
output_path = Path("../output/extracted")
output_path.mkdir(parents=True, exist_ok=True)

with open(output_path / f"{pdf_path.stem}.json", "w", encoding="utf-8") as f:
    json.dump(json_doc, f, ensure_ascii=False, indent=2)

print(f"\nบันทึก JSON ไปที่: {output_path / f'{pdf_path.stem}.json'}")
```

**Cell 7 (Code): Explore document hierarchy**

```python
# สำรวจโครงสร้างของเอกสาร (Document hierarchy)
doc = result.document

# นับจำนวน elements
print("สรุปโครงสร้างเอกสาร:")
print(f"  จำนวนตาราง (Tables): {len(list(doc.tables))}")
print(f"  จำนวนรูปภาพ (Pictures): {len(list(doc.pictures))}")

# แสดง text elements
print("\nเนื้อหาข้อความ (แสดง 5 รายการแรก):")
for i, (item, _level) in enumerate(doc.iterate_items()):
    if i >= 5:
        print("  ...")
        break
    text_preview = item.text[:80] if hasattr(item, 'text') else str(type(item).__name__)
    print(f"  [{i}] {text_preview}")
```

**Cell 8 (Code): Extract tables**

```python
import pandas as pd

# ดึงตารางจากเอกสาร
tables = list(doc.tables)
print(f"พบตาราง {len(tables)} ตาราง\n")

for i, table in enumerate(tables):
    print(f"--- ตาราง {i+1} ---")
    df = table.export_to_dataframe(doc=doc)
    print(df.to_markdown(index=False))

    # บันทึกเป็น CSV
    csv_path = output_path / f"{pdf_path.stem}_table_{i+1}.csv"
    df.to_csv(csv_path, index=False)
    print(f"บันทึกที่: {csv_path}\n")
```

**Cell 9 (Code): Convert PPTX**

```python
# แปลงไฟล์ PowerPoint
pptx_path = Path("../data/samples/thai_sample.pptx")

if pptx_path.exists():
    pptx_result = converter.convert(str(pptx_path))
    pptx_markdown = pptx_result.document.export_to_markdown()

    print("PPTX Markdown Output (500 ตัวอักษรแรก):")
    print("=" * 60)
    print(pptx_markdown[:500])

    # บันทึก
    with open(output_path / f"{pptx_path.stem}.json", "w", encoding="utf-8") as f:
        json.dump(pptx_result.document.export_to_dict(), f, ensure_ascii=False, indent=2)

    # บันทึก Markdown
    with open(output_path / f"{pptx_path.stem}.md", "w", encoding="utf-8") as f:
        f.write(pptx_markdown)

    print(f"\nบันทึกผลลัพธ์ไปที่ {output_path}/")
else:
    print(f"ไม่พบไฟล์ {pptx_path} — ข้ามขั้นตอนนี้")
```

**Cell 10 (Code): Batch convert all documents**

```python
from pathlib import Path

# แปลงไฟล์ทั้งหมดในโฟลเดอร์ data/samples/
sample_dir = Path("../data/samples")
all_files = list(sample_dir.glob("*.pdf")) + list(sample_dir.glob("*.pptx"))

print(f"พบไฟล์ {len(all_files)} ไฟล์:")
for f in all_files:
    print(f"  - {f.name}")

# แปลงทีละไฟล์
results = []
for file_path in all_files:
    print(f"\nกำลังแปลง: {file_path.name}...")
    res = converter.convert(str(file_path))
    results.append(res)

    # บันทึก Markdown
    md_path = output_path / f"{file_path.stem}.md"
    with open(md_path, "w", encoding="utf-8") as f:
        f.write(res.document.export_to_markdown())

    # บันทึก JSON
    json_path = output_path / f"{file_path.stem}.json"
    with open(json_path, "w", encoding="utf-8") as f:
        json.dump(res.document.export_to_dict(), f, ensure_ascii=False, indent=2)

    print(f"  สถานะ: {res.status}")
    print(f"  บันทึก: {md_path.name}, {json_path.name}")

print(f"\nแปลงเสร็จสิ้น {len(results)} ไฟล์!")
```

**Cell 11 (Markdown): Save Markdown for next notebook**

```markdown
## บันทึกผลลัพธ์สำหรับ Notebook ถัดไป

เราบันทึกข้อมูลที่ดึงออกมาไว้ใน `output/extracted/`:
- ไฟล์ `.md` — Markdown สำหรับอ่านง่าย
- ไฟล์ `.json` — JSON สำหรับประมวลผลต่อ
- ไฟล์ `.csv` — ตารางที่ดึงออกมา

ใน Notebook 2 เราจะนำข้อมูลเหล่านี้มาทำความสะอาดและแบ่งเป็นชิ้นส่วน (chunks)
```

**Cell 12 (Code): Summary**

```python
import os

# สรุปไฟล์ที่สร้าง
print("ไฟล์ที่สร้างใน output/extracted/:")
for f in sorted(output_path.iterdir()):
    size = f.stat().st_size
    print(f"  {f.name} ({size:,} bytes)")
```

**Step: Verify notebook runs**

Run: `uv run jupyter execute notebooks/01_document_extraction.ipynb`
Expected: All cells execute without error (assuming sample data exists).

---

## Task 2: Notebook 2 — Data Cleaning & Chunking

**Files:**
- Create: `notebooks/02_cleaning_and_chunking.ipynb`

This notebook covers Thai text normalization, two chunking strategies (Docling native + custom fixed-size), metadata attachment, deduplication, and JSONL export.

**Cell 1 (Markdown): Title**

```markdown
# Notebook 2: การทำความสะอาดและแบ่งชิ้นส่วนข้อมูล
# (Data Cleaning & Chunking)

## สิ่งที่จะได้เรียนรู้
- ทำความสะอาดข้อความภาษาไทย (Thai text normalization)
- แบ่งข้อมูลเป็นชิ้นส่วน (chunking) 2 วิธี: Docling native และ fixed-size
- เพิ่ม metadata ให้แต่ละ chunk
- ลบข้อมูลซ้ำ (deduplication)
- บันทึกเป็น JSONL สำหรับใช้งานต่อ
```

**Cell 2 (Code): Load extracted data**

```python
import json
from pathlib import Path

# โหลดข้อมูลที่ดึงออกมาจาก Notebook 1
extracted_dir = Path("../output/extracted")
json_files = list(extracted_dir.glob("*.json"))

print(f"พบ {len(json_files)} ไฟล์ JSON:")
for f in json_files:
    print(f"  - {f.name}")

# โหลดไฟล์แรกเป็นตัวอย่าง
sample_file = json_files[0]
with open(sample_file, "r", encoding="utf-8") as f:
    doc_data = json.load(f)

print(f"\nกำลังใช้: {sample_file.name}")
```

**Cell 3 (Code): Thai text normalization with pythainlp**

```python
from pythainlp.util import normalize as thai_normalize
from pythainlp.tokenize import word_tokenize

# ตัวอย่างการ normalize ข้อความภาษาไทย
sample_texts = [
    "ภาษาไทย  มี   ช่องว่าง  หลายที่",
    "ข้อความที\u0e49มีรูปแบบ Unicode ต่างกัน",  # different Unicode form
    "Mixed Thai and English text การผสม",
]

print("ก่อน/หลัง Normalization:")
print("=" * 60)
for text in sample_texts:
    normalized = thai_normalize(text)
    print(f"ก่อน: {text!r}")
    print(f"หลัง: {normalized!r}")
    print()
```

**Cell 4 (Code): Define cleaning function**

```python
import re
from pythainlp.util import normalize as thai_normalize

def clean_thai_text(text: str) -> str:
    """ทำความสะอาดข้อความภาษาไทย"""
    # 1. Normalize Unicode
    text = thai_normalize(text)

    # 2. ลบช่องว่างซ้ำ (collapse multiple spaces)
    text = re.sub(r' +', ' ', text)

    # 3. ลบบรรทัดว่างซ้ำ (collapse multiple newlines)
    text = re.sub(r'\n{3,}', '\n\n', text)

    # 4. ตัดช่องว่างหัวท้าย
    text = text.strip()

    return text

# ทดสอบ
test = "  ข้อความ   ที่มี    ช่องว่าง  เยอะ  \n\n\n\nและบรรทัดว่าง  "
print(f"ก่อน: {test!r}")
print(f"หลัง: {clean_thai_text(test)!r}")
```

**Cell 5 (Markdown): Explain chunking strategies**

```markdown
## วิธีการแบ่งชิ้นส่วน (Chunking Strategies)

### 1. Docling HybridChunker (แนะนำ)
- ใช้โครงสร้างเอกสารจริง (หัวข้อ, ย่อหน้า, ตาราง)
- รวม chunks ที่เล็กเกินไป และแยก chunks ที่ใหญ่เกินไป
- เหมาะกับ RAG เพราะรักษาบริบทตามโครงสร้าง

### 2. Fixed-Size Chunking
- ตัดข้อความตามจำนวนตัวอักษรหรือ tokens
- ใช้ overlap เพื่อไม่ให้สูญเสียบริบทตรงรอยต่อ
- เหมาะเมื่อเอกสารไม่มีโครงสร้างชัดเจน
```

**Cell 6 (Code): Docling HybridChunker**

```python
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker

# แปลงเอกสารใหม่เพื่อได้ DoclingDocument object
sample_path = list(Path("../data/samples").glob("*.pdf"))[0]
converter = DocumentConverter()
result = converter.convert(str(sample_path))
doc = result.document

# ใช้ HybridChunker
chunker = HybridChunker(max_tokens=512)
chunks_iter = chunker.chunk(dl_doc=doc)
docling_chunks = list(chunks_iter)

print(f"HybridChunker สร้าง {len(docling_chunks)} chunks\n")

# แสดงตัวอย่าง 3 chunks แรก
for i, chunk in enumerate(docling_chunks[:3]):
    enriched = chunker.contextualize(chunk=chunk)
    print(f"--- Chunk {i+1} ---")
    print(f"ข้อความ: {chunk.text[:200]}...")
    print(f"พร้อมบริบท: {enriched[:200]}...")
    print()
```

**Cell 7 (Code): Fixed-size chunking with overlap**

```python
def fixed_size_chunk(text: str, chunk_size: int = 500, overlap: int = 50) -> list[dict]:
    """แบ่งข้อความเป็นชิ้นส่วนขนาดคงที่พร้อม overlap"""
    chunks = []
    start = 0
    chunk_id = 0

    while start < len(text):
        end = start + chunk_size
        chunk_text = text[start:end]

        # ไม่ตัดกลางคำ — หาจุดตัดที่เหมาะสม (ช่องว่างหรือขึ้นบรรทัดใหม่)
        if end < len(text):
            last_break = max(
                chunk_text.rfind('\n'),
                chunk_text.rfind(' '),
                chunk_text.rfind('。'),
            )
            if last_break > chunk_size * 0.5:  # ถ้าหาจุดตัดได้ในครึ่งหลัง
                chunk_text = chunk_text[:last_break]
                end = start + last_break

        chunks.append({
            "chunk_id": f"chunk_{chunk_id:04d}",
            "text": chunk_text.strip(),
            "start_char": start,
            "end_char": end,
        })

        chunk_id += 1
        start = end - overlap  # เลื่อนไปข้างหน้าโดยมี overlap

    return chunks

# โหลด Markdown ที่ดึงออกมา
md_files = list(extracted_dir.glob("*.md"))
if md_files:
    with open(md_files[0], "r", encoding="utf-8") as f:
        raw_text = f.read()

    cleaned = clean_thai_text(raw_text)
    fixed_chunks = fixed_size_chunk(cleaned, chunk_size=500, overlap=50)

    print(f"Fixed-size chunks: {len(fixed_chunks)} chunks")
    print(f"\nตัวอย่าง chunk แรก:")
    print(json.dumps(fixed_chunks[0], ensure_ascii=False, indent=2))
```

**Cell 8 (Code): Add metadata to chunks**

```python
def add_metadata(chunks: list[dict], source_file: str, method: str) -> list[dict]:
    """เพิ่ม metadata ให้แต่ละ chunk"""
    enriched = []
    for chunk in chunks:
        chunk_with_meta = {
            **chunk,
            "metadata": {
                "source": source_file,
                "chunking_method": method,
                "char_count": len(chunk["text"]),
            }
        }
        enriched.append(chunk_with_meta)
    return enriched

# เพิ่ม metadata ให้ fixed-size chunks
if fixed_chunks:
    enriched_chunks = add_metadata(
        fixed_chunks,
        source_file=md_files[0].name,
        method="fixed_size_500_overlap_50"
    )

    print("ตัวอย่าง chunk พร้อม metadata:")
    print(json.dumps(enriched_chunks[0], ensure_ascii=False, indent=2))
```

**Cell 9 (Code): Deduplication**

```python
from hashlib import md5

def deduplicate_chunks(chunks: list[dict]) -> list[dict]:
    """ลบ chunks ที่ซ้ำกัน (โดยเปรียบเทียบเนื้อหา)"""
    seen_hashes = set()
    unique_chunks = []
    duplicates = 0

    for chunk in chunks:
        text_hash = md5(chunk["text"].encode("utf-8")).hexdigest()
        if text_hash not in seen_hashes:
            seen_hashes.add(text_hash)
            unique_chunks.append(chunk)
        else:
            duplicates += 1

    print(f"ลบ chunks ซ้ำ: {duplicates} จากทั้งหมด {len(chunks)}")
    return unique_chunks

# ทดสอบ dedup
if enriched_chunks:
    unique_chunks = deduplicate_chunks(enriched_chunks)
    print(f"เหลือ {len(unique_chunks)} chunks ที่ไม่ซ้ำ")
```

**Cell 10 (Code): Process all documents and export JSONL**

```python
from tqdm import tqdm

# ประมวลผลทุกไฟล์
all_chunks = []

for md_file in tqdm(md_files, desc="กำลังแบ่งชิ้นส่วน"):
    with open(md_file, "r", encoding="utf-8") as f:
        text = f.read()

    # ทำความสะอาด
    cleaned = clean_thai_text(text)

    # แบ่ง chunks
    chunks = fixed_size_chunk(cleaned, chunk_size=500, overlap=50)

    # เพิ่ม metadata
    chunks = add_metadata(chunks, source_file=md_file.name, method="fixed_size")

    all_chunks.extend(chunks)

# Dedup
all_chunks = deduplicate_chunks(all_chunks)

# บันทึกเป็น JSONL
output_path = Path("../output/chunks")
output_path.mkdir(parents=True, exist_ok=True)
jsonl_path = output_path / "chunks.jsonl"

with open(jsonl_path, "w", encoding="utf-8") as f:
    for chunk in all_chunks:
        f.write(json.dumps(chunk, ensure_ascii=False) + "\n")

print(f"\nบันทึก {len(all_chunks)} chunks ไปที่ {jsonl_path}")
```

**Cell 11 (Code): Chunk statistics**

```python
import matplotlib.pyplot as plt

# สถิติของ chunks
lengths = [len(c["text"]) for c in all_chunks]

print("สถิติ Chunks:")
print(f"  จำนวน: {len(all_chunks)}")
print(f"  ความยาวเฉลี่ย: {sum(lengths)/len(lengths):.0f} ตัวอักษร")
print(f"  สั้นสุด: {min(lengths)} ตัวอักษร")
print(f"  ยาวสุด: {max(lengths)} ตัวอักษร")

# กราฟ
plt.figure(figsize=(10, 4))
plt.hist(lengths, bins=30, edgecolor='black', alpha=0.7)
plt.xlabel("ความยาว (ตัวอักษร)")
plt.ylabel("จำนวน chunks")
plt.title("การกระจายความยาวของ Chunks")
plt.tight_layout()
plt.savefig("../output/chunks/chunk_length_distribution.png", dpi=100)
plt.show()
```

---

## Task 3: Notebook 3 — LLM-Assisted Data Structuring

**Files:**
- Create: `notebooks/03_llm_structuring.ipynb`

This notebook covers Ollama setup, prompt design, Q&A pair generation, quality filtering, and SFT dataset export.

**Cell 1 (Markdown): Title**

```markdown
# Notebook 3: การใช้ LLM ช่วยจัดโครงสร้างข้อมูล
# (LLM-Assisted Data Structuring with Ollama)

## สิ่งที่จะได้เรียนรู้
- ตั้งค่า Ollama กับ Qwen2.5
- ออกแบบ prompt สำหรับสร้างคู่คำถาม-คำตอบภาษาไทย
- สร้าง Q&A pairs จาก chunks
- กรองคุณภาพข้อมูล (quality filtering)
- บันทึกเป็น Alpaca format สำหรับ fine-tuning
```

**Cell 2 (Markdown): Prerequisites**

```markdown
## ข้อกำหนดเบื้องต้น

ก่อนเริ่ม ตรวจสอบว่า:
1. ติดตั้ง Ollama แล้ว: `curl -fsSL https://ollama.com/install.sh | sh`
2. ดาวน์โหลดโมเดล: `ollama pull qwen2.5:3b`
3. Ollama กำลังทำงาน: `ollama serve` (ในอีก terminal)
```

**Cell 3 (Code): Verify Ollama connection**

```python
import ollama

# ทดสอบเชื่อมต่อ Ollama
try:
    models = ollama.list()
    print("เชื่อมต่อ Ollama สำเร็จ!")
    print("โมเดลที่มี:")
    for model in models.get("models", []):
        print(f"  - {model['name']}")
except Exception as e:
    print(f"เชื่อมต่อ Ollama ไม่สำเร็จ: {e}")
    print("กรุณาตรวจสอบว่า Ollama กำลังทำงาน (ollama serve)")
```

**Cell 4 (Code): Test basic generation**

```python
# ทดสอบสร้างข้อความภาษาไทย
response = ollama.chat(
    model="qwen2.5:3b",
    messages=[
        {"role": "user", "content": "สวัสดีครับ ช่วยอธิบายว่า AI คืออะไรสั้นๆ เป็นภาษาไทย"}
    ]
)

print("คำตอบจาก Qwen2.5:")
print(response["message"]["content"])
```

**Cell 5 (Code): Design Q&A generation prompt**

```python
QA_PROMPT_TEMPLATE = """คุณเป็นผู้เชี่ยวชาญในการสร้างชุดข้อมูลสำหรับสอนโมเดล AI
จากข้อความด้านล่าง กรุณาสร้างคู่คำถาม-คำตอบ 3 คู่ เป็นภาษาไทย

กฎ:
- คำถามต้องตอบได้จากข้อความที่ให้มาเท่านั้น
- คำตอบต้องถูกต้องและครบถ้วน
- ใช้ภาษาไทยทั้งหมด
- ตอบในรูปแบบ JSON ที่กำหนด

ข้อความ:
{chunk_text}

ตอบเป็น JSON array ดังนี้:
[
  {{"instruction": "คำถามที่ 1", "input": "", "output": "คำตอบที่ 1"}},
  {{"instruction": "คำถามที่ 2", "input": "", "output": "คำตอบที่ 2"}},
  {{"instruction": "คำถามที่ 3", "input": "", "output": "คำตอบที่ 3"}}
]

JSON:"""

# ทดสอบกับข้อความตัวอย่าง
test_chunk = "ปัญญาประดิษฐ์ หรือ AI คือศาสตร์ที่เกี่ยวข้องกับการสร้างเครื่องจักรที่สามารถทำงานที่ต้องใช้สติปัญญาของมนุษย์ เช่น การเรียนรู้ การตัดสินใจ และการแก้ปัญหา"

prompt = QA_PROMPT_TEMPLATE.format(chunk_text=test_chunk)
print("Prompt ที่สร้าง:")
print(prompt[:500])
```

**Cell 6 (Code): Generate Q&A pairs function**

```python
import json
import re

def generate_qa_pairs(chunk_text: str, model: str = "qwen2.5:3b") -> list[dict]:
    """สร้างคู่คำถาม-คำตอบจาก chunk ด้วย Ollama"""
    prompt = QA_PROMPT_TEMPLATE.format(chunk_text=chunk_text)

    response = ollama.chat(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        options={"temperature": 0.3}  # ใช้ temperature ต่ำเพื่อความสม่ำเสมอ
    )

    raw_output = response["message"]["content"]

    # พยายาม parse JSON จากคำตอบ
    try:
        # หา JSON array ในคำตอบ
        json_match = re.search(r'\[.*\]', raw_output, re.DOTALL)
        if json_match:
            qa_pairs = json.loads(json_match.group())
            return qa_pairs
    except json.JSONDecodeError:
        pass

    return []  # คืนค่าว่างถ้า parse ไม่สำเร็จ

# ทดสอบ
test_pairs = generate_qa_pairs(test_chunk)
print(f"สร้างได้ {len(test_pairs)} คู่:")
print(json.dumps(test_pairs, ensure_ascii=False, indent=2))
```

**Cell 7 (Code): Process all chunks**

```python
from pathlib import Path
from tqdm import tqdm
import time

# โหลด chunks จาก Notebook 2
chunks_path = Path("../output/chunks/chunks.jsonl")
chunks = []
with open(chunks_path, "r", encoding="utf-8") as f:
    for line in f:
        chunks.append(json.loads(line))

print(f"โหลด {len(chunks)} chunks")

# สร้าง Q&A pairs สำหรับแต่ละ chunk
all_qa_pairs = []
failed_chunks = 0

for chunk in tqdm(chunks, desc="กำลังสร้าง Q&A"):
    try:
        pairs = generate_qa_pairs(chunk["text"])

        # เพิ่ม metadata ว่ามาจาก chunk ไหน
        for pair in pairs:
            pair["source_chunk_id"] = chunk["chunk_id"]
            pair["source_file"] = chunk["metadata"]["source"]

        all_qa_pairs.extend(pairs)
    except Exception as e:
        failed_chunks += 1
        continue

    time.sleep(0.1)  # ป้องกัน rate limiting

print(f"\nสรุป:")
print(f"  สร้าง Q&A pairs: {len(all_qa_pairs)}")
print(f"  Chunks ที่ล้มเหลว: {failed_chunks}")
```

**Cell 8 (Code): Quality filtering**

```python
from pythainlp.util import isthai

def quality_filter(qa_pairs: list[dict]) -> list[dict]:
    """กรองคุณภาพ Q&A pairs"""
    filtered = []
    reasons = {"too_short": 0, "no_thai": 0, "duplicate": 0, "passed": 0}
    seen_instructions = set()

    for pair in qa_pairs:
        instruction = pair.get("instruction", "")
        output = pair.get("output", "")

        # 1. ตรวจความยาวขั้นต่ำ
        if len(instruction) < 10 or len(output) < 10:
            reasons["too_short"] += 1
            continue

        # 2. ตรวจว่ามีภาษาไทย
        thai_chars = sum(1 for c in instruction + output if '\u0e00' <= c <= '\u0e7f')
        total_chars = len(instruction + output)
        if total_chars > 0 and thai_chars / total_chars < 0.3:
            reasons["no_thai"] += 1
            continue

        # 3. ตรวจซ้ำ
        if instruction in seen_instructions:
            reasons["duplicate"] += 1
            continue
        seen_instructions.add(instruction)

        reasons["passed"] += 1
        filtered.append(pair)

    print("ผลการกรองคุณภาพ:")
    for reason, count in reasons.items():
        print(f"  {reason}: {count}")

    return filtered

# กรองคุณภาพ
filtered_pairs = quality_filter(all_qa_pairs)
print(f"\nเหลือ {len(filtered_pairs)} Q&A pairs หลังกรอง")
```

**Cell 9 (Code): Export to Alpaca JSONL**

```python
# บันทึกเป็น Alpaca format
output_dir = Path("../output/datasets")
output_dir.mkdir(parents=True, exist_ok=True)

sft_path = output_dir / "sft_dataset.jsonl"

# เก็บเฉพาะ fields ที่ต้องการสำหรับ Alpaca format
with open(sft_path, "w", encoding="utf-8") as f:
    for pair in filtered_pairs:
        alpaca_entry = {
            "instruction": pair["instruction"],
            "input": pair.get("input", ""),
            "output": pair["output"],
        }
        f.write(json.dumps(alpaca_entry, ensure_ascii=False) + "\n")

print(f"บันทึก SFT dataset: {sft_path}")
print(f"จำนวน: {len(filtered_pairs)} entries")

# แสดงตัวอย่าง
print("\nตัวอย่าง 2 entries แรก:")
for pair in filtered_pairs[:2]:
    print(json.dumps(
        {"instruction": pair["instruction"], "input": pair.get("input", ""), "output": pair["output"]},
        ensure_ascii=False, indent=2
    ))
    print()
```

---

## Task 4: Notebook 4 — Building Final Datasets

**Files:**
- Create: `notebooks/04_final_datasets.ipynb`

This notebook covers RAG embedding creation, SFT dataset validation, statistics, visualization, and next steps.

**Cell 1 (Markdown): Title**

```markdown
# Notebook 4: การสร้างชุดข้อมูลขั้นสุดท้าย
# (Building Final Datasets)

## สิ่งที่จะได้เรียนรู้
- สร้าง RAG dataset พร้อม embeddings
- ตรวจสอบคุณภาพ SFT dataset
- วิเคราะห์สถิติชุดข้อมูล
- ขั้นตอนถัดไป: การนำไปใช้จริง
```

**Cell 2 (Code): Load data from previous notebooks**

```python
import json
from pathlib import Path

# โหลด SFT dataset
sft_path = Path("../output/datasets/sft_dataset.jsonl")
sft_data = []
with open(sft_path, "r", encoding="utf-8") as f:
    for line in f:
        sft_data.append(json.loads(line))

# โหลด chunks
chunks_path = Path("../output/chunks/chunks.jsonl")
chunks = []
with open(chunks_path, "r", encoding="utf-8") as f:
    for line in f:
        chunks.append(json.loads(line))

print(f"SFT entries: {len(sft_data)}")
print(f"Chunks สำหรับ RAG: {len(chunks)}")
```

**Cell 3 (Markdown): RAG embeddings explanation**

```markdown
## สร้าง Embeddings สำหรับ RAG

**Embedding** คือการแปลงข้อความเป็นเวกเตอร์ตัวเลข เพื่อให้สามารถค้นหาข้อความที่คล้ายกันได้

เราจะใช้โมเดล `intfloat/multilingual-e5-small` ซึ่ง:
- รองรับหลายภาษารวมถึงภาษาไทย
- ขนาดเล็ก ทำงานได้บน CPU
- ให้ผลลัพธ์ดีสำหรับงาน retrieval
```

**Cell 4 (Code): Create embeddings**

```python
from sentence_transformers import SentenceTransformer
from tqdm import tqdm
import numpy as np

# โหลดโมเดล embedding
print("กำลังโหลดโมเดล embedding...")
embed_model = SentenceTransformer("intfloat/multilingual-e5-small")
print("โหลดสำเร็จ!")

# สร้าง embeddings
texts = [chunk["text"] for chunk in chunks]

print(f"กำลังสร้าง embeddings สำหรับ {len(texts)} chunks...")
embeddings = embed_model.encode(
    texts,
    show_progress_bar=True,
    batch_size=32,
    normalize_embeddings=True
)

print(f"ขนาด embedding: {embeddings.shape}")
print(f"มิติ: {embeddings.shape[1]}")
```

**Cell 5 (Code): Build RAG dataset**

```python
# สร้าง RAG dataset
rag_dataset = []

for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
    rag_entry = {
        "chunk_id": chunk["chunk_id"],
        "text": chunk["text"],
        "metadata": chunk["metadata"],
        "embedding": embedding.tolist(),
    }
    rag_dataset.append(rag_entry)

# บันทึก
rag_path = Path("../output/datasets/rag_dataset.jsonl")
with open(rag_path, "w", encoding="utf-8") as f:
    for entry in rag_dataset:
        f.write(json.dumps(entry, ensure_ascii=False) + "\n")

print(f"บันทึก RAG dataset: {rag_path}")
print(f"จำนวน: {len(rag_dataset)} entries")
```

**Cell 6 (Code): Test similarity search**

```python
# ทดสอบค้นหาด้วย embedding (simple similarity search)
def search(query: str, top_k: int = 3) -> list[dict]:
    """ค้นหา chunks ที่เกี่ยวข้องกับ query"""
    query_embedding = embed_model.encode([query], normalize_embeddings=True)

    # คำนวณ cosine similarity
    similarities = np.dot(embeddings, query_embedding.T).flatten()

    # หา top-k
    top_indices = np.argsort(similarities)[::-1][:top_k]

    results = []
    for idx in top_indices:
        results.append({
            "chunk_id": chunks[idx]["chunk_id"],
            "score": float(similarities[idx]),
            "text": chunks[idx]["text"][:200] + "...",
        })
    return results

# ทดสอบค้นหา
query = "ปัญญาประดิษฐ์"  # หรือเปลี่ยนเป็นคำถามที่เกี่ยวข้องกับเอกสารตัวอย่าง
print(f"ค้นหา: '{query}'\n")

results = search(query)
for r in results:
    print(f"[{r['score']:.4f}] {r['chunk_id']}")
    print(f"  {r['text']}")
    print()
```

**Cell 7 (Code): SFT dataset validation**

```python
import pandas as pd

# ตรวจสอบ SFT dataset
df = pd.DataFrame(sft_data)

print("SFT Dataset Summary:")
print(f"  จำนวน entries: {len(df)}")
print(f"  Columns: {list(df.columns)}")
print()

# ตรวจค่าว่าง
print("ค่าว่าง:")
for col in df.columns:
    empty = df[col].apply(lambda x: len(str(x)) == 0).sum()
    print(f"  {col}: {empty} ว่าง")

# ความยาว
print("\nความยาว instruction:")
print(f"  เฉลี่ย: {df['instruction'].str.len().mean():.0f} ตัวอักษร")
print(f"  min: {df['instruction'].str.len().min()}")
print(f"  max: {df['instruction'].str.len().max()}")

print("\nความยาว output:")
print(f"  เฉลี่ย: {df['output'].str.len().mean():.0f} ตัวอักษร")
print(f"  min: {df['output'].str.len().min()}")
print(f"  max: {df['output'].str.len().max()}")
```

**Cell 8 (Code): Visualization**

```python
import matplotlib.pyplot as plt
import matplotlib

# ตั้งค่าให้แสดงภาษาไทย
matplotlib.rcParams['font.family'] = 'DejaVu Sans'

fig, axes = plt.subplots(1, 3, figsize=(15, 4))

# 1. Instruction length distribution
axes[0].hist(df['instruction'].str.len(), bins=20, edgecolor='black', alpha=0.7, color='steelblue')
axes[0].set_xlabel("Length (chars)")
axes[0].set_ylabel("Count")
axes[0].set_title("Instruction Length Distribution")

# 2. Output length distribution
axes[1].hist(df['output'].str.len(), bins=20, edgecolor='black', alpha=0.7, color='coral')
axes[1].set_xlabel("Length (chars)")
axes[1].set_ylabel("Count")
axes[1].set_title("Output Length Distribution")

# 3. RAG chunk length distribution
chunk_lengths = [len(c["text"]) for c in chunks]
axes[2].hist(chunk_lengths, bins=20, edgecolor='black', alpha=0.7, color='mediumseagreen')
axes[2].set_xlabel("Length (chars)")
axes[2].set_ylabel("Count")
axes[2].set_title("RAG Chunk Length Distribution")

plt.tight_layout()
plt.savefig("../output/datasets/dataset_statistics.png", dpi=150)
plt.show()

print("Dataset statistics saved!")
```

**Cell 9 (Markdown): Final summary**

```markdown
## สรุป

### ชุดข้อมูลที่สร้างได้:

| Dataset | Path | Format | จำนวน |
|---|---|---|---|
| SFT (Fine-tuning) | `output/datasets/sft_dataset.jsonl` | Alpaca JSONL | ดูข้างบน |
| RAG (Retrieval) | `output/datasets/rag_dataset.jsonl` | JSONL + embeddings | ดูข้างบน |

### ขั้นตอนถัดไป:

**สำหรับ Fine-tuning:**
- ใช้กับ [Unsloth](https://github.com/unslothai/unsloth) สำหรับ fine-tune Llama/Qwen
- อัปโหลดไป [Hugging Face Hub](https://huggingface.co/docs/datasets/)
- ใช้กับ [TRL](https://github.com/huggingface/trl) สำหรับ SFT training

**สำหรับ RAG:**
- ใช้กับ vector database เช่น ChromaDB, Milvus, Qdrant
- สร้าง retrieval pipeline ด้วย LangChain หรือ LlamaIndex
```

**Cell 10 (Code): Final file listing**

```python
# แสดงไฟล์ทั้งหมดที่สร้าง
output_base = Path("../output")

print("ไฟล์ทั้งหมดที่สร้างจาก Workshop:\n")
for subdir in ["extracted", "chunks", "datasets"]:
    dir_path = output_base / subdir
    if dir_path.exists():
        print(f"  {subdir}/")
        for f in sorted(dir_path.iterdir()):
            size_kb = f.stat().st_size / 1024
            print(f"    {f.name} ({size_kb:.1f} KB)")
        print()

print("Workshop เสร็จสิ้น!")
```

---

## Task 5: Sample Data & Testing

**Files:**
- Create: `data/samples/create_sample_thai_pdf.py` — Script to create a sample Thai PDF for testing
- Create: `README.md` — Workshop instructions in Thai

**Step 1: Create sample data generation script**

```python
"""สร้างไฟล์ PDF ตัวอย่างภาษาไทยสำหรับ Workshop"""
# Uses reportlab or a simpler approach — participants can also bring their own documents
```

**Step 2: Create README with workshop instructions**

The README should include:
- Workshop title and description (Thai)
- Prerequisites (Python, uv, Ollama)
- Setup instructions (`uv sync`, `ollama pull qwen2.5:3b`)
- How to run each notebook in order
- Troubleshooting common issues

**Step 3: Test full pipeline end-to-end**

Run all 4 notebooks sequentially with sample data to verify the complete pipeline works.

```bash
cd /home/moonie/IPST/workshop
uv run jupyter execute notebooks/01_document_extraction.ipynb
uv run jupyter execute notebooks/02_cleaning_and_chunking.ipynb
uv run jupyter execute notebooks/03_llm_structuring.ipynb
uv run jupyter execute notebooks/04_final_datasets.ipynb
```

---

## Summary

| Task | What | Est. Cells |
|---|---|---|
| 0 | Project setup (uv, dirs) | — |
| 1 | Notebook 1: Docling extraction | 12 cells |
| 2 | Notebook 2: Cleaning & chunking | 11 cells |
| 3 | Notebook 3: Ollama Q&A generation | 9 cells |
| 4 | Notebook 4: Final datasets | 10 cells |
| 5 | Sample data, README, E2E test | — |

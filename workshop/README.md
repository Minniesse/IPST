# Workshop: การเตรียมและจัดโครงสร้างข้อมูลสำหรับสอนโมเดล

Workshop สำหรับเรียนรู้การดึงข้อมูลจากเอกสาร PDF/PPTX ภาษาไทย และจัดโครงสร้างเป็นชุดข้อมูลสำหรับ Fine-tuning (SFT) และ RAG

## สิ่งที่ต้องเตรียม

### ซอฟต์แวร์

- **Python 3.10+**
- **uv** (package manager): `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Ollama**: `curl -fsSL https://ollama.com/install.sh | sh`

### ดาวน์โหลดโมเดล

```bash
ollama pull qwen2.5:3b
```

## การติดตั้ง

```bash
# Clone หรือดาวน์โหลดโปรเจค
cd workshop

# ติดตั้ง dependencies
uv sync

# สร้างไฟล์ตัวอย่าง
uv run python data/samples/create_sample_data.py
```

## การใช้งาน

เปิด Jupyter Notebook และรันทีละ Notebook ตามลำดับ:

```bash
# เริ่ม Jupyter
uv run jupyter notebook notebooks/
```

### ลำดับ Notebooks

| # | Notebook | เวลา | เนื้อหา |
|---|---|---|---|
| 1 | `01_document_extraction.ipynb` | 45 นาที | ดึงข้อมูลจาก PDF/PPTX ด้วย Docling |
| 2 | `02_cleaning_and_chunking.ipynb` | 45 นาที | ทำความสะอาดและแบ่งชิ้นส่วนข้อมูล |
| 3 | `03_llm_structuring.ipynb` | 60 นาที | ใช้ LLM สร้าง Q&A pairs |
| 4 | `04_final_datasets.ipynb` | 30 นาที | สร้างชุดข้อมูล SFT + RAG |

**สำคัญ:** ต้องรัน Ollama ก่อนเริ่ม Notebook 3:
```bash
ollama serve  # รันในอีก terminal
```

## โครงสร้างโปรเจค

```
workshop/
├── notebooks/
│   ├── 01_document_extraction.ipynb
│   ├── 02_cleaning_and_chunking.ipynb
│   ├── 03_llm_structuring.ipynb
│   └── 04_final_datasets.ipynb
├── data/
│   └── samples/          # ไฟล์ตัวอย่างภาษาไทย
├── output/
│   ├── extracted/         # ผลลัพธ์จาก Notebook 1
│   ├── chunks/            # ผลลัพธ์จาก Notebook 2
│   └── datasets/          # ชุดข้อมูลขั้นสุดท้าย
├── pyproject.toml
└── README.md
```

## ผลลัพธ์ที่ได้

- **SFT Dataset** (`output/datasets/sft_dataset.jsonl`) — Alpaca format สำหรับ fine-tuning
- **RAG Dataset** (`output/datasets/rag_dataset.jsonl`) — Chunks + embeddings สำหรับ retrieval

## แก้ปัญหาที่พบบ่อย

### Ollama เชื่อมต่อไม่ได้
```bash
# ตรวจสอบว่า Ollama ทำงานอยู่
ollama list

# ถ้าไม่ทำงาน ให้เริ่มใหม่
ollama serve
```

### หน่วยความจำไม่พอสำหรับ Qwen2.5:7b
ใช้โมเดลขนาดเล็กกว่า:
```bash
ollama pull qwen2.5:3b
```
แล้วแก้ `model="qwen2.5:3b"` ใน Notebook 3

### ข้อความภาษาไทยแสดงผิดปกติ
ตรวจสอบว่าไฟล์บันทึกเป็น UTF-8 และติดตั้ง pythainlp:
```bash
uv add pythainlp
```

## เครื่องมือที่ใช้

- [Docling](https://github.com/docling-project/docling) — Document extraction
- [Ollama](https://ollama.com/) — Local LLM inference
- [PyThaiNLP](https://github.com/PyThaiNLP/pythainlp) — Thai NLP toolkit
- [Sentence Transformers](https://www.sbert.net/) — Text embeddings

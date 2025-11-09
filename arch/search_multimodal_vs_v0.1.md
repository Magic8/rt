# search_multimodal_vs_v0.1.md
> **Multimodal Search Architecture (Text + Images + Spatial Map)**  
> For the *RealTalk HOA Agent* — preserve legal fidelity while enabling spatial Q&A (e.g., “nearest fire exit to Unit 203”, “where is the gym?”, “show the floor plan”).

---

## 0) Principles & Guardrails

- **Archive ≠ Search Format.** The authoritative record is a **searchable “sandwich” PDF** (image visible, OCR text layer beneath). The searchable corpus for the LLM is **Markdown + metadata**, with **images and a spatial JSON layer** referenced by ID.
- **Safety first.** Never infer life‑safety answers (exits/egress) from ad‑hoc vision at query time. Use **verified Layer‑C annotations**. Mark estimates (e.g., occupancy) clearly with code references/assumptions.
- **Determinism & audit.** Every result carries **page, exhibit, file_id, and/or geometry IDs**. Answers cite governing text and link to the underlying image/PDF.
- **Token & latency discipline.** Clean Markdown chunks (by heading) minimize tokens and latency; images are retrieved via metadata, not pixels in vectors.

---

## 1) Layers Overview

| Layer | Function | Input | Stored As | Indexed For |
|------|----------|-------|-----------|-------------|
| **A – Text Corpus** | Legal text, policies, minutes | OCR’d PDF → Markdown | `.md` chunks | Semantic text search |
| **B – Image Corpus** | Floor plans, diagrams, exhibits | PDF images → PNG/TIFF | image + caption | Caption/text search |
| **C – Spatial Map** | Coordinates, exits, units, amenities | Manual/verified JSON | `.json` | Geometric queries & routing |

---

## 2) OCR → Markdown (Layer A)

### 2.1 Export Options (ABBYY FineReader)
- **Layout:** Text under the page image  
- **Quality:** High quality  
- **Options:** Enable PDF tagging  
- Keep pictures; do not strip headers/footers in archive (strip later in text cleanup).

### 2.2 Headless Alternative (free)
```bash
# Build a “sandwich” searchable PDF
ocrmypdf --language eng --output-type pdfa --pdf-renderer sandwich \
  "428Alice.pdf" "428Alice_Searchable.pdf"

# Extract text and convert to Markdown
pdftotext -layout "428Alice_Searchable.pdf" - | \
  pandoc -f markdown -t markdown_strict --wrap=none -o "428Alice.md"
```

### 2.3 Markdown Cleanup Rules
- Promote headings via regex:
  - `^ARTICLE\s+[IVXLC]+` → `# ARTICLE …`
  - `^\d+\.\d+\s+` → `## …`
  - `^\d+\.\d+\.\d+\s+` → `### …`
- Strip repeating headers/footers/page numbers.
- Normalize hyphenation and whitespace.
- Add YAML per section (example below).

---

## 3) Layer A — Markdown Schema & Example

**Schema + Body example**
```markdown
---
doc_id: "428Alice_CC&Rs"
source: "CC&Rs (Recorded 2006)"
article: "6"
section: "6.1(d)"
title: "No Cumulative Voting"
pages: "72"
file_id_master_pdf: "file-xxxxxxxx"  # OpenAI File Store id of searchable PDF
doc_type: "law"
---
## 6.1(d) No Cumulative Voting
Cumulative voting is not permitted. Each Member is entitled to cast one (1) vote per vacancy to be filled but may not allocate multiple votes to a single candidate.
```

**Chunking strategy**
- Split at headings (Article/Section/Subsection).
- Aim ~0.5–1.0k tokens per chunk.

**Python splitter (sketch)**
```python
import re

def split_md_into_chunks(md_text):
    parts = re.split(r'(?m)^(#{1,3})\s+', md_text)
    chunks, i = [], 1
    while i < len(parts):
        level = len(parts[i]); title = parts[i+1].strip()
        body = parts[i+2] if i+2 < len(parts) else ""
        meta = {"heading": title, "level": level}
        chunks.append({"meta": meta, "body": f"## {title}\n{body}"})
        i += 3
    return chunks
```

---

## 4) Layer B — Image Corpus (Floor Plans & Diagrams)

### 4.1 Extract Images
```bash
pdfimages -p -png "428Alice_Searchable.pdf" ./floorplans/
# -> floorplans/p097_001.png, floorplans/p098_001.png, ...
```

### 4.2 Captions (deterministic, reviewed)
Examples:
```
Building 1 – Level 1 – Units 101–106 – Elevator E1 – Stairs S1/S2 – Courtyard
Building 2 – Level 2 – Units 201–206 – Elevator E1 – Stairs S1/S2
```

### 4.3 Image Metadata
```json
{
  "doc_id": "428Alice_CC&Rs_ExhibitD",
  "page": 97,
  "image_path": "floorplans/p097_001.png",
  "building": "B1",
  "level": "L1",
  "caption": "Building 1 – Level 1 – Units 101–106 – Elevator E1 – Stairs S1/S2 – Courtyard",
  "features": ["units:101-106","elevator:E1","stairs:S1,S2","area:courtyard"]
}
```

### 4.4 OpenAI‑only API (File Store + Vector Store)
```python
from openai import OpenAI
client = OpenAI()

# Upload image to File Store
with open("floorplans/p097_001.png", "rb") as f:
    file = client.files.create(file=f, purpose="user_data")
file_id = file.id  # e.g., 'file_abc123'

# Create an empty Vector Store
vs = client.vector_stores.create(name="hoa-floorplans")
vs_id = vs.id

# Link the file to the vector store (provenance; not searchable by itself)
client.vector_stores.files.create(vector_store_id=vs_id, file_id=file_id)

# Make it searchable by embedding the CAPTION text
caption = "Building 1 – Level 1 – Units 101–106 – Elevator E1 – Stairs S1/S2 – Courtyard (p.97)"
emb = client.embeddings.create(model="text-embedding-3-large", input=caption).data[0].embedding

# Upsert into vector store (embedding + metadata)
client.vector_stores.add_items(
    vector_store_id=vs_id,
    items=[{
        "id": "floorplan_p097",
        "embedding": emb,
        "metadata": {
            "file_id": file_id,
            "page": 97,
            "building": "B1",
            "level": "L1",
            "caption": caption
        }
    }]
)
```

**Key:** Vector store keeps **embeddings + metadata** only; image bytes live in **File Store**.

---

## 5) Layer C — Spatial Map (Structured JSON)

### 5.1 Minimal Schema
```json
{
  "doc_id": "428Alice_CC&Rs_ExhibitD",
  "page": 97,
  "image_path": "floorplans/p097_001.png",
  "scale": {"pixels_per_foot": 3.2, "source": "title block 1/8\"=1'-0\""},
  "units": [
    {"unit": "101", "centroid": [12.4, 38.1]},
    {"unit": "102", "centroid": [22.7, 35.0]}
  ],
  "amenities": [
    {"id":"GYM","name":"Gym","type":"amenity","centroid":[48.2,12.5]},
    {"id":"COURTYARD","name":"Courtyard","type":"area","polygon":[[30,20],[60,20],[60,45],[30,45]]}
  ],
  "egress": [
    {"id":"EXIT_A","name":"Exit A","type":"exit","centroid":[5.0,10.0]},
    {"id":"EXIT_B","name":"Exit B","type":"exit","centroid":[55.0,50.0]}
  ],
  "vertical": [
    {"id":"S1","name":"Stairs S1","type":"stairs","centroid":[18.0,12.0]},
    {"id":"E1","name":"Elevator E1","type":"elevator","centroid":[40.0,18.0]}
  ]
}
```

### 5.2 Spatial Computation (examples)
```python
import math

def dist(a,b):
    return math.hypot(a[0]-b[0], a[1]-b[1])

def nearest_exit(unit_xy, egress):
    best = min(egress, key=lambda e: dist(unit_xy, e["centroid"]))
    return best, dist(unit_xy, best["centroid"])
```

**Notes**
- Store Layer‑C JSON in DB and/or File Store; reference via metadata (e.g., `geom_id`) from the vector store item.
- For routing (optional), add a corridor graph (nodes/edges) and compute shortest path.

---

## 6) Retrieval Orchestration

### 6.1 Query Routing
| Query Type | Layer | Logic |
|-----------|------|------|
| “nearest exit” | C | Geometry/nearest or path search |
| “how many people fit the courtyard” | C + A | Area/scale × IBC factor (estimate) |
| “show plan for unit 103” | B | Caption/metadata match + image display |
| “what is cumulative voting” | A | Text search + cite section |

### 6.2 Response Assembly
1. Retrieve vectors → top‑k (text or caption).  
2. Fetch asset via `file_id` (image/PDF).  
3. Compose: answer text + citation (page/exhibit) + image display; include safety disclaimer for egress/occupancy.

---

## 7) QA & Risk Checklist
- OCR completeness verified (esp. election rules and “No Cumulative Voting”).  
- Image fidelity: no down‑rez; PNG/TIFF for linework.  
- Captions deterministic & reviewed.  
- Geometry & scale verified; record scale source (title block).  
- Life‑safety answers: include disclaimer; posted signage takes precedence.  
- Occupancy: label as **estimate** unless stamped load present.  
- Traceability: every answer links page/exhibit and underlying file.

---

## 8) Optional Automation for Plans
- **Symbol detection:** OpenCV template matching for EXIT/STAIR/ELEV icons; or fine‑tune a small YOLO model on your icons. **Always verify** annotations.  
- **Auto‑captioning:** One‑time multimodal draft → human review → store captions.  
- **Area extraction:** Use polygon tools to trace courtyard/amenity areas; store as Layer‑C polygons.

---

## 9) Storage Layout (recommended)

```
/corpus/
  428Alice_Searchable.pdf          # authoritative archive (File Store: file_xxx)
  428Alice.md                      # Layer A text
  /floorplans/                     # Layer B images
    p097_001.png
    p098_001.png
  /geom/                           # Layer C JSON
    geom_p097.json
    geom_p098.json
```

Vector store: **embeddings + metadata** referencing assets (`file_id`, page, building, level, geom_id).  
File Store: **binary assets** (PDFs, PNGs, JSON).

---

## 10) Why this tri‑layer design
- **Markdown only** drops spatial answers.  
- **Images only** are not language‑searchable.  
- **Spatial JSON + captions + Markdown** → precise, auditable, low‑latency multimodal answers.


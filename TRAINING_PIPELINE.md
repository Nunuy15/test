# Training Pipeline - Chatbot AIMS (Qwen3 Fine-Tuning)

Dokumentasi lengkap proses fine-tuning model Qwen3 untuk Chatbot AIMS
(Sistem Informasi Cuaca & Kebencanaan Indonesia).

---

## Daftar Isi

1. [Arsitektur Umum](#1-arsitektur-umum)
2. [Alur Data Training](#2-alur-data-training)
3. [Sumber Data Training (6 Sumber)](#3-sumber-data-training-6-sumber)
4. [Pipeline Steps (run_training.sh)](#4-pipeline-steps)
5. [Detail Setiap Script](#5-detail-setiap-script)
6. [Tabel Database yang Dicakup (27 Tabel)](#6-tabel-database-yang-dicakup)
7. [Konfigurasi & Mode](#7-konfigurasi--mode)
8. [Cara Menjalankan](#8-cara-menjalankan)

---

## 1. Arsitektur Umum

```mermaid
graph TB
    subgraph INPUT["Sumber Data"]
        A1[("PostgreSQL<br/>Database AIMS")]
        A2["skemadatabase.txt<br/>(94 tabel)"]
        A3["populate_chromadb.py<br/>TABLE_TRAINING_DATA<br/>(27 tabel)"]
        A4["Google Custom<br/>Search API"]
        A5["cb_conversation<br/>(riwayat chat)"]
        A6["Domain Knowledge<br/>(hardcoded Q&A)"]
    end

    subgraph PREP["Data Preparation"]
        B1["scrape_web_knowledge.py<br/>Web Scraping + Ollama Q&A"]
        B2["prepare_training_data.py<br/>Merge 6 sumber data"]
    end

    subgraph TRAINING["Training"]
        C1["train_qwen3.py<br/>QLoRA Fine-Tuning"]
    end

    subgraph EXPORT["Export & Deploy"]
        D1["export_to_ollama.py<br/>Merge + GGUF + Register"]
        D2["evaluate_model.py<br/>Perbandingan kualitas"]
        D3[".env update<br/>Deploy ke production"]
    end

    subgraph OUTPUT["Output"]
        E1["Ollama Model<br/>qwen3-aims-finetuned"]
        E2["evaluation/<br/>comparison_report.json"]
    end

    A4 --> B1
    B1 -->|web_knowledge.jsonl| B2
    A2 --> B2
    A3 --> B2
    A5 --> B2
    A6 --> B2
    A1 -->|lokasi augmentasi| B2
    B2 -->|train.jsonl + val.jsonl| C1
    C1 -->|LoRA adapter| D1
    D1 --> E1
    E1 --> D2
    D2 --> E2
    D1 --> D3

    style INPUT fill:#e1f5fe
    style PREP fill:#fff3e0
    style TRAINING fill:#fce4ec
    style EXPORT fill:#e8f5e9
    style OUTPUT fill:#f3e5f5
```

---

## 2. Alur Data Training

```mermaid
flowchart LR
    subgraph sources["6 Sumber Data"]
        direction TB
        S1["1. Text-to-SQL<br/>TABLE_TRAINING_DATA<br/>(195 examples)"]
        S2["2. SQL-to-Response<br/>Format contoh<br/>(18 examples)"]
        S3["3. Domain Q&A<br/>Hardcoded<br/>(18 examples)"]
        S4["4. Web Knowledge<br/>Google Search<br/>(~430 examples)"]
        S5["5. Conversation History<br/>PostgreSQL<br/>(variabel)"]
        S6["6. Augmentasi<br/>Variasi lokasi/bahasa<br/>(variabel)"]
    end

    subgraph merge["Merge & Split"]
        M1["Gabung semua<br/>examples"]
        M2["Shuffle<br/>(seed=42)"]
        M3["Split 90/10"]
    end

    subgraph output["Output Files"]
        O1["data/train.jsonl<br/>(90%)"]
        O2["data/val.jsonl<br/>(10%)"]
        O3["data/metadata.json"]
    end

    S1 --> M1
    S2 --> M1
    S3 --> M1
    S4 --> M1
    S5 --> M1
    S6 --> M1
    M1 --> M2 --> M3
    M3 --> O1
    M3 --> O2
    M3 --> O3
```

---

## 3. Sumber Data Training (6 Sumber)

### 3.1 Text-to-SQL Examples

```mermaid
flowchart TD
    A["TABLE_TRAINING_DATA<br/>(populate_chromadb.py)<br/>27 tabel"] --> B["generate_text_to_sql_examples()"]
    C["SchemaParser<br/>(skemadatabase.txt)<br/>94 tabel"] --> B
    B --> D["Untuk setiap tabel:"]
    D --> E["Ambil example_questions[]<br/>& example_sql[]"]
    E --> F["Pasangkan question → SQL<br/>berdasarkan keyword matching"]
    F --> G["Buat chat message:<br/>system + user + assistant"]

    G --> H["Output: ~195 examples<br/>type=text_to_sql"]

    style H fill:#c8e6c9
```

**Format output:**
```json
{
  "messages": [
    {"role": "system", "content": "Kamu adalah asisten AI ... [schema tabel]"},
    {"role": "user", "content": "Ada berapa hotspot di Kalimantan hari ini?"},
    {"role": "assistant", "content": "SELECT date_hotspot, nama_provinsi ... FROM indo_hotspot ..."}
  ],
  "type": "text_to_sql",
  "table": "indo_hotspot"
}
```

### 3.2 SQL-to-Response Format Examples

```mermaid
flowchart TD
    A["generate_format_response_examples()"] --> B["Mock data per tabel"]
    B --> C["Contoh: pertanyaan + hasil SQL + jawaban terformat"]
    C --> D["16 jenis format:<br/>cuaca, gempa, berita, bencana,<br/>faskes, RS, TNI, hotspot,<br/>gunung api, prediksi banjir,<br/>ketinggian air, penduduk,<br/>SAR, SITABA, dll"]
    D --> E["Output: 18 examples<br/>type=sql_to_response"]

    style E fill:#c8e6c9
```

**Format output:**
```json
{
  "messages": [
    {"role": "system", "content": "Kamu adalah asisten AI. Format jawaban dari hasil query..."},
    {"role": "user", "content": "Pertanyaan: ...\n\nHasil Query:\n[{...}]"},
    {"role": "assistant", "content": "Berikut data ...\n\n**1. ...**\n- Detail: ..."}
  ],
  "type": "sql_to_response",
  "table": "bmkg_info_cuaca"
}
```

### 3.3 Domain Q&A (Hardcoded)

```mermaid
flowchart TD
    A["generate_domain_qa_examples()"] --> B["18 pasang Q&A manual"]
    B --> C["Topik:<br/>- Apa itu BMKG/BNPB/BPBD/SITABA<br/>- Jenis bencana Indonesia<br/>- Prosedur saat banjir/longsor/gempa/tsunami/kebakaran/cuaca ekstrem<br/>- TNI KODIM KORAMIL KOREM<br/>- Musim hujan di Indonesia<br/>- Isi tas siaga bencana<br/>- Nomor darurat penting"]
    C --> D["Output: 18 examples<br/>type=domain_qa"]

    style D fill:#c8e6c9
```

### 3.4 Web Knowledge (Google Search + Ollama)

```mermaid
flowchart TD
    A["GOOGLE_API_KEY +<br/>GOOGLE_CSE_ID"] --> B["Google Custom Search API<br/>(66 queries, 17 kategori)"]
    B --> C["~330 URL hasil pencarian"]
    C --> D["fetch_and_extract()<br/>trafilatura / BeautifulSoup"]
    D --> E["~180 artikel<br/>(min 200 karakter)"]
    E --> F["generate_qa_pairs()<br/>Ollama LLM"]
    F --> G["3 Q&A per artikel"]
    G --> H["save_qa_pairs()<br/>Deduplicated by MD5"]
    H --> I["data/web_knowledge.jsonl<br/>(~430 Q&A pairs)"]

    J["SearchLog<br/>(idempotency)"] -.->|track queries & URLs| B
    J -.->|track URLs| D
    K["Rate Limit<br/>100 queries/hari"] -.->|enforce| B

    style I fill:#c8e6c9
```

**17 Kategori Pencarian:**

| No | Kategori | Queries | Contoh |
|----|----------|---------|--------|
| 1 | cuaca | 7 | prakiraan cuaca BMKG, El Nino La Nina |
| 2 | gempa | 7 | gempa bumi BMKG, Ring of Fire |
| 3 | bencana | 6 | BNPB fungsi, mitigasi bencana |
| 4 | banjir | 4 | prosedur evakuasi banjir |
| 5 | longsor | 4 | tanda-tanda longsor |
| 6 | kebakaran | 4 | karhutla, kabut asap |
| 7 | tsunami | 4 | peringatan dini tsunami |
| 8 | tni_response | 3 | TNI bantuan bencana |
| 9 | fasilitas_kesehatan | 4 | puskesmas, RS rujukan |
| 10 | kesiapsiagaan | 4 | tas siaga, nomor darurat |
| 11 | gunung_api | 4 | PVMBG, MAGMA Indonesia |
| 12 | hotspot_karhutla | 3 | satelit MODIS VIIRS |
| 13 | prediksi_bencana | 3 | early warning, ketinggian air |
| 14 | sar_basarnas | 2 | BASARNAS, prosedur SAR |
| 15 | kesehatan_bencana | 2 | penyakit pasca bencana |
| 16 | kependudukan | 2 | BPS sensus penduduk |
| 17 | general | 3 | geografi, klimatologi |

### 3.5 Conversation History

```mermaid
flowchart TD
    A["PostgreSQL<br/>cb_conversation"] --> B["extract_conversation_history()"]
    B --> C["Filter: hanya yang<br/>mengandung SQL query"]
    C --> D["Buat pasangan:<br/>user question → SQL response"]
    D --> E["Output: variabel<br/>type=conversation"]

    style E fill:#c8e6c9
```

### 3.6 Augmentasi (Variasi Lokasi & Bahasa)

```mermaid
flowchart TD
    A["Text-to-SQL examples"] --> B["augment_text_to_sql()"]
    C["Lokasi dari DB<br/>(wilayah_kabupaten_kota +<br/>wilayah_kecamatan)"] --> B
    D["Fallback: 50 kota<br/>hardcoded"] --> B
    B --> E["Untuk setiap example:"]
    E --> F["Ganti nama kota/provinsi<br/>di pertanyaan & SQL"]
    E --> G["Variasi bahasa informal:<br/>'gimana', 'kek apa',<br/>'kayak gimana'"]
    F --> H["Output: variabel<br/>augmented=True"]
    G --> H

    style H fill:#c8e6c9
```

---

## 4. Pipeline Steps

### Full Training (6 Steps)

```mermaid
flowchart TD
    S1["Step 1/6<br/>Setup Environment"]
    S2["Step 2/6<br/>Prepare Training Data"]
    S3["Step 3/6<br/>Train Model (QLoRA)"]
    S4["Step 4/6<br/>Export ke Ollama"]
    S5["Step 5/6<br/>Evaluate Model"]
    S6["Step 6/6<br/>Deploy"]

    S1 --> S2 --> S3 --> S4 --> S5 --> S6

    S1 -.- S1D["- Cek hardware (NVIDIA/Apple/CPU)<br/>- Buat venv_training/<br/>- Install requirements-training.txt<br/>- Verifikasi PyTorch + CUDA"]

    S2 -.- S2D["- Jalankan scrape_web_knowledge.py<br/>  (jika GOOGLE_API_KEY ada)<br/>- Jalankan prepare_training_data.py<br/>- Output: data/train.jsonl + val.jsonl"]

    S3 -.- S3D["- Load base model (Qwen3-14B)<br/>- 4-bit quantization (BitsAndBytes)<br/>- LoRA adapter (rank=64, alpha=128)<br/>- SFTTrainer (3 epoch)<br/>- Output: output/qwen3-finetuned/final/"]

    S4 -.- S4D["- Clone llama.cpp (jika belum ada)<br/>- Merge LoRA adapter + base model<br/>- Convert ke GGUF<br/>- Quantize (Q4_K_M)<br/>- Register di Ollama"]

    S5 -.- S5D["- Test model asli vs fine-tuned<br/>- SQL generation accuracy<br/>- Response quality<br/>- Domain knowledge<br/>- Output: evaluation/comparison_report.json"]

    S6 -.- S6D["- Update .env:<br/>  OLLAMA_MODEL=qwen3-aims-finetuned<br/>- Restart Flask app"]

    style S1 fill:#bbdefb
    style S2 fill:#ffe0b2
    style S3 fill:#f8bbd0
    style S4 fill:#c8e6c9
    style S5 fill:#d1c4e9
    style S6 fill:#b2dfdb
```

### Local Test (3 Steps - MacBook)

```mermaid
flowchart LR
    S1["Step 1/3<br/>Setup<br/>(tanpa bitsandbytes)"] --> S2["Step 2/3<br/>Prepare Data<br/>(--no-db default)"] --> S3["Step 3/3<br/>Train Test<br/>(Qwen3-0.6B, 5 steps)"]

    style S1 fill:#bbdefb
    style S2 fill:#ffe0b2
    style S3 fill:#f8bbd0
```

---

## 5. Detail Setiap Script

### 5.1 `scripts/prepare_training_data.py`

Orchestrator utama yang menggabungkan 6 sumber data.

```mermaid
flowchart TB
    subgraph prepare["prepare_training_data.py"]
        direction TB
        P1["[1/6] SchemaParser<br/>Parse skemadatabase.txt → 94 tabel"]
        P2["[2/6] generate_text_to_sql_examples()<br/>TABLE_TRAINING_DATA → 195 examples"]
        P3["[3/6] generate_format_response_examples()<br/>Mock data → 18 examples"]
        P4["[4/6] generate_domain_qa_examples()<br/>Hardcoded → 18 examples"]
        P5["[5/6] load_web_knowledge_examples()<br/>web_knowledge.jsonl → ~430 examples"]
        P6["[6/6] extract_conversation_history()<br/>PostgreSQL → variabel"]

        P1 --> P2 --> P3 --> P4 --> P5 --> P6

        P6 --> AUG["Augmentasi lokasi<br/>(opsional)"]
        AUG --> MERGE["Merge + Shuffle + Split 90/10"]
        MERGE --> OUT1["data/train.jsonl"]
        MERGE --> OUT2["data/val.jsonl"]
        MERGE --> OUT3["data/metadata.json"]
    end

    style OUT1 fill:#c8e6c9
    style OUT2 fill:#c8e6c9
    style OUT3 fill:#c8e6c9
```

**Fungsi-fungsi utama:**

| Fungsi | Baris | Deskripsi |
|--------|-------|-----------|
| `_load_training_data()` | 40 | Load TABLE_TRAINING_DATA dari populate_chromadb.py via AST parsing |
| `load_locations_from_db()` | 168 | Ambil lokasi dari wilayah_kabupaten_kota + wilayah_kecamatan |
| `SchemaParser` | 252 | Parse skemadatabase.txt, generate condensed schema per tabel |
| `build_system_prompt_sql()` | 445 | System prompt untuk Text-to-SQL |
| `build_system_prompt_format()` | 464 | System prompt untuk SQL-to-Response |
| `build_system_prompt_domain()` | 478 | System prompt untuk Domain Q&A |
| `generate_text_to_sql_examples()` | 484 | Sumber 1: Text → SQL |
| `generate_format_response_examples()` | 566 | Sumber 2: SQL result → formatted response |
| `generate_domain_qa_examples()` | 1179 | Sumber 3: Domain knowledge Q&A |
| `load_web_knowledge_examples()` | 1260 | Sumber 4: Web knowledge dari JSONL |
| `extract_conversation_history()` | 1321 | Sumber 5: Riwayat percakapan |
| `augment_text_to_sql()` | 1414 | Sumber 6: Augmentasi variasi |
| `build_dataset()` | 1574 | Main orchestrator |

### 5.2 `scripts/scrape_web_knowledge.py`

Web scraper yang mengumpulkan knowledge dari internet.

```mermaid
flowchart TD
    START["main()"] --> CHECK{"GOOGLE_API_KEY<br/>tersedia?"}
    CHECK -->|Tidak| ERROR["Error: setup .env dulu"]
    CHECK -->|Ya| LOOP["Untuk setiap topic & query:"]

    LOOP --> IDEM{"Query sudah<br/>selesai?<br/>(SearchLog)"}
    IDEM -->|Ya| SKIP1["[skip] Already done"]
    IDEM -->|Tidak| LIMIT{"Daily limit<br/>tercapai?"}
    LIMIT -->|Ya| STOP["[STOP] Run lagi besok"]
    LIMIT -->|Tidak| SEARCH["google_search()<br/>5 results per query"]

    SEARCH --> FETCH["Untuk setiap URL:"]
    FETCH --> URL_CHECK{"URL sudah<br/>di-fetch?"}
    URL_CHECK -->|Ya| SKIP2["skip"]
    URL_CHECK -->|Tidak| EXTRACT["fetch_and_extract()<br/>trafilatura / BeautifulSoup"]
    EXTRACT --> CONTENT{"Konten > 200<br/>karakter?"}
    CONTENT -->|Tidak| SKIP3["[skip] No content"]
    CONTENT -->|Ya| GENERATE["generate_qa_pairs()<br/>Ollama LLM → 3 Q&A"]
    GENERATE --> SAVE["save_qa_pairs()<br/>Deduplicated append"]
    SAVE --> JSONL[("data/web_knowledge.jsonl")]

    SKIP1 --> LOOP
    SKIP2 --> FETCH
    SKIP3 --> FETCH

    style JSONL fill:#c8e6c9
    style ERROR fill:#ffcdd2
    style STOP fill:#fff9c4
```

**Komponen utama:**

| Komponen | Deskripsi |
|----------|-----------|
| `SearchLog` | Idempotency tracker - simpan query & URL yang sudah diproses |
| `google_search()` | Google Custom Search API (lr=lang_id, gl=id) |
| `fetch_and_extract()` | Extract konten artikel (trafilatura primary, BeautifulSoup fallback) |
| `generate_qa_pairs()` | Ollama LLM generate 3 Q&A per artikel |
| `save_qa_pairs()` | Append ke JSONL dengan dedup by MD5 hash |

### 5.3 `scripts/train_qwen3.py`

Fine-tuning model dengan QLoRA.

```mermaid
flowchart TD
    A["Load train.jsonl + val.jsonl"] --> B["Detect Device<br/>(CUDA / MPS / CPU)"]
    B --> C["Load Base Model<br/>(Qwen3-14B)"]
    C --> D["4-bit Quantization<br/>(BitsAndBytes NF4)"]
    D --> E["Pasang LoRA Adapter<br/>(rank=64, alpha=128)"]
    E --> F["Format Dataset<br/>(chat template)"]
    F --> G["SFTTrainer<br/>(Supervised Fine-Tuning)"]
    G --> H["Training Loop<br/>(3 epochs)"]
    H --> I["Save Adapter<br/>output/qwen3-finetuned/final/"]

    style I fill:#c8e6c9
```

**Parameter default vs H100:**

| Parameter | Default | H100 | Local Test |
|-----------|---------|------|------------|
| Model | Qwen3-14B | Qwen3-14B | Qwen3-0.6B |
| Batch size | 4 | 8 | 1 |
| LoRA rank | 64 | 128 | 8 |
| LoRA alpha | 128 | 256 | 16 |
| Epochs | 3 | 3 | 1 |
| Max seq len | 2048 | 2048 | 512 |
| Max steps | - | - | 5 |
| TF32 | off | on | off |

### 5.4 `scripts/export_to_ollama.py`

Konversi model ke format Ollama.

```mermaid
flowchart LR
    A["LoRA Adapter<br/>(output/final/)"] --> B["Merge dengan<br/>Base Model"]
    B --> C["Convert ke<br/>GGUF format<br/>(llama.cpp)"]
    C --> D["Quantize<br/>(Q4_K_M)"]
    D --> E["Buat<br/>Modelfile"]
    E --> F["ollama create<br/>qwen3-aims-finetuned"]

    style F fill:#c8e6c9
```

### 5.5 `scripts/evaluate_model.py`

Evaluasi perbandingan model asli vs fine-tuned.

```mermaid
flowchart LR
    A["val.jsonl<br/>(test data)"] --> B["Test pada<br/>Model Asli"]
    A --> C["Test pada<br/>Model Fine-tuned"]
    B --> D["Bandingkan:<br/>- SQL Accuracy<br/>- Response Quality<br/>- Domain Knowledge"]
    C --> D
    D --> E["evaluation/<br/>comparison_report.json"]

    style E fill:#c8e6c9
```

---

## 6. Tabel Database yang Dicakup

### 27 Tabel dalam TABLE_TRAINING_DATA

```mermaid
graph TB
    subgraph cuaca["Cuaca & Klimatologi"]
        T1["bmkg_info_cuaca<br/>(prakiraan cuaca)"]
    end

    subgraph gempa["Gempa & Seismologi"]
        T2["bmkg_info_gempa<br/>(data gempa)"]
    end

    subgraph bencana["Laporan Bencana"]
        T4["laporan_bencana_harian<br/>(laporan harian)"]
        T5["laporan_bencana_khusus<br/>(laporan detail)"]
        T7["master_kejadian_bencana<br/>(agregat kejadian)"]
        T8["sitaba_bencana<br/>(SITABA records)"]
        T6["bmkg_info_bencana<br/>(peringatan dini)"]
    end

    subgraph gunung["Vulkanologi"]
        T12["indo_hotspot<br/>(titik panas/karhutla)"]
        T13["magma_letusan_gunung<br/>(erupsi)"]
        T14["master_gunung_api<br/>(127 gunung api)"]
    end

    subgraph prediksi["Prediksi & Monitoring"]
        T15["prediksi_banjir<br/>(prediksi per kecamatan)"]
        T16["potensi_banjir<br/>(klasifikasi risiko)"]
        T17["potensi_kebakaran<br/>(klasifikasi risiko)"]
        T24["sitaba_ketinggian_air<br/>(water level)"]
        T22["master_pergerakan_tanah<br/>(ground movement)"]
    end

    subgraph berita["Berita & Informasi"]
        T3["articles<br/>(berita kebencanaan)"]
        T18["bmkg_news<br/>(berita BMKG)"]
        T19["bnpb_news<br/>(berita BNPB)"]
        T25["summ_news_daily<br/>(ringkasan harian)"]
    end

    subgraph responder["Responder & Operasi"]
        T9["tni_satker<br/>(satuan kerja TNI)"]
        T23["master_operasi_sar<br/>(operasi SAR)"]
        T27["master_operasi_bencana<br/>(operasi aktif)"]
        T26["master_rekomendasi_bencana<br/>(rekomendasi)"]
    end

    subgraph faskes["Fasilitas Kesehatan"]
        T10["master_puskesmas<br/>(puskesmas)"]
        T11["master_rs_nasional<br/>(rumah sakit)"]
        T21["master_data_penyakit_wabah<br/>(penyakit/wabah)"]
    end

    subgraph demografi["Demografi"]
        T20["bps_jumlah_penduduk<br/>(data penduduk BPS)"]
    end

    style cuaca fill:#bbdefb
    style gempa fill:#ffcdd2
    style bencana fill:#ffe0b2
    style gunung fill:#d7ccc8
    style prediksi fill:#b2ebf2
    style berita fill:#f0f4c3
    style responder fill:#c8e6c9
    style faskes fill:#f8bbd0
    style demografi fill:#d1c4e9
```

### Cakupan per Jenis Training Data

| Tabel | Text-to-SQL | SQL-to-Response | Web Knowledge |
|-------|:-----------:|:---------------:|:-------------:|
| bmkg_info_cuaca | v | v | v |
| bmkg_info_gempa | v | v | v |
| articles | v | v | - |
| laporan_bencana_harian | v | v | v |
| laporan_bencana_khusus | v | v | - |
| bmkg_info_bencana | v | v | - |
| master_kejadian_bencana | v | v | - |
| sitaba_bencana | v | v | - |
| tni_satker | v | v | v |
| master_puskesmas | v | v | v |
| master_rs_nasional | v | v | v |
| indo_hotspot | v | v | v |
| magma_letusan_gunung | v | v | v |
| master_gunung_api | v | - | v |
| prediksi_banjir | v | v | v |
| potensi_banjir | v | - | v |
| potensi_kebakaran | v | - | v |
| bmkg_news | v | - | - |
| bnpb_news | v | - | - |
| bps_jumlah_penduduk | v | v | v |
| master_data_penyakit_wabah | v | - | v |
| master_pergerakan_tanah | v | - | - |
| master_operasi_sar | v | v | v |
| sitaba_ketinggian_air | v | v | v |
| summ_news_daily | v | - | - |
| master_rekomendasi_bencana | v | - | - |
| master_operasi_bencana | v | - | - |

---

## 7. Konfigurasi & Mode

### Environment Variables (.env)

```
# Database (untuk conversation history & augmentasi lokasi)
POSTGRES_HOST, POSTGRES_PORT, POSTGRES_USER, POSTGRES_PASS, POSTGRES_DB_NAME

# Ollama (untuk Q&A generation di web scraper)
OLLAMA_BASE_URL="https://llm.alindo.id"
OLLAMA_MODEL="qwen3-coder:latest"

# Google Custom Search (untuk web knowledge)
GOOGLE_API_KEY=""
GOOGLE_CSE_ID=""
```

### Mode Training

```mermaid
graph LR
    subgraph modes["3 Mode"]
        M1["Default<br/>Qwen3-14B<br/>GPU standar"]
        M2["--h100<br/>Qwen3-14B<br/>H100 optimized"]
        M3["--local-test<br/>Qwen3-0.6B<br/>MacBook CPU/MPS"]
    end

    M1 -.- D1["batch=4, rank=64<br/>alpha=128, epoch=3"]
    M2 -.- D2["batch=8, rank=128<br/>alpha=256, TF32=on"]
    M3 -.- D3["batch=1, rank=8<br/>max_steps=5, seq=512"]
```

### CLI Flags (run_training.sh)

| Flag | Deskripsi |
|------|-----------|
| `--local-test` | Test pipeline di MacBook (3 steps saja) |
| `--h100` | Preset optimasi untuk NVIDIA H100 80GB |
| `--step N` | Mulai dari step ke-N (1-6) |
| `--model MODEL` | Override HuggingFace model ID |
| `--batch N` | Override batch size |
| `--rank N` | Override LoRA rank |
| `--epochs N` | Override jumlah epoch |
| `--quant TYPE` | Quantization: Q4_K_M, Q5_K_M, Q8_0 |
| `--name NAME` | Nama model di Ollama |
| `--no-db` | Skip ekstraksi database |
| `--skip-web` | Skip web knowledge scraping |
| `--use-unsloth` | Gunakan Unsloth (2x lebih cepat) |

---

## 8. Cara Menjalankan

### Persiapan Awal

```bash
# 1. Setup Google Custom Search (opsional, untuk web knowledge)
#    - https://console.cloud.google.com/ → Enable Custom Search API
#    - https://programmablesearchengine.google.com/ → Buat search engine
#    - Isi GOOGLE_API_KEY dan GOOGLE_CSE_ID di .env

# 2. Test pipeline di MacBook
./run_training.sh --local-test

# 3. Test tanpa database (tanpa PostgreSQL)
./run_training.sh --local-test --no-db
```

### Training Penuh (Server GPU)

```bash
# Default (Qwen3-14B, GPU standar)
./run_training.sh

# H100 80GB (optimized)
./run_training.sh --h100

# GPU kecil (8-16GB VRAM)
./run_training.sh --model Qwen/Qwen3-8B --batch 1

# Skip web scraping
./run_training.sh --skip-web

# Mulai dari step tertentu (misal sudah prepare data)
./run_training.sh --step 3
```

### Script Individual

```bash
# Web scraping saja
python scripts/scrape_web_knowledge.py --dry-run      # Preview
python scripts/scrape_web_knowledge.py                 # Full run
python scripts/scrape_web_knowledge.py --topic cuaca   # Satu kategori

# Prepare data saja
python scripts/prepare_training_data.py --no-db --no-web --no-augment

# Evaluasi saja
python scripts/evaluate_model.py --finetuned_model qwen3-aims-finetuned
```

---

## Struktur File

```
chatbot/
├── .env                              # Environment variables
├── run_training.sh                   # Main pipeline orchestrator
├── requirements-training.txt         # Python dependencies
├── skemadatabase.txt                 # Database schema (94 tabel)
│
├── scripts/
│   ├── populate_chromadb.py          # TABLE_TRAINING_DATA (27 tabel) + ChromaDB
│   ├── prepare_training_data.py      # Data preparation (6 sumber → JSONL)
│   ├── scrape_web_knowledge.py       # Google Search → Q&A via Ollama
│   ├── train_qwen3.py               # QLoRA fine-tuning
│   ├── export_to_ollama.py           # Merge + GGUF + Ollama register
│   └── evaluate_model.py            # Model comparison evaluation
│
├── data/                             # (gitignored)
│   ├── train.jsonl                   # Training dataset
│   ├── val.jsonl                     # Validation dataset
│   ├── metadata.json                 # Dataset statistics
│   ├── web_knowledge.jsonl           # Web-scraped Q&A pairs
│   ├── web_cache/                    # Cached web pages
│   └── web_search_log.json           # Idempotency log
│
├── output/                           # (gitignored)
│   ├── qwen3-finetuned/
│   │   └── final/                    # LoRA adapter weights
│   ├── qwen3-finetuned-q4_k_m.gguf  # Quantized model
│   └── logs/                         # Training & pipeline logs
│
└── evaluation/                       # (gitignored)
    └── comparison_report.json        # Evaluation results
```

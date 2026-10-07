# 🤖 RAG AI Agent - Panduan Penulisan Skripsi

Repositori ini berisi workflow **n8n (Self-Host)** untuk membuat **Agen AI Chatbot Web berbasis RAG (Retrieval-Augmented Generation)**. Sistem ini secara otomatis memproses dokumen panduan penulisan tugas akhir/skripsi dari Google Docs ke basis data vektor Supabase, lalu menjawab pertanyaan pengguna secara akurat berbasis data tersebut.

---

## 🛠️ Fitur & Keunggulan

* **Event-Driven Ingestion**: Otomatis mendeteksi dan mengolah file Google Docs baru di Google Drive menjadi data vektor.
* **Vector Store Supabase (`pgvector`)**: Menyimpan pencarian semantik (vektor) dengan tingkat presisi tinggi.
* **Google Gemini AI Models**:
  * `models/gemini-embedding-001` untuk pembuatan teks embedding.
  * Gemini Chat Model / Groq untuk pemrosesan obrolan interaktif.
* **Memori Percakapan**: Mengingat riwayat percakapan sebelumnya agar obrolan terasa alami.
* **Guardrails anti-Halusinasi**: AI diberi batasan untuk hanya menjawab berdasarkan dokumen yang tersedia.

---

## 🏗️ Arsitektur Workflow

Proyek ini terdiri dari dua alur utama (*pipeline*) dalam satu workflow n8n:

1. **Ingestion Pipeline**: 
   `Google Drive Trigger` ➔ `Edit Fields` ➔ `Download File (.txt)` ➔ `Extract from File` ➔ `Supabase Vector Store` (dengan `Gemini Embeddings` & `Recursive Text Splitter`).
2. **RAG AI Agent Pipeline**: 
   `Chat Trigger` ➔ `AI Agent` ➔ `Gemini Chat Model` + `Chat Memory` + `Supabase Vector Store (Tool)`.

---

## 🚀 Cara Menggunakan (Setup)

### 1. Penyiapan Database Supabase
Jalankan skrip SQL pada **SQL Editor** Supabase

### 2. Impor Workflow ke n8n
* Unduh file workflow.json dari repositori ini.
* Buka dashboard n8n kamu ➔ klik Workflows ➔ Import from File.
* Hubungkan kredensial:
* Google Drive OAuth2: Akses ke folder penyimpanan Google Docs.
* Supabase API Key & Host: Akses ke REST API Supabase.
* Google Gemini API: Akses ke model embedding dan chat.

### 3. Jalankan & Terbitkan Workflow
* Upload file Google Docs panduan skripsi ke folder target di Google Drive.
* Klik Execute Workflow untuk memproses file ke Supabase.
* Ubah status sakelar di n8n menjadi Active / Publish untuk mengaktifkan Web Chatbot publik.

### 📌 Deliverables & Pengujian
Workflow File: workflow.json
Status Ingestion: Berhasil terintegrasi tanpa error (Success 200 OK).

Web Chatbot: Terhubung dan dapat diuji secara publik via Production Chat URL.

### Jalankan skrip SQL berikut pada **SQL Editor** Supabase kamu untuk menyiapkan ekstensi `pgvector` dan tabel dokumen:

```sql
-- 1. Aktifkan ekstensi vector
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Buat tabel khusus dokumen skripsi
CREATE TABLE documents_skripsi (
  id bigserial PRIMARY KEY,
  content text,
  metadata jsonb,
  embedding vector(3072)
);

-- 3. Buat fungsi pencarian vektor
CREATE OR REPLACE FUNCTION match_documents_skripsi (
  query_embedding vector(3072),
  match_count int DEFAULT null,
  filter jsonb DEFAULT '{}'
) RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
) LANGUAGE plpgsql AS $$
BEGIN
  RETURN QUERY
  SELECT
    documents_skripsi.id,
    documents_skripsi.content,
    documents_skripsi.metadata,
    1 - (documents_skripsi.embedding <=> query_embedding) AS similarity
  FROM documents_skripsi
  WHERE documents_skripsi.metadata @> filter
  ORDER BY documents_skripsi.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;

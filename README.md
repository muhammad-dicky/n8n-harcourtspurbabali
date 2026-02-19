# 🤖 RAG AI Property Consultant (n8n + Supabase + WhatsApp)

**Version:** 1.0.0
**Stack:** n8n, Supabase (pgvector), OpenAI, Google Drive, WhatsApp Business API

## 📋 Overview

This n8n workflow implements an autonomous **Retrieval-Augmented Generation (RAG) Agent** designed for Real Estate queries. It functions as a "Property Consultant" that ingests property data (PDFs, Spreadsheets, Docs) from a specific Google Drive folder, vectorizes the content into a Supabase database, and serves client queries via WhatsApp.

The system is designed with a **"Hot-Sync" architecture**: as soon as files are added or updated in Google Drive, the knowledge base is automatically refreshed, ensuring the AI always recommends current listings.

## 🌟 Key Features

* **Multi-Modal Ingestion:** Automatically processes and categorizes diverse file formats:
* 📄 **PDFs** (Property Brochures)
* 📊 **Excel / Google Sheets** (Listing Price Lists)
* 📝 **Google Docs / CSV** (Text descriptions)


* **Vector Search (RAG):** Uses OpenAI Embeddings (`text-embedding-3-small`) and Supabase `pgvector` for semantic search.
* **Persistent Conversation Memory:** Stores WhatsApp chat history in Postgres, allowing the agent to remember context across turns.
* **Data Integrity Protocol:** The agent is prompted to strictly filter out listings with missing prices or invalid links ("Zero-Hallucination" policy on data availability).
* **Automated Sync:** Watches a Google Drive folder for `File Created` or `File Updated` events to keep the vector store pristine.
* **WhatsApp Interface:** Native integration for receiving user queries and sending formatted property cards.

## 🏗️ Architecture

The workflow consists of two main distinct pipelines:

### 1. The ETL (Extract, Transform, Load) Pipeline

1. **Trigger:** Detects new/updated files in Google Drive.
2. **Metadata Management:** Deletes existing vectors for the specific file ID (deduplication) and inserts fresh metadata.
3. **Extraction:** Downloads the file and routes it based on MIME type:
* *Spreadsheets:* Parsed into rows, aggregated, and summarized to preserve tabular context.
* *Documents:* Text extracted via LangChain loaders.


4. **Embedding:** Text is split into chunks and embedded using OpenAI.
5. **Storage:** Vectors and metadata are upserted into Supabase.

### 2. The Inference (Chat) Pipeline

1. **Trigger:** Incoming WhatsApp message.
2. **Context Retrieval:** The Agent searches the Supabase Vector Store for documents relevant to the user's prompt.
3. **Reasoning:** The LLM (OpenAI GPT) synthesizes the retrieved data into a "Property Consultant" persona response.
4. **Response:** Sends the final answer back to the user via WhatsApp.

## ⚙️ Prerequisites

* **n8n:** Self-hosted or Cloud (Version 1.0+ recommended with LangChain support).
* **Supabase Project:** With a Postgres database.
* **OpenAI API Key:** For Embeddings and Chat completion.
* **Google Cloud Console:** Service account with Drive API enabled.
* **Meta for Developers:** WhatsApp Business API app set up.

## 🛠️ Setup & Configuration

### 1. Database Schema (Supabase)

You must initialize your database before running the workflow. Run the following SQL commands in your Supabase SQL Editor:

```sql
-- 1. Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Create Metadata Table
CREATE TABLE document_metadata (
    id TEXT PRIMARY KEY,
    title TEXT,
    url TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    schema TEXT
);

-- 3. Create Documents/Vectors Table
CREATE TABLE documents (
    id bigserial primary key,
    content text,
    metadata jsonb,
    embedding vector(1536) -- Matches OpenAI text-embedding-3-small
);

-- 4. Create Similarity Search Function
CREATE OR REPLACE FUNCTION match_documents (
  query_embedding vector(1536),
  match_count int DEFAULT null,
  filter jsonb DEFAULT '{}'
) RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE metadata @> filter
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;

-- 5. Create Chat Memory Table (Optional if not auto-generated)
CREATE TABLE chat_memory (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id TEXT NOT NULL,
    message JSONB NOT NULL
);

```

### 2. Environment Credentials

Configure the following credentials in n8n:

* `Postgres account` (Supabase connection string)
* `OpenAI API`
* `Google Drive OAuth2 API`
* `WhatsApp API` (Access Token)

### 3. Node Configuration

After importing the JSON, verify the following nodes:

* **Google Drive Trigger:** Update the `Folder to Watch` ID (`1QlXYkIrX...`) to your specific Drive folder ID.
* **WhatsApp Trigger:** Ensure the Webhook URL matches your Meta App settings.
* **Postgres/Supabase Nodes:** Ensure the credentials are selected.

## 🧩 Workflow Logic Details

### Intelligent Spreadsheet Parsing

Unlike standard RAG parsers that flatten spreadsheets into nonsense, this workflow uses a specialized branch for Excel/CSV:

1. Extracts raw rows.
2. Aggregates all rows.
3. **Set Schema:** Dynamically grabs headers to create a "schema" string.
4. Injects the schema into the metadata so the LLM understands the column structure (e.g., "Price column represents Monthly Rent").

### The "Property Consultant" Persona

The Agent node (`RAG AI Agent1`) is pre-prompted with specific behavioral guardrails:

* **Tone:** Professional, relaxed, "Senior Agent".
* **Format:** Returns a structured list (Price, Location, Specs, Highlight, Link).
* **Validation:** It silently discards retrieved nodes that lack a valid URL or Price, ensuring users never see broken listings.

## 🚀 Usage

1. **Ingest Data:** Upload a PDF brochure or an Excel price list to the watched Google Drive folder.
2. **Wait for Sync:** Watch the n8n executions. You should see the `File Created` trigger fire and the data populate in Supabase.
3. **Chat:** Send a message to the configured WhatsApp number:
> "I'm looking for a 2-bedroom villa in Canggu under $300k."


4. **Response:** The agent will query the vector store and reply with specific listings found in your documents.

## 📄 License

This project is provided as-is. MIT License.

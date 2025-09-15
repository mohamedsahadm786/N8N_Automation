# N8N_Automation

---

## Prerequisites

- **n8n** (self-hosted or n8n Cloud)
- Accounts/API keys for the services you use in a given workflow:
  - **Telegram Bot** (via BotFather)
  - **Google AI Studio** API key (Gemini / Veo 3)
  - **OpenAI** API key (for embeddings)
  - **Supabase** project (URL + Service Role key) for vector storage
  - **Gmail** (OAuth2) if you want the cover-letter draft auto-created
  - **NanoBanana** (image editing API) for the photo editor

> Workflows are modular—configure only the credentials required by the one you import.

---

## Environment / Credentials

Configure via **n8n Credentials** or environment variables.

| Name | Used by | Purpose |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | 01, 03 | Telegram Trigger/Send nodes |
| `GOOGLE_API_KEY` | 02, 03, 04 | Gemini (LLM/vision) & Veo 3 video |
| `OPENAI_API_KEY` | 04 | Embeddings for RAG |
| `SUPABASE_URL` | 04 | Supabase REST URL |
| `SUPABASE_SERVICE_ROLE_KEY` | 04 | Vector upsert/read (service role recommended) |
| `NANOBANANA_API_KEY` | 01, 02 | Image editing API |
| n8n **Gmail OAuth2** cred | 04 | Create Gmail draft (select in node) |

> If credential names in the JSON don’t match yours, open the node and select your credential.

---

## Import & Run (Quick Start)

1. **Clone/Download** this repo.
2. In n8n: **Workflows → Import from File** and choose a JSON under `/workflows`.
3. Open the imported workflow and:
   - Select your **Credentials** on required nodes.
   - Confirm **base URLs**, **model names**, and any **folder/table IDs**.
4. **Activate** the workflow (or **Execute Workflow**) and test using the steps below.

---

## 01 — Prompt-to-Photo Editor (Telegram)

**Goal:** Send a photo + caption (prompt) to a Telegram bot; receive an edited image; iterate edits.

**Flow (core nodes):**
- `Telegram Trigger` → read incoming photo/caption  
- `Get File` / download → `HTTP Request` → **NanoBanana** edit  
- Optional confirm/retry loop  
- `Telegram → Send photo` → return result

**Usage:**  
Send a photo with a caption like: *“Make the sky dramatic, add warm sunset tones.”*  
Reply with a new caption to continue editing.

**Requires:** `TELEGRAM_BOT_TOKEN`, `NANOBANANA_API_KEY`.

---

## 02 — Photo → Video Creator (Edit + Veo 3)

**Goal:** Edit a photo (as in 01) and then generate a short video from that edited image using **Google Veo 3**.

**Flow (core nodes):**
- Photo edit path → `Convert to File`  
- `HTTP Request` → Veo 3 **create job**  
- `Wait / Poll` status → retrieve video URL  
- `Telegram → Send video`

**Usage:**  
After the edited image is returned, the workflow kicks off Veo 3. You’ll receive an **.mp4** in Telegram.

**Requires:** `NANOBANANA_API_KEY`, `GOOGLE_API_KEY`, `TELEGRAM_BOT_TOKEN`.

---

## 03 — Multimodal Telegram Assistant

**Goal:** Personal assistant on Telegram that accepts **text, audio, video, images, and documents**, then replies with grounded answers.

**Flow (core nodes):**
- `Telegram Trigger` with content-type **classification** lanes  
- Audio/video → transcription; image/doc → analysis/extract  
- `Gemini` (via HTTP/Gemini node) for reasoning  
- `Telegram → Send message` (optional follow-ups)

**Usage:**  
Send any supported modality to the bot; the assistant processes and replies. For large files, ensure n8n upload limits are adequate.

**Requires:** `TELEGRAM_BOT_TOKEN`, `GOOGLE_API_KEY` (plus ASR/OCR creds if you add them).

---

## 04 — RAG Cover-Letter Builder (Supabase + Gmail Draft)

**Goal:** Generate a job-specific cover letter based on **your background** stored in a **Supabase vector store**. Optionally save as a **Gmail draft**.

**Stages:**
1. **Vectorization** — ingest your background docs (PDFs/Docs), create **OpenAI embeddings**, and upsert into `Supabase Vector Store`.
2. **Generation** — provide job title, company, location, and JD; run **Gemini** with retrieved context; optionally create a **Gmail draft**.

**Usage:**
- Run the **vectorization** workflow once to index your background.  
- Trigger the **cover-letter** workflow with the job details.  
- Copy the letter or open the saved Gmail draft.

**Requires:** `OPENAI_API_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `GOOGLE_API_KEY`, and Gmail OAuth2 (if drafting).

---

## Configuration Notes

- **Model names:** Update Gemini/Veo endpoints if versions change.  
- **Supabase schema:** Vector table should include `id`, `content`, `metadata`, and `embedding (vector)`. Point the node to your table name.  
- **Telegram routing:** Most send nodes use chat ID from the trigger (`={{$json.chatId}}`). Set a static ID if needed.  
- **File limits:** Large media may require raising n8n’s `N8N_PAYLOAD_SIZE_MAX` and host upload limits.  
- **Costs/quotas:** External model calls incur provider costs. Add sensible retries and rate limits.

---

## Troubleshooting

- **Duplicate Telegram responses:** Keep only one active `Telegram Trigger` per bot; avoid overlapping polls. If you custom-poll, persist `update_id` offsets.  
- **Veo 3 “pending” forever:** Verify job ID and polling URL; use a `Wait` + status loop.  
- **RAG feels generic:** Confirm embeddings actually inserted (row counts), and that retrieval filters (namespace/user metadata) are correct.

---

## Roadmap

- **Code-first ports:** LangGraph, CrewAI, Agno, Microsoft AutoGen  
- **More no/low-code variants:** Make, Zapier  
- **Extras:** one-click env template, Supabase SQL, sample prompts

---



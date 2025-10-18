---
title: "How chat-PrynAI Handles File Uploads: From Bytes → Text → Useful Context (Safely)"
date: 2025-10-15
description: "The exact pipeline we use to accept files, extract text (with optional OCR), and feed a compact 'attachments context' to the model—streamed end-to-end."
tags: uploads, ocr, chainlit, fastapi, sse, security, llm
---

**TL;DR**  
- Users can drop up to **5 files** (≤ **10 MB** each).
- We read the bytes server‑side, extract text using pure‑Python parsers (PDF/Office/CSV/JSON/XML/code), optionally OCR images/PDFs, and build a compact **ATTACHMENTS CONTEXT** system message. No code is executed; audio/video are blocked.
- We stream the model’s response over SSE and log both user + assistant turns to the thread transcript.

### Why we built it this way

Uploads are powerful but risky. We wanted three things at once:

1. **Safety** — treat files as *data*, never as executable content.  
2. **Relevance** — extract just enough text to help the model reason.  
3. **Streamability** — keep the chat flow live while the answer is generated.

So the gateway does strict type/size checks, extracts text server‑side (optionally OCR), and injects a **single, compact context block** the model can use. :contentReference[oaicite:4]{index=4}

---

### The happy path (one screen, no surprises)

**Browser → Chainlit UI → Gateway `/api/chat/stream_files` → LangGraph agent → SSE back to browser**

- In the UI, when a message includes uploaded files, we build a multipart request with a JSON `payload` (the chat message + thread info) and the `files[]` array. The UI then streams the SSE back, appending tokens into one Chat message. :contentReference[oaicite:5]{index=5}

```python
# apps/chainlit-ui/src/main.py (trimmed)
uploads = _collect_uploads(message)
form = {"payload": json.dumps(payload)}
async with client.stream("POST", "/api/chat/stream_files", data=form, files=files, headers=headers) as resp:
    async for event, data in iter_sse_events(resp):
        if event == "done": break
        elif event == "policy": await cl.Message(content=f"**Safety notice:** {data}").send()
        else: await out.stream_token(data)
```
- On the server, the gateway’s uploads router owns POST /api/chat/stream_files, authenticates the user, enforces limits, extracts text, builds ATTACHMENTS CONTEXT, and streams the model response using SSE.
### Guardrails first: limits & filters

### How many / how big

- Max files: 5
- Max file size: 10 MB each
- Exceed either? We return 413. (The file is read in 512 KiB chunks and stopped if over limit.)

### Allowed extensions (lower‑case):
```
.pdf .docx .txt .csv .pptx .xlsx .json .xml .png .jpg .jpeg .gif .py .js .html .css .yaml .yml .sql .ipynb .md

Blocked: .exe .dll .bin .dmg .iso .apk .msi .so → 415. We also block any audio/ or video/ MIME → 415.
```
### UI expectations
- Chainlit’s UI is configured for spontaneous uploads: accept */*, max_files=5, max_size_mb=10. The server remains the source of truth for enforcement

### Text extraction: pure‑Python, pragmatic
- For each file, we try a dedicated parser; if none matches, we gracefully fall back:
#### PDF
- pypdf text extraction. If text is empty and OCR is enabled (see below), we OCR pages (capped).

#### DOCX/PPTX/XLSX 
- unzip and strip XML (word/document.xml, ppt/slides/*, selected xl/*), then squash whitespace.

#### CSV/TXT/MD/HTML/JS/PY/CSS/YAML/YML/SQL
- decode UTF‑8; crude tag‑strip for HTML to avoid markup noise.

#### IPYNB
- parse JSON and join cell sources.
#### JSON/XML
- pretty‑print or tag‑strip to human text

- All paths end up as plain text. There’s no code execution—we don’t “run” notebooks or scripts; we only extract their text

### Optional OCR: opt‑in, capped, and polite

- Enable OCR by setting UPLOADS_OCR=tesseract (defaults to none).
- Images (.png .jpg .jpeg .gif) → Tesseract via Pillow.
- PDFs → try native text first; otherwise render pages with PyMuPDF and OCR those bitmaps.
- Caps: UPLOADS_OCR_MAX_PAGES (default 10), UPLOADS_OCR_DPI (default 180), UPLOADS_OCR_LANG (default eng).

#### This ensures OCR never balloons latency or cost on a giant scan dump; we only do as much as configured and only when necessary.

### The secret sauce: ATTACHMENTS CONTEXT

- After extraction, we compose a compact system message:

```
ATTACHMENTS CONTEXT
Use only the content below for semantic understanding (no code execution).

• file-1.pdf

<trimmed text up to 12k chars>

• file-2.docx

<trimmed text up to 12k chars>

```
- Per‑file trim: 12,000 chars
- Total trim across all files: 24,000 chars (then we append a “truncated” notice)
- Goal: keep context semantic and bounded, so the model stays fast and relevant.
- We send this system message as the first message, followed by the user’s prompt, to the LangGraph agent.

### Streaming & transcripts (end‑to‑end)

- We stream tokens over Server‑Sent Events (“text/event-stream”), building events correctly (data: lines, blank line terminator). That keeps proxies happy and the UI simple.
- We run input moderation before invoking the agent, and best‑effort output moderation after the stream—if flagged, we emit a policy SSE event that the UI renders as a safety notice.
- We write transcripts: the user turn (prompt) right away, and the assistant turn (joined tokens) at the end—scoped to the current thread and user.

### Security posture (in one paragraph)

- We accept only a known set of extensions and block audio/video to avoid surprise workloads.
- We never execute uploaded content; we extract text only.
- We cap file counts and sizes—and we enforce those caps server‑side.
- OCR is opt‑in and capped by pages/DPI/lang.
- That’s the boring, dependable security you want around file uploads.


### How to reuse this pattern

- Turn on uploads in your UI, but enforce limits on the server. Use chunked reads and return 413/415 meaningfully.
- Extract text with pure‑Python parsers; only OCR when you must (and cap it).
- Build a single ATTACHMENTS CONTEXT message with per‑file and global trims; avoid dumping whole files.
- Stream the response over SSE and show safety notices inline. The UX feels instant and stays simple to debug.
- Log transcripts (user + assistant) per thread so conversations are durable and auditable

 ### Code hotspots (for the curious)

#### Gateway uploads router
-  /api/chat/stream_files (limits, extractors, OCR, context, SSE streaming).
apps/gateway-fastapi/src/features/uploads.py

#### SSE framing (gateway)
- spec‑compliant data: lines; shared helper and streaming in /api/chat/stream.
apps/gateway-fastapi/src/main.py

#### UI SSE parser
- tiny generator that yields (event, data); used by the chat handler.
apps/chainlit-ui/src/sse_utils.py

#### UI chat handler
- builds multipart request for uploads, streams chunks into a single message.
apps/chainlit-ui/src/main.py

#### Transcript API & helpers
- per‑thread messages storage and readout.
apps/gateway-fastapi/src/features/transcript.py

#### UI config for uploads
- max_size_mb=10. apps/chainlit-ui/src/config.toml


---
title: "Safety & Guardrails: Practical Defenses for Chat, Uploads, and Auth"
date: 2025-10-01
description: "How we keep conversations safe: input/output moderation, strict upload filters, HTML disabled in chat, authenticated APIs, and clear safety signals in the stream."
tags: security, moderation, uploads, chainlit, fastapi, sse, entra, jwt
---

> **TL;DR**  
> - **Auth** is handled with Microsoft Entra External ID; the UI bridges tokens to an **HttpOnly** cookie and the Gateway **verifies JWTs** (issuer/audience/signature).  
> - **Moderation** runs on **input and output**. If something trips a rule, the stream sends a `policy` event that the UI surfaces as a **Safety notice**.  
> - **Uploads** are **bounded and filtered** (known extensions only, size caps, audio/video blocked, optional OCR). We never execute user content—only **extract text**.  
> - **Rendering** in the chat UI has **HTML disabled**; we style Markdown safely.  
> - **CORS** is locked to a small allowlist.  
> All of this is implemented in a handful of files with small, readable helpers. 

---

## 1) Identity, sessions, and API access (defense #1)

**Browser → /auth → HttpOnly cookie → Chainlit session → Gateway JWT check**

- The UI serves an **/auth** page that uses MSAL (Microsoft Entra External ID). After sign‑in, we **POST a token** to `/_auth/token`; the UI server sets an **HttpOnly** cookie (no JavaScript access) and also clears it on logout. Chainlit’s `header_auth_callback` then reads the cookie (or Authorization header) to establish the session. :contentReference[oaicite:5]{index=5}  
- On the Gateway, every API request extracts and **verifies the JWT** against the tenant’s OIDC discovery and **JWKS** keys; we validate issuer/audience and handle expiry. A dev‑only bypass exists for local smoke tests.
- Gateway CORS is explicit: **`https://chat.prynai.com`** and **`http://localhost:3000`** by default. 

**Why this matters:** users authenticate through a standard OIDC flow, secrets never need to live in your frontend code, and every API turn is protected by **signature‑verified** tokens. 

---

## 2) Moderation on both sides of the model (defense #2)

We wrap the model call with **input** and **output** checks and **signal safety** in the stream.

- Config toggles: `MODERATION_ENABLED` (default **true**) and `MODERATION_MODEL` (`omni-moderation-latest`). These are defined once in the Gateway (and mirrored in the uploads path). 
- **Input moderation**: if a user message is flagged, we **don’t** run the model. Instead we stream a `policy` event and end the SSE with a clear message (the UI renders it as a Safety notice). 
- **Output moderation**: after we’ve streamed tokens, we optionally check the full text; if that’s flagged, we emit a `policy` event near the end. (We still deliver the tokens, but make the safety status explicit.)

**UI behavior:** our SSE client listens for `policy` events and shows them as a separate message (“Safety notice: …”). We parse the stream with a small `iter_sse_events` generator and route by `event` type. 

---

## 3) File uploads with strict boundaries (defense #3)

Uploads are useful—and dangerous if over‑privileged. Ours are intentionally narrow:

- **Limits & filters**  
  - Max files: **5**; max size: **10 MB** each. The server reads in **512 KiB chunks** and returns **413** if over. :contentReference[oaicite:16]{index=16}  
  - Allowed extensions (examples): **.pdf, .docx, .txt, .csv, .pptx, .xlsx, .json, .xml, .png, .jpg, .jpeg, .gif, .py, .js, .html, .css, .yaml, .yml, .sql, .ipynb, .md**. Blocked: executables (**.exe, .dll, .bin, .dmg, .iso, .apk, .msi, .so**). **Audio/** and **video/** MIME types are rejected with **415**. :contentReference[oaicite:17]{index=17}  
  - The Chainlit UI is configured for **spontaneous uploads** (accept `*/*`, `max_files=5`, `max_size_mb=10`) but **the server is the source of truth** for enforcement. :contentReference[oaicite:18]{index=18}

- **Extraction only (no execution)**  
  We convert bytes to text with pure‑Python parsers: pypdf for PDFs; unzip & strip XML for **docx/pptx/xlsx**; pretty‑print or tag‑strip for **json/xml/html**; decode **txt/md/js/py**; parse **ipynb** cells. If a PDF page has no extractable text and OCR is **enabled**, we render page bitmaps and OCR them—**opt‑in and capped** by page count/DPI/lang. :contentReference[oaicite:19]{index=19}

- **Batched, bounded context**  
  The server builds one system message named **“ATTACHMENTS CONTEXT”** with files listed and each file’s text **trimmed to 12k chars**; we cap total across files at **24k chars** to keep the model prompt lean. No code is ever run—this is **semantic context only**. :contentReference[oaicite:20]{index=20}

- **Streaming safety, again**  
  The uploads path uses the same SSE streaming and **policy** signaling as the no‑file path. We also write transcript entries (user then assistant) per thread. :contentReference[oaicite:21]{index=21}:contentReference[oaicite:22]{index=22}

---

## 4) Rendering in the UI: HTML off, Markdown styled (defense #4)

- In Chainlit, **HTML rendering is disabled** (`unsafe_allow_html = false`). This avoids passing model‑emitted HTML (or any user HTML) directly to the DOM. Instead, we rely on Markdown → HTML conversion inside Chainlit and a **small CSS theme** for code, tables, and blockquotes—safer by default. :contentReference[oaicite:23]{index=23}  
- The Markdown theme is loaded via `custom_css` and scoped to the chat root, keeping styling predictable. :contentReference[oaicite:24]{index=24}

**Result:** clear, readable messages without exposing a generic “HTML injection” surface in the chat. :contentReference[oaicite:25]{index=25}

---

## 5) The streaming contract: make safety visible (defense #5)

We stream with **SSE** (“`text/event-stream`”), formatting events correctly (`data:` lines joined by newlines, blank line to terminate). The Gateway uses a tiny helper to frame chunks, and the UI parser collects them into a single message—while still handling `policy` and `error` events specially. This keeps safety **user‑visible** without breaking the typing‑back effect.

---

## Fail closed (where possible), fail loud (when necessary)

- **Auth missing/invalid** → we stream an `error` event, then `[DONE]`. You see a clear message and nothing else happens.
- **Moderation blocked** → we stream a `policy` event explaining why, then end the stream. The user sees a **Safety notice**. 
- **Upload rejected** (too big, wrong type) → standard HTTP **413/415** with explicit reason.

---

## Configuration cheat‑sheet

- **Gateway moderation**  
  `MODERATION_ENABLED=true|false` (default **true**), `MODERATION_MODEL=omni-moderation-latest`. 
- **Uploads**  
  `UPLOADS_OCR=none|tesseract`, `UPLOADS_OCR_MAX_PAGES` (default **10**), `UPLOADS_OCR_DPI` (**180**), `UPLOADS_OCR_LANG` (**eng**). 
- **Auth / cookies (UI server)**  
  `/_auth/token` sets the **HttpOnly** cookie; `/ _auth/logout` clears it; `COOKIE_DOMAIN` is honored when set. 
- **CORS**  
  Allowlist is defined in the Gateway (`ALLOWED_ORIGINS`). 
  Chainlit config: `unsafe_allow_html = false`; custom CSS: `/public/markdown-theme.css`. 
---

## Known limitations (and what we’ll add next)

- **No AV scanning of uploads** yet. Today we rely on type/size/extension filters and text extraction only. If your threat model requires it, wire a clamav‑style sidecar before extraction. (Out‑of‑scope in this repo.)  
- **No per‑user safety dashboard** yet. We already write transcripts per thread; surfacing moderation hits in a UI would help teams audit content. 
- **Rate limiting** is not implemented at the Gateway. Consider adding IP‑ or user‑scoped limits at your ingress or API layer.

---

## Code hotspots (jump list)

- **Gateway (moderation + SSE framing + CORS)**  
  `apps/gateway-fastapi/src/main.py` 

- **Uploads pipeline (limits, extraction, OCR, “ATTACHMENTS CONTEXT”, SSE)**  
  `apps/gateway-fastapi/src/features/uploads.py` 

- **Auth (verify JWT via OIDC discovery + JWKS)**  
  `apps/gateway-fastapi/src/auth/entra.py` 

- **UI server (HttpOnly cookie bridge, Chainlit header auth)**  
  `apps/chainlit-ui/src/server.py` 

- **UI SSE parser + chat streamer (policy/error handling)**  
  `apps/chainlit-ui/src/sse_utils.py` 
  `apps/chainlit-ui/src/main.py` 

- **Chainlit UI config (HTML off, uploads enabled, custom CSS)**  
  `apps/chainlit-ui/src/config.toml`

- **Transcript API & helper**  
  `apps/gateway-fastapi/src/features/transcript.py` 

---

**Takeaway:** Security doesn’t have to be heavyweight. Auth + verified JWTs, clear **moderation** on both ends of the model, **strict upload boundaries**, and **HTML‑off** rendering give you a defense‑in‑depth baseline that’s easy to reason about and hard to misuse—while keeping the streaming UX intact.

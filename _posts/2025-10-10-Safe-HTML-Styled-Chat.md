---
title: "How PrynAI Renders AI Replies: Markdown → Safe HTML → Styled Chat"
date: 2025-10-18
description: "The practical pipeline we use to turn LLM Markdown into secure, readable chat—without letting raw HTML creep in."
tags: frontend, chainlit, markdown, xss, ui, accessibility
---

#### In this post I’ll show- The practical pipeline we use to turn LLM Markdown into secure, readable chat—without letting raw HTML creep in.
Repo (server + clients + infra): https://github.com/PrynAI/PrynAI-chat/tree/main

**TL;DR**  
- Our chat UI renders model output as **Markdown**, not raw HTML. We keep Chainlit’s HTML turned off, apply a small custom CSS theme, and stream tokens into the message as they arrive. This gives us **safety by default** and **pleasant, legible output** with almost zero moving parts. The relevant knobs live in `config.toml`, and the styling lives in `markdown-theme.css`.

---

## The problem we had to solve
LLMs speak *Markdown*—lists, code fences, tables, blockquotes. That’s great for comprehension, but it tempts you to render **untrusted HTML** into a chat window. We wanted three things at once:

1) **Safety:** no raw HTML injection paths.  
2) **Readability:** code blocks, tables, quotes that don’t look like a ransom note.  
3) **Simplicity:** minimal JavaScript and no heavy sanitization stack to babysit.

---

## Our rendering pipeline (boringly pragmatic)

**Model output (Markdown)** → **Chainlit Markdown renderer** → **safe HTML (no raw HTML allowed)** → **small CSS theme**

- We explicitly **disable HTML** in Chainlit so model output can’t inject tags/scripts. That’s a one‑line switch:
  
  ```toml
  # apps/chainlit-ui/src/config.toml
  [features]
  unsafe_allow_html = false
  ```

- Chainlit’s own comment says it plainly: “Process and display HTML in messages. This can be a security risk…”—we keep it off and let Markdown do the heavy lifting
- We attach a custom CSS file that only styles what the Markdown renderer already emits. In Chainlit, this is set here:

```
# apps/chainlit-ui/src/config.toml
[UI]
custom_css = "/public/markdown-theme.css"
custom_js  = "/public/login-redirect.js"
```

- The chat handler uses that generator and calls out.stream_token(...) for each chunk, then finalizes the message.

  ## The CSS that does the real work

 - We keep the CSS tiny and focused on readability. Here are the representative bits we ship:

```
/* apps/chainlit-ui/src/public/markdown-theme.css */
.cl-root pre {
  background: #0f0f13;
  border: 1px solid #2a2a2e;
  padding: 12px 14px;
  border-radius: 8px;
  overflow: auto;
}

.cl-root code {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace;
  font-size: .95em;
}

.cl-root p code {
  background: rgba(127, 127, 127, .15);
  padding: 0 .25em;
  border-radius: 4px;
}

.cl-root blockquote {
  border-left: 3px solid #7b5cff;
  margin: .5rem 0;
  padding: .25rem .75rem;
  opacity: .9;
}

.cl-root table {
  border-collapse: collapse;
  margin: .75rem 0;
}
.cl-root th,
.cl-root td {
  border: 1px solid #2a2a2e;
  padding: .4rem .6rem;
}

```

- Code blocks get a dark panel, subtle border, and horizontal scroll if needed.
- Inline code gets a pill background for instant visual scanning.
- Blockquotes get a left rule (purple accent) and breathing room to reduce wall‑of‑text fatigue.
- Tables get borders and spacing so they’re legible on first glance.
- All of this is in apps/chainlit-ui/src/public/markdown-theme.css

### Where this shows up in the app
#### Config:
- apps/chainlit-ui/src/config.toml toggles HTML off and wires the CSS into the UI.

#### Runtime streaming:
- apps/chainlit-ui/src/main.py consumes SSE and streams tokens into one growing message (then update() to finalize).

#### SSE parsing: 
- apps/chainlit-ui/src/sse_utils.py turns the text/event-stream into (event, data) tuples that our UI loop consumes.
- If you’re curious how the server shapes the SSE, the gateway builds spec‑compliant events (multiple data: lines, blank line terminator) and streams them back to the browser. That logic lives in the gateway’s chat streaming route.

### Why we did it this way (the trade‑offs)

#### Safety by default.
  - With unsafe_allow_html = false, model output can’t inject raw HTML. We rely on Markdown conversion that Chainlit performs instead of juggling our own sanitizer/HTML whitelist. Less surface area, fewer toys to break.

  #### Legibility without JS bloat.
   - A single CSS file improves reading flow for code, tables, and quotes. No client‑side Markdown libraries, no runtime sanitizers.

  #### Great with streaming.
  - Because the UI streams token deltas into a single message, users see thoughts unfold line by line. Our SSE utility is tiny and robust.

  #### Consistent across themes.
  - The selectors are scoped under .cl-root, so they play nicely with Chainlit and don’t leak into other UI parts

### What we didn’t do (on purpose)
- - We don’t render raw HTML from the model. If a future feature requires HTML, we’ll gate it behind role checks and still keep a sanitizer in the path. Today, Markdown only.
- We don’t add a syntax‑highlighting library yet. Chainlit’s Markdown output is already readable with the monospace stack and dark block panels. If we add Prism or highlight.js later, we’ll do it selectively to keep bundles small.

## Benefits you can reuse
- Flip the same two switches in your project:
    - Set unsafe_allow_html = false to avoid raw HTML injection paths.
    - Add a custom CSS file for code/blockquote/table polish.
- Keep streaming simple: parse SSE and call your UI’s “append token” method until done. It reads well and debugs easily.
- Optimize for reading over “fancy.” Most users copy code, scan tables, and skim lists. Those are the elements we tuned.

## Pointers to repo docs & code
- Feature note → Chat Responses: Styled HTML (docs/Features/ChatResponsesStyledHTML.md).
- CSS theme → apps/chainlit-ui/src/public/markdown-theme.css.
- Chainlit UI config → apps/chainlit-ui/src/config.toml.
- SSE parser → apps/chainlit-ui/src/sse_utils.py
- Chat streaming UI → apps/chainlit-ui/src/main.py (uses iter_sse_events, streams via out.stream_token(...)).
  

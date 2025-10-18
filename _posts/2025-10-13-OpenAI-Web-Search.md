---
title: "Fresh Answers on Demand: How PrynAI Uses OpenAI Web Search"
date: 2025-10-18
description: "A code-backed walkthrough of our Web Search feature—from the UI toggle to the agent that (optionally) forces a single search call per turn."
tags: openai, web-search, responses-api, chainlit, langgraph, sse
---

**TL;DR**  
- We added a **Web search** toggle to chat settings. When it's ON, the agent uses the **OpenAI built‑in `web_search` tool** and we **force exactly one call** to it per turn.
- The agent also receives a small system tip that says “include the source URLs you used.” When the toggle is OFF, the model runs plain—no tools bound. This is implemented with a UI switch → request flag → LangGraph config → agent code that binds the tool via the **Responses API**.


## The problem we’re solving

- Some questions demand **fresh facts**: “Who just won?”, “What’s the latest release?”, “today”, “price”, “score”. We wanted answers that **cite sources** and don’t hallucinate recency. Rather than bolting on a custom crawler, we use **OpenAI’s built‑in `web_search` tool** exposed by the **Responses API**—and we bind it only when the user asks for it via a simple toggle.


## From toggle to tool call: the end‑to‑end path

### 1) The UI: a single switch in Chat Settings
We render a “**Web search (OpenAI)**” switch in Chainlit’s settings panel. Its value is stored in the user session so it’s available on the very first turn. The helper also exposes `is_web_search_enabled()` for easy reads. 

```python
# apps/chainlit-ui/src/settings_websearch.py (trimmed)
Switch(id="web_search", label="Web search (OpenAI)", initial=False, tooltip="Allow the model to use OpenAI's built-in Web Search tool.")
# … cl.user_session.set("chat_settings", {"web_search": False})

```
- When the user sends a message, the UI includes that boolean in the payload it streams to the Gateway (both the no‑file and file‑upload paths).

```
# apps/chainlit-ui/src/main.py (trimmed)
payload = {
  "message": message.content,
  "thread_id": _active_thread_id(),
  "web_search": is_web_search_enabled(),
}

```
## The Gateway: carry the flag, don’t interpret it

- The Gateway validates auth, then just passes the flag through into the LangGraph config under configurable.web_search. We keep the shape tiny using a Pydantic model.

```
# apps/gateway-fastapi/src/features/websearch.py (trimmed)
class ChatIn(BaseModel):
    message: str
    thread_id: str | None = None
    web_search: bool = False

def build_langgraph_config(payload: ChatIn) -> dict:
    cfg = {"configurable": {}}
    if payload.thread_id:
        cfg["configurable"]["thread_id"] = payload.thread_id
    cfg["configurable"]["web_search"] = bool(payload.web_search)
    return cfg
```
- In the streaming endpoint we attach user_id into the same configurable block so the agent can scope memory and transcripts correctly. Then we stream SSE back to the browser.

## The Agent: Responses API + built‑in web_search

- This is where the switch becomes behavior. We create a base ChatOpenAI that uses the Responses API (so built‑in tools are available) and streams. If configurable.web_search is False: we return the base LLM and the original messages—no tools bound. If it’s True: we prepend a system tip and bind the tool, forcing one call to "web_search" via tool_choice="web_search"

```
  # apps/agent-langgraph/my_agent/features/web_search.py (trimmed)
def llm_and_messages_for_config(config, messages):
    llm = ChatOpenAI(model="gpt-5-mini", streaming=True, use_responses_api=True, output_version="responses/v1")
    if not should_use_web_search(config):
        return llm, messages
    msgs = [SystemMessage(content=SYSTEM_TIP_WHEN_SEARCH_ON)] + list(messages)
    llm  = llm.bind_tools([{"type": "web_search"}], tool_choice="web_search")  # force one call
    return llm, msgs
```
- The system tip is short and explicit: “When the user requests current or time‑sensitive facts… call web_search… Include the source URLs you used.” This nudges the model to cite in its natural language response.

## Where the graph wires it in

- Our chat graph calls the helper above before invoking the model. Long‑term memory (if any) can also prepend a compact system tip—search and memory are orthogonal.

## Why we force a tool call (and how to change it)

- With the toggle ON, we currently force exactly one tool call per turn. Why? Determinism and debuggability: if a user flips it on, they’re asking “please use the web.” For a future “Auto” mode, the same helper already supports tool_choice="Auto"; you’d just flip force_specific_tool=False in with_openai_web_search to let the model decide per prompt. Today, we keep it simple and explicit.

## Streaming, moderation, transcripts—unchanged

### Search doesn’t change the rest of the engine:
#### SSE streaming:
- The Gateway emits spec‑compliant text/event-stream with data: lines and blank‑line terminators. The UI’s tiny SSE parser streams tokens into a single growing message
#### Input/output moderation:
- Best‑effort moderation surrounds the turn; violations show as policy events in the stream.
#### Transcripts:
- We write the user turn immediately and the assistant turn after streaming completes, scoped to the active thread and user.

## What the user sees
- A settings switch labeled “Web search (OpenAI).” Off by default. Flip it on for time‑sensitive queries.
- Answers that include source URLs when the model uses the tool (per system tip).
- The same fast streaming UX and thread history as non‑search chats.

## Trade‑offs we made

#### Explicit > implicit.
- Users control search; we don’t silently browse. That’s why the toggle exists and defaults to OFF.

#### Determinism when ON.
- Forcing one call makes behavior predictable and cost‑bounded per turn. If you prefer heuristics, switch to tool_choice="Auto".

#### Keep the agent simple
- We didn’t write a custom scraping pipeline; we rely on the built‑in tool surface provided by the Responses API. Maintenance stays low.

## Code hotspots (jump list)

- UI toggle & session storage
apps/chainlit-ui/src/settings_websearch.py

- UI payload includes web_search flag (both /stream and /stream_files)
apps/chainlit-ui/src/main.py

- Gateway shape & config pass‑through (ChatIn, build_langgraph_config)
apps/gateway-fastapi/src/features/websearch.py

- Agent helper: prepend tip, bind tool, force tool call
apps/agent-langgraph/my_agent/features/web_search.py

- Graph wiring: choose LLM/messages (search on/off), then run turn
apps/agent-langgraph/my_agent/graphs/chat.py

- Streaming & transcript writing (unchanged by search)
Gateway stream: apps/gateway-fastapi/src/main.py
Transcript API: apps/gateway-fastapi/src/features/transcript.py

## FAQ
#### Q: Will it search even if my question isn’t time‑sensitive?
- With the toggle ON, yes—one search call is forced by design for now. If you want the model to decide, we can switch to an “Auto” mode using the same helper.

#### Q: Where do the source links come from?
- The system tip nudges the model to include the URLs it uses when web_search is enabled

#### Q: Does this change memory or transcripts?
- No. Memory retrieval/writes and transcript logging are unchanged; search only affects the model/tool binding and a small system tip.

#### Takeaway:
- Fresh info shouldn’t require a complicated sidecar. A toggle, a config flag, and a tiny agent helper are enough to get reliable, cited answers when you need them—and plain, cheap answers when you don’t.
  

Perfect ✅ — let’s wire up **token tracking** inside your **FastAPI + LangGraph + MCP multi-node orchestration agent** (the one you mentioned that runs `JiraNode → JenkinsNode → KBNode → SummaryNode`).

Below is a **drop-in modification** that will give you **per-request token usage** and **aggregate cost tracking** (works with OpenAI-compatible or Tachyon endpoints).

---

## 🧠 Goal

Add token tracking so that every request to your orchestration agent (each new session) logs:

* Input tokens
* Output tokens
* Total tokens
* Cost (USD or equivalent)
* Optionally: log to console / database / Prometheus

---

## 🧩 Step-by-Step Integration

### 1. Import tracking utilities

At the top of your `agent_with_asyncstack_and_endpoint.py` (or wherever your FastAPI endpoint is):

```python
from langchain.callbacks import get_openai_callback
import logging

logger = logging.getLogger(__name__)
```

---

### 2. Wrap the **outermost agent invocation**

Locate where your FastAPI route or handler calls the global agent — likely something like:

```python
result = global_agent.invoke(session_input)
```

Replace it with a callback context:

```python
from datetime import datetime

with get_openai_callback() as cb:
    result = global_agent.invoke(session_input)

# After agent finishes
logger.info(
    f"[{datetime.now()}] Session={session_id} "
    f"PromptTokens={cb.prompt_tokens} "
    f"CompletionTokens={cb.completion_tokens} "
    f"TotalTokens={cb.total_tokens} "
    f"Cost=${cb.total_cost:.6f}"
)
```

✅ This ensures that **all tokens consumed across all nodes (Jira → Jenkins → KB → Summary)** are captured together, since the callback propagates automatically through LangGraph’s `Runnable` hierarchy.

---

### 3. Optional — add persistent tracking (CSV or DB)

If you want to accumulate session metrics:

```python
import csv, os
from pathlib import Path

usage_log = Path("token_usage.csv")
header = ["timestamp", "session_id", "prompt_tokens", "completion_tokens", "total_tokens", "cost"]

# Initialize once (e.g., at app startup)
if not usage_log.exists():
    with open(usage_log, "w", newline="") as f:
        csv.writer(f).writerow(header)

# Each request
with get_openai_callback() as cb:
    result = global_agent.invoke(session_input)

with open(usage_log, "a", newline="") as f:
    csv.writer(f).writerow([
        datetime.now(),
        session_id,
        cb.prompt_tokens,
        cb.completion_tokens,
        cb.total_tokens,
        cb.total_cost,
    ])
```

This gives you a daily/weekly log of token usage and spend per user/session.

---

### 4. Optional — expose usage via FastAPI route

Add a quick diagnostic endpoint:

```python
from fastapi import APIRouter
import pandas as pd

router = APIRouter()

@router.get("/usage")
def get_usage():
    df = pd.read_csv("token_usage.csv")
    summary = {
        "total_sessions": len(df),
        "total_tokens": int(df["total_tokens"].sum()),
        "total_cost_usd": float(df["cost"].sum()),
    }
    return summary
```

Now you can call `/usage` to see cumulative spend.

---

### 5. If you’re using **non-OpenAI LLMs** (like Tachyon / local LLaMA)

`get_openai_callback()` won’t return token data automatically —
so implement a **custom callback handler** and attach it when initializing your agent:

```python
from langchain.callbacks.base import BaseCallbackHandler

class TokenTracker(BaseCallbackHandler):
    def __init__(self):
        self.input_tokens = 0
        self.output_tokens = 0

    def on_llm_end(self, response, **kwargs):
        usage = getattr(response, "usage_metadata", {})
        self.input_tokens += usage.get("input_tokens", 0)
        self.output_tokens += usage.get("output_tokens", 0)

tracker = TokenTracker()
agent = create_react_agent(llm, prompt, callbacks=[tracker])
```

Then print or store `tracker.input_tokens`, `tracker.output_tokens` after `.invoke()`.

You can even normalize cost with your local per-token pricing if Tachyon exposes it.

---

## 🧾 Example Combined (FastAPI Handler)

Here’s how it all fits inside an endpoint:

```python
@app.post("/run-agent")
async def run_agent(request: Request):
    session_id = request.headers.get("X-Session-ID", "default")

    data = await request.json()
    session_input = data.get("input", "")

    with get_openai_callback() as cb:
        result = global_agent.invoke({"input": session_input})

    # Log usage
    logger.info(
        f"[Session {session_id}] Prompt={cb.prompt_tokens}, "
        f"Completion={cb.completion_tokens}, Total={cb.total_tokens}, "
        f"Cost=${cb.total_cost:.6f}"
    )

    return {
        "session_id": session_id,
        "result": result,
        "usage": {
            "prompt_tokens": cb.prompt_tokens,
            "completion_tokens": cb.completion_tokens,
            "total_tokens": cb.total_tokens,
            "cost_usd": cb.total_cost,
        },
    }
```

Now each API call returns token/cost stats directly in the response 👇

```json
{
  "session_id": "abc123",
  "result": "Build succeeded for Jenkins pipeline SDLCDEVOPS-755075",
  "usage": {
    "prompt_tokens": 356,
    "completion_tokens": 478,
    "total_tokens": 834,
    "cost_usd": 0.00253
  }
}
```

---

## 🚀 TL;DR

| Goal                     | Implementation                                   |
| ------------------------ | ------------------------------------------------ |
| Track tokens per request | Wrap `.invoke()` in `with get_openai_callback()` |
| Track across nodes       | Works automatically — callbacks propagate        |
| Non-OpenAI models        | Use custom `BaseCallbackHandler`                 |
| Persist usage            | Write to CSV / DB after each session             |
| Aggregate                | Sum across logs or expose `/usage` endpoint      |

---

Would you like me to show the same integration using your **LangGraph flow definition** (e.g., instrumenting inside `GraphRunnable` or each node separately for per-node usage)?
That’s useful if you want to see *which node* (Jira vs Jenkins vs KB) consumes the most tokens.

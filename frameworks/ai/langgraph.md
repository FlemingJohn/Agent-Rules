# Writing LangGraph code

Builds on `core/` and `languages/python.md` or `languages/typescript.md`.
This file covers only what is special to LangGraph. Every node, state shape,
schema, prompt and router sits alone in its own file, named after it.

The examples use one made-up graph throughout: a support request comes in, a
model sorts it, and it is either answered or passed to a person. Swap in your
own domain; the patterns stay the same.

Written against LangGraph 1.x (Python `langgraph`, TypeScript
`@langchain/langgraph`). Check the API names against your installed version.

---

## The rules

1. **Three shapes of state.** What comes in, what goes out, and what the graph
   uses inside. Callers never see the inside.
2. **Nodes return only what they changed.** Never copy the whole state back.
3. **The node that decides, routes.** Use `Command` to update and choose the
   next node in one return. No separate router full of `if` and `else`.
4. **One status field, not many flags.** `status: "answered"` beats
   `is_done`, `has_error`, `needs_review` side by side.
5. **The model's answer has a schema.** Structured output, never parsing text
   by hand.
6. **State holds ids, not things.** Big data lives in the database. State
   carries the id to find it.
7. **Settings and clients are context, not state.** Model, database client,
   tenant id: passed in at run time, never saved in a checkpoint.
8. **Failures retry by policy.** `RetryPolicy` on the node, not `try` and
   `except` branches inside it.

---

## Naming

Plain words that say what the thing does. No cleverness, no shorthand.

- **Verbs for things that do:** `get`, `set`, `make`, `start`, `end`,
  `update`, `handle`, `check`, `write`, `send`, `pick`.
- **Nouns for things that are:** `request`, `results`, `status`, `reply`.
- **`is`, `has` or `can` for yes or no:** `is_urgent`, `has_owner`,
  `can_retry`.
- **No abbreviations:** `configuration` not `config`, `message` not `msg`,
  `result` not `res`, `response` not `resp`. The exception is names the outside
  world already uses, like `id`, `url` and `thread_id`.
- **Name things for what they hold or do,** never for what they look like or
  where they sit. `node1`, `step_a`, `helper` and `utils` say nothing.

| Thing | Python | TypeScript | Rule |
|---|---|---|---|
| Node | `classify_request` | `classifyRequest` | verb then noun |
| Node maker | — | `makeClassifyRequest` | `make` then the node name, when the node needs the model |
| Router function | `pick_next_step` | `pickNextStep` | starts with `pick` |
| Fan-out function | `send_each_item` | `sendEachItem` | starts with `send_each` |
| Graph builder | `make_request_graph` | `makeRequestGraph` | starts with `make`, ends with `graph` |
| Input state | `RequestInput` | `RequestInput` | area then `Input` |
| Output state | `RequestOutput` | `RequestOutput` | area then `Output` |
| Inside state | `RequestState` | `RequestState` | area then `State` |
| Model answer schema | `Classification` | `Classification` | noun for what the model returns |
| Status type | `RequestStatus` | `RequestStatus` | area then `Status` |
| Status values | `"answered"` | `"answered"` | past tense or adjective, lower case |
| Run settings | `AppContext` | `AppContext` | ends with `Context` |
| State keys | `request_text`, `results` | `requestText`, `results` | nouns |
| Yes or no keys | `is_urgent`, `has_owner` | `isUrgent`, `hasOwner` | `is`, `has`, `can` |
| Prompt builder | `make_classify_prompt` | `makeClassifyPrompt` | starts with `make`, ends with `prompt` |

The node name in the graph is the function name. Python takes it from the
function. In TypeScript, use the same string as the function name.

    good                    bad
    ──────────────────────────────────────────
    classify_request        node1
    pick_next_step          router
    send_each_item          fanout
    RequestState            State
    results                 res
    is_urgent               urgent_flag
    status                  st
    make_request_graph      build

**Python file names.** One thing per file, named after the thing in
`snake_case`, the way Python modules are written. `request_state.py` holds
`RequestState`. `classify_request.py` holds `classify_request`.

**TypeScript file names.** One thing per file, in the thing's own case.
`RequestState.ts` holds `RequestState`. `classifyRequest.ts` holds
`classifyRequest`.

---

## Folder layout

One folder per graph. Inside it, a folder per kind of piece, read in the
order the work happens.

Python:

    request_graph/
      state/
        request_input.py
        request_output.py
        request_state.py
        request_status.py
      schemas/
        classification.py
      prompts/
        make_classify_prompt.py
      nodes/
        classify_request.py
        write_answer.py
        escalate_request.py
      context/
        app_context.py
      make_request_graph.py

TypeScript:

    src/requestGraph/
      state/
        RequestInput.ts
        RequestOutput.ts
        RequestState.ts
        RequestStatus.ts
      schemas/
        Classification.ts
      prompts/
        makeClassifyPrompt.ts
      nodes/
        makeClassifyRequest.ts
        writeAnswer.ts
        escalateRequest.ts
      makeRequestGraph.ts

---

## State: three shapes

The caller sends `RequestInput` and gets back `RequestOutput`. Everything
else stays inside `RequestState`. Nodes can also read a narrower shape of
their own, so each one sees only what it needs.

### Python

`state/request_status.py`

```python
from typing import Literal

RequestStatus = Literal["answered", "escalated"]
```

`state/request_input.py`

```python
from typing import TypedDict


class RequestInput(TypedDict):
    request_text: str
```

`state/request_output.py`

```python
from typing import TypedDict

from .request_status import RequestStatus


class RequestOutput(TypedDict):
    reply: str
    status: RequestStatus
```

`state/request_state.py`

```python
from .request_input import RequestInput
from .request_output import RequestOutput


class RequestState(RequestInput, RequestOutput, total=False):
    category: str
```

### TypeScript

`state/RequestStatus.ts`

```ts
import { z } from "zod/v4";

export const RequestStatus = z.enum(["answered", "escalated"]);
```

`state/RequestInput.ts`

```ts
import { StateSchema } from "@langchain/langgraph";
import { z } from "zod/v4";

export const RequestInput = new StateSchema({
  requestText: z.string(),
});
```

`state/RequestOutput.ts`

```ts
import { StateSchema } from "@langchain/langgraph";
import { z } from "zod/v4";
import { RequestStatus } from "./RequestStatus";

export const RequestOutput = new StateSchema({
  reply: z.string(),
  status: RequestStatus,
});
```

`state/RequestState.ts`

```ts
import { StateSchema } from "@langchain/langgraph";
import { z } from "zod/v4";
import { RequestStatus } from "./RequestStatus";

export const RequestState = new StateSchema({
  requestText: z.string(),
  reply: z.string(),
  status: RequestStatus,
  category: z.string(),
});
```

---

## Routing without if and else

Bloated graphs usually come from a router function that reads five flags and
branches on each. There are three ways out, in order of preference.

### 1. Let the model choose the next node

Give the model's answer a field whose allowed values are node names. The
node returns a `Command` that goes there. No branch is written at all.

`schemas/classification.py`

```python
from typing import Literal

from pydantic import BaseModel, Field


class Classification(BaseModel):
    category: Literal["billing", "technical", "other"]
    next_step: Literal["write_answer", "escalate_request"]
    reason: str = Field(description="One sentence on why")
```

`nodes/classify_request.py`

```python
from typing import Literal

from langgraph.runtime import Runtime
from langgraph.types import Command

from ..context.app_context import AppContext
from ..prompts.make_classify_prompt import make_classify_prompt
from ..schemas.classification import Classification
from ..state.request_input import RequestInput


def classify_request(
    state: RequestInput, runtime: Runtime[AppContext]
) -> Command[Literal["write_answer", "escalate_request"]]:
    model = runtime.context.model.with_structured_output(Classification)
    classification = model.invoke(make_classify_prompt(state["request_text"]))
    return Command(
        update={"category": classification.category},
        goto=classification.next_step,
    )
```

The `Literal` in the return type tells LangGraph where the node can go, so
the graph still draws correctly. Do not also add a plain edge out of a node
that returns `Command`.

TypeScript, `schemas/Classification.ts`:

```ts
import { z } from "zod/v4";

export const Classification = z.object({
  category: z.enum(["billing", "technical", "other"]),
  nextStep: z.enum(["writeAnswer", "escalateRequest"]),
  reason: z.string().describe("One sentence on why"),
});
```

`nodes/makeClassifyRequest.ts`

```ts
import { Command } from "@langchain/langgraph";
import type { BaseChatModel } from "@langchain/core/language_models/chat_models";
import { Classification } from "../schemas/Classification";
import { makeClassifyPrompt } from "../prompts/makeClassifyPrompt";
import { RequestInput } from "../state/RequestInput";

export function makeClassifyRequest(model: BaseChatModel) {
  const classifier = model.withStructuredOutput(Classification);
  return async (state: typeof RequestInput.State) => {
    const classification = await classifier.invoke(makeClassifyPrompt(state.requestText));
    return new Command({
      update: { category: classification.category },
      goto: classification.nextStep,
    });
  };
}
```

In TypeScript the node is made by a function that takes the model, so the
model stays out of state.

### 2. Map a status to a node

When code, not the model, decides, look the answer up instead of branching.

```python
from typing import Literal

NEXT_STEP_BY_STATUS = {
    "found": "score_item",
    "missing": "skip_item",
    "failed": "report_failure",
}


def pick_next_step(state: ItemState) -> Literal["score_item", "skip_item", "report_failure"]:
    return NEXT_STEP_BY_STATUS[state["status"]]
```

```ts
const nextStepByStatus = {
  found: "scoreItem",
  missing: "skipItem",
  failed: "reportFailure",
} as const;

export function pickNextStep(state: typeof ItemState.State) {
  return nextStepByStatus[state.status];
}
```

Adding a new status means adding one line to the map, not a new branch.

### 3. A plain conditional edge

Only when the choice is a single yes or no, and no state update goes with it.

```python
def pick_next_step(state: ReviewState) -> Literal["publish_post", "write_post"]:
    return "publish_post" if state["is_approved"] else "write_post"
```

---

## Nodes return only what changed

```python
def write_answer(state: RequestState) -> RequestOutput:
    return {"reply": make_reply(state["category"]), "status": "answered"}
```

Never this:

```python
def write_answer(state: RequestState) -> RequestState:
    state["reply"] = make_reply(state["category"])
    state["status"] = "answered"
    return state
```

---

## One status field

    good                                bad
    ────────────────────────────────────────────────────────────
    status: "answered"                  is_done: True
    status: "escalated"                 has_error: False
                                        needs_review: True
                                        was_escalated: False

Four flags can be in sixteen combinations, most of them impossible, and
every router has to check all of them. One status has exactly the values
you listed.

---

## Many items at once: Send

Do not loop over items inside a node. Send each item to a worker node. Each
worker gets only its own item, and the results are joined by a reducer.

Python:

```python
from operator import add
from typing import Annotated, TypedDict

from langgraph.types import Send


class BatchState(TypedDict):
    items: list[str]
    results: Annotated[list[ItemResult], add]


class ItemInput(TypedDict):
    item: str


def send_each_item(state: BatchState) -> list[Send]:
    return [Send("check_item", {"item": item}) for item in state["items"]]


def check_item(state: ItemInput) -> dict:
    return {"results": [make_item_result(state["item"])]}
```

```python
builder.add_conditional_edges("load_items", send_each_item, ["check_item"])
builder.add_edge("check_item", "summarise_results")
```

TypeScript:

```ts
import { ReducedValue, Send, StateSchema } from "@langchain/langgraph";
import { z } from "zod/v4";

export const BatchState = new StateSchema({
  items: z.array(z.string()),
  results: new ReducedValue(z.array(ItemResult).default(() => []), {
    reducer: (current, update) => current.concat(update),
  }),
});

export function sendEachItem(state: typeof BatchState.State) {
  return state.items.map((item) => new Send("checkItem", { item }));
}
```

Only keys that more than one node writes at the same time need a reducer.
Everything else is simply replaced.

---

## What stays out of state

| Leave out | Put it here instead |
|---|---|
| Model, database client, API keys | Run context (Python) or the node maker (TypeScript) |
| Tenant id, user id, model name | Run context |
| Large documents, query results, files | The database. Keep the id in state. |
| Anything you can work out from other keys | A small function that works it out |
| Values only one step needs | That node's private input shape |
| Message history, when the graph is not a chat | Nowhere |

State is saved at every step when a checkpointer is on, so every key costs
space on every step. Keep a run's state well under 1 MB.

---

## Run context

Python, `context/app_context.py`:

```python
from dataclasses import dataclass

from langchain_core.language_models import BaseChatModel


@dataclass
class AppContext:
    model: BaseChatModel
    tenant_id: str
```

```python
graph.invoke(
    {"request_text": "My card was charged twice"},
    context=AppContext(model=model, tenant_id="acme"),
)
```

Nodes read it as `runtime.context.model`.

---

## Failures

Put a retry policy on the node that talks to the outside world. The node
itself stays a straight line.

```python
from langgraph.types import RetryPolicy

builder.add_node(classify_request, retry_policy=RetryPolicy(max_attempts=3))
```

```ts
builder.addNode("classifyRequest", classifyRequest, { retryPolicy: { maxAttempts: 3 } });
```

If a failure is a real outcome the user should see, make it a status value,
such as `"failed"`, and route on it like any other.

---

## The graph file

The file that builds the graph reads top to bottom like the steps of the
work. Nothing else goes in it.

Python, `make_request_graph.py`:

```python
from langgraph.graph import END, START, StateGraph
from langgraph.types import RetryPolicy

from .context.app_context import AppContext
from .nodes.classify_request import classify_request
from .nodes.escalate_request import escalate_request
from .nodes.write_answer import write_answer
from .state.request_input import RequestInput
from .state.request_output import RequestOutput
from .state.request_state import RequestState


def make_request_graph(checkpointer=None):
    builder = StateGraph(
        RequestState,
        input_schema=RequestInput,
        output_schema=RequestOutput,
        context_schema=AppContext,
    )
    builder.add_node(classify_request, retry_policy=RetryPolicy(max_attempts=3))
    builder.add_node(write_answer)
    builder.add_node(escalate_request)
    builder.add_edge(START, "classify_request")
    builder.add_edge("write_answer", END)
    builder.add_edge("escalate_request", END)
    return builder.compile(checkpointer=checkpointer)
```

TypeScript, `makeRequestGraph.ts`:

```ts
import { END, START, StateGraph } from "@langchain/langgraph";
import type { BaseCheckpointSaver } from "@langchain/langgraph";
import type { BaseChatModel } from "@langchain/core/language_models/chat_models";
import { makeClassifyRequest } from "./nodes/makeClassifyRequest";
import { writeAnswer } from "./nodes/writeAnswer";
import { escalateRequest } from "./nodes/escalateRequest";
import { RequestInput } from "./state/RequestInput";
import { RequestOutput } from "./state/RequestOutput";
import { RequestState } from "./state/RequestState";

export function makeRequestGraph(model: BaseChatModel, checkpointer?: BaseCheckpointSaver) {
  return new StateGraph({ state: RequestState, input: RequestInput, output: RequestOutput })
    .addNode("classifyRequest", makeClassifyRequest(model), {
      ends: ["writeAnswer", "escalateRequest"],
      retryPolicy: { maxAttempts: 3 },
    })
    .addNode("writeAnswer", writeAnswer)
    .addNode("escalateRequest", escalateRequest)
    .addEdge(START, "classifyRequest")
    .addEdge("writeAnswer", END)
    .addEdge("escalateRequest", END)
    .compile({ checkpointer });
}
```

---

## Checkpointer

Pass it in when compiling. Every run gets a `thread_id`, so it can pause,
resume after a crash, or be replayed.

```python
graph = make_request_graph(checkpointer=saver)
graph.invoke(request, {"configurable": {"thread_id": "request-1042"}}, context=context)
```

```ts
const graph = makeRequestGraph(model, saver);
await graph.invoke(request, { configurable: { thread_id: "request-1042" } });
```

Name thread ids after the thing being worked on: `request-1042`,
`order-5531`, `user-88-chat`. Never a random string with no meaning.

---

## Before you commit

- Does the caller send only the input shape and get back only the output
  shape?
- Does every node return only the keys it changed?
- Is there a router with more than one `if`? Replace it with `Command`, a
  status map, or a model choice.
- Are there two or more yes or no flags that describe the same thing? Make
  them one status.
- Is a model, client, key or large blob in state? Move it out.
- Does the model's answer have a schema?
- Is each node, state shape, schema and prompt in its own file, named after
  it?
- Can someone read the graph file top to bottom and know the steps?

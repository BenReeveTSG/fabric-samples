# Migrating Fabric Data Agent querying from the Assistants API to the Responses API

## Who this is for

If you query a Fabric Data Agent from Python by using the `FabricOpenAI` client
(the one built on the OpenAI **Assistants API**, with `assistants`, `threads`,
`runs`), this guide shows you how to move to the new `FabricOpenAIResponses`
client, which is built on the OpenAI **Responses API**.

The management plane (creating a data agent, adding data sources, selecting
tables, publishing) doesn't change. Only the code that *sends questions and
reads answers* changes. This guide is limited to that part.

## Why this change is happening

The data plane of the Fabric Data Agent SDK is moving from the Assistants API
to the Responses API. The Responses API is a smaller surface: instead of
creating an assistant, creating a thread, posting a message, and starting a
run, you make a single `responses.create(input=...)` call. Follow-up turns are
handled with a response ID or a conversation ID rather than a shared thread.

This guide helps you understand what the Assistant API deprecation means for your workflow so you can make any needed changes before it takes effect.

## The short version

| Task | Assistants API | Responses API |
| --- | --- | --- |
| Create the client | `FabricOpenAI(artifact_name=...)` | `FabricOpenAIResponses(artifact_name=..., workspace_name=..., ai_skill_stage=...)` |
| Set up before asking | `beta.assistants.create(model=...)` then `beta.threads.create()` | Nothing to set up; call `responses.create(...)` directly |
| Ask a question | `beta.threads.messages.create(...)` then `beta.threads.runs.create(...)` | `responses.create(input=...)` |
| Wait for the result | Poll `beta.threads.runs.retrieve(...)` on `run.status` | Poll `responses.retrieve(id)` on the response `status` |
| Read the answer | `beta.threads.messages.list(...)` then `content[0].text.value` | Read output items of type `message`, or `output_text` |
| Continue the conversation | Reuse the same `thread_id` | Pass `previous_response_id=...`, or create a conversation and pass `conversation=...` |
| Inspect intermediate steps | `beta.threads.runs.steps.list(...)` | Read output items of type `function_call`, `function_call_output`, `code_interpreter_call` |
| Stream the answer | Supported by the Assistants API | `responses.stream(...)` with `response.output_text.delta` events |
| Clean up | `beta.threads.delete(...)` | Not required when using `previous_response_id` |
| Run an evaluation | `evaluate_data_agent(df, ...)` | `evaluate_data_agent(df, ..., client_class=FabricOpenAIResponses)` |

## The change, step by step

### 1. Creating the client

Today:

```python
from fabric.dataagent.client import FabricOpenAI

fabric_client = FabricOpenAI(artifact_name=data_agent_name)
```

New:

```python
from fabric.dataagent.client import FabricOpenAIResponses

fabric_client = FabricOpenAIResponses(
    artifact_name=data_agent_name,
    workspace_name=workspace_id or workspace_name,
    ai_skill_stage="sandbox",
)
```

The import name and the class name change. `FabricOpenAIResponses` accepts a
`workspace_name` and an `ai_skill_stage` (for example `"sandbox"` or
`"production"`) so you can target the draft or published version of the data agent.

### 2. Asking one question

By using the Assistants API, you create an assistant and a thread once. Then, for
each question, you post a message and start a run.

```python
assistant = fabric_client.beta.assistants.create(model="gpt-4o")
thread = fabric_client.beta.threads.create()

fabric_client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="What was the best selling product by volume in 2013?",
)
run = fabric_client.beta.threads.runs.create(
    thread_id=thread.id,
    assistant_id=assistant.id,
)
```

By using the Responses API, you don't need an assistant or a thread. One call carries the
question.

```python
response = fabric_client.responses.create(
    input="What was the best selling product by volume in 2013?",
)
```

Note that the new call doesn't have a `model` argument. The SDK injects a default
Responses model, so you don't pass one unless you have a reason.

### 3. Waiting for the answer

By using the Assistants API, you poll the run until its status leaves the running
states.

```python
import time

while run.status in ("queued", "in_progress"):
    run = fabric_client.beta.threads.runs.retrieve(
        thread_id=thread.id,
        run_id=run.id,
    )
    time.sleep(2)
```

By using the Responses API, you poll the response until its status is terminal
(`completed`, `failed`, `incomplete`, or `cancelled`). The response object
carries its own `id` and `status`, so you retrieve it by ID.

```python
response = fabric_client.responses.retrieve(response.id)
```

The Responses sample notebook wraps this process in a `wait_for_response` helper that
also handles the case where the service returns raw streamed text instead of an
object. That helper is worth copying as-is; see `responses-api-notebook.ipynb`.

### 4. Reading the answer

With the Assistants API you list the thread's messages and read the text off the
message content:

```python
messages = fabric_client.beta.threads.messages.list(thread_id=thread.id, order="asc")
for m in messages:
    print(f"{m.role}: {m.content[0].text.value}")
```

With the Responses API the answer lives in the response's `output` items. The
items of type `message` hold the text; there is also a convenience
`output_text` field. The sample notebook's `extract_response_text` helper walks
the `message` items and joins their text, falling back to `output_text`.

### 5. Continuing the conversation

This is the biggest conceptual change. With the Assistants API a conversation is
a thread: you keep posting to the same `thread_id`. With the Responses API there
are two ways to continue, and you pick one:

**a. Chain by previous response ID.** Pass the ID of the last response as
`previous_response_id`. No conversation object is needed.

```python
follow_up = fabric_client.responses.create(
    input="Which month follows the month from your previous answer?",
    previous_response_id=response.id,
)
```

**b. Use a conversation.** Create a conversation once, then pass its ID on every
turn.

```python
conversation = fabric_client.conversations.create()

first = fabric_client.responses.create(
    input="Which month has the most public holidays?",
    conversation=conversation.id,
)
second = fabric_client.responses.create(
    input="Which month follows that one?",
    conversation=conversation.id,
)
```

Both approaches keep context across turns. Chaining by response ID is the
lightest option; a conversation is useful when you want a single ID that groups
a whole exchange.

### 6. Streaming

Streaming is available in the Assistants API as well. With the Responses API you
open a stream and read events as they arrive:

```python
with fabric_client.responses.stream(
    input="Give me the first 10 rows from the public holidays table.",
    previous_response_id=response.id,
) as stream:
    for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="")
```

For table or row requests, the useful result may come back as output items
rather than as final message text, so do not rely on the streamed text alone.

### 7. Inspecting intermediate steps

With the Assistants API you list run steps:

```python
run_steps = fabric_client.beta.threads.runs.steps.list(
    thread_id=thread.id,
    run_id=run.id,
)
```

With the Responses API the same information is in the response's `output` items.
Items of type `function_call` and `function_call_output` carry the tool calls
and their results; `code_interpreter_call` carries executed code. The sample's
*Inspect the intermediate steps* cell turns these into a small DataFrame.

### 8. Cleanup

With the Assistants API you delete the thread when you are done:

```python
fabric_client.beta.threads.delete(thread.id)
```

With the Responses API there is nothing to delete when you chain by
`previous_response_id`. If you created a conversation, that is the object that
represents the exchange.

### 9. Evaluation

`evaluate_data_agent` keeps the same shape. Today it runs against the Assistants
API by default:

```python
from fabric.dataagent.evaluation import evaluate_data_agent

evaluation_id = evaluate_data_agent(
    df,
    data_agent_name,
    workspace_name=workspace_name,
    table_name=table_name,
    data_agent_stage="sandbox",
)
```

To run the same evaluation against the Responses API, pass
`client_class=FabricOpenAIResponses`:

```python
from fabric.dataagent.client import FabricOpenAIResponses
from fabric.dataagent.evaluation import evaluate_data_agent

evaluation_id = evaluate_data_agent(
    df,
    data_agent_name=data_agent_name,
    workspace_name=workspace_name,
    table_name=table_name,
    data_agent_stage="sandbox",
    client_class=FabricOpenAIResponses,
)
```

Either way, results land in two tables: `<table_name>` for the row-level results
and `<table_name>_steps` for the per-step detail. On the Responses path the
step table records the function calls and tool outputs (columns such as
`function_names`, `function_queries`, `function_outputs`, `sql_steps`,
`dax_steps`, `kql_steps`).

## Behavioral differences to keep in mind

- **No assistant or thread objects.** The setup calls
  (`assistants.create`, `threads.create`) don't have equivalents. A response is
  self-contained.
- **`model` is optional.** The Responses calls in the sample omit `model` and
  rely on the SDK's default. You don't need to pin `gpt-4o` as you did when
  creating an assistant.
- **Two ways to keep context.** `previous_response_id` and `conversation` both
  work. Pick one per exchange rather than mixing them.
- **Results might not be plain text.** Especially for table and row requests, the
  answer can arrive as streamed output items. Check the output items, not only
  the final text.
- **Evaluation is opt-in.** `evaluate_data_agent` still defaults to the
  Assistants API. You must pass `client_class=FabricOpenAIResponses` to test the
  Responses path.

## Migration checklist

1. Update the import and client: `FabricOpenAI` becomes `FabricOpenAIResponses`.
1. Remove `assistants.create` and `threads.create`.
1. Replace `messages.create` + `runs.create` with a single `responses.create(input=...)`.
1. Replace run-status polling with response-status polling (`responses.retrieve`).
1. Replace `messages.list` + `content[0].text.value` with reading `output` items / `output_text`.
1. Replace thread reuse with `previous_response_id` or a `conversation`.
1. Replace `runs.steps.list` with reading `function_call` / `function_call_output` / `code_interpreter_call` items, if you use step detail.
1. Remove `threads.delete`.
1. If you evaluate, add `client_class=FabricOpenAIResponses`.
1. Run `responses-api-notebook.ipynb` in this folder to confirm the answers match before you switch.
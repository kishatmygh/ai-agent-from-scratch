# Agent SDK Quick Reference

A quick reference guide to the classes and data structures behind an agent loop, sourced from Anthropic's Python SDK (`anthropic` 2.3.0).
The worked example throughout is the TimeToRead Agent in [`Agent-TimeToRead.ipynb`](Agent-TimeToRead.ipynb).

> [!IMPORTANT]
> **Strongly recommended: read this guide alongside the code.** The agent's logic (the loop, the tools, the memory) is simple to describe, but implementing it means using Anthropic's Python SDK, and the SDK's classes and syntax are not obvious from the code alone. If you're new to the SDK, you can follow the code without this knowledge, but you won't understand it well enough to reproduce it yourself.
>
> This guide makes that link between the logic and the code: what each SDK object is, what it contains and how the code reads it. For anything beyond this example, see the official [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python) and the [Claude API documentation](https://platform.claude.com/docs).

## Contents

0. [The example task: the TimeToRead Agent](#0-the-example-task-the-timetoread-agent)
1. [The big picture: from API to your code](#1-the-big-picture-from-api-to-your-code)
2. [The `Anthropic` class (the client)](#2-the-anthropic-class-the-client)
3. [The `Message` object (`response`)](#3-the-message-object-response)
4. [`stop_reason` values](#4-stop_reason-values)
5. [Content blocks (`block`)](#5-content-blocks-block)
6. [Structures you build yourself](#6-structures-you-build-yourself)
7. [Why "content" appears three times](#7-why-content-appears-three-times)
8. [Reading code: brackets and dots](#8-reading-code-brackets-and-dots)
9. [How to explore on your own](#9-how-to-explore-on-your-own)

---

## 0. The example task: the TimeToRead Agent

**Goal:** "How long would it take to read this text at 200 words per minute?" The agent has two tools and decides when to use each one.

| Tool name (schema and `TOOLS` key) | Python function | Input | Returns |
|---|---|---|---|
| `"count_words"` | `count_words(args)` | `{"text_body": "..."}` | word count as a `str`, e.g. `"16"` |
| `"calculate"` | `calculate(args)` | `{"expression": "..."}` | result as a `str`, e.g. `"4.8"` |

### The saved run (each step is one call to the model)

```text
Step 0  model → tool_use: count_words({"text_body": "I love GenAI. ..."})   your code → "16"     tokens in/out: 694/81
Step 1  model → tool_use: calculate({"expression": "16 / 200"})             your code → "0.08"   tokens in/out: 781/52
Step 2  model → tool_use: calculate({"expression": "0.08 * 60"})            your code → "4.8"    tokens in/out: 841/54
Step 3  model → end_turn: "... would take about 4.8 seconds."               run_agent returns    tokens in/out: 903/84
```

*This is the run saved in the notebook (total cost: $0.009). The model chooses its own steps, so another run may differ, for example by doing the maths in one calculation. Notice that input tokens grow every step, because the full history is resent each time.*

### Where each part lives in the code

| Name in the code | What it is |
|---|---|
| `client` | the `Anthropic` client ([Section 2](#2-the-anthropic-class-the-client)) |
| `call_llm(messages)` | wraps `client.messages.create(...)` and returns a `Message` |
| `response` | the `Message` object from one call ([Section 3](#3-the-message-object-response)) |
| `block` | one item in `response.content` ([Section 5](#5-content-blocks-block)) |
| `messages`, `tool_results`, `TOOLS`, `TOOL_SCHEMA`, `step_log` | plain lists and dicts you build ([Section 6](#6-structures-you-build-yourself)) |
| `run_tool(name, inputs)` | looks up `TOOLS[name]` and runs it; any failure becomes an `"Error..."` string |
| `run_agent(goal)` | the loop: Perceive → Plan → Act → Adapt. Returns a `dict` with the answer, status and metrics |

---

## 1. The big picture: from API to your code

**Step 1: the API sends back plain JSON text.** Values such as `"tool_use"` come from Anthropic's server. This is step 0 of the saved run (IDs shortened):

```json
{
  "id": "msg_01XYZ",
  "type": "message",
  "role": "assistant",
  "model": "claude-sonnet-5-5",
  "stop_reason": "tool_use",
  "content": [
    {"type": "tool_use", "id": "toolu_01A", "name": "count_words",
     "input": {"text_body": "I love GenAI. Building agents from scratch is so enlightening. ..."}}
  ],
  "usage": {"input_tokens": 694, "output_tokens": 81}
}
```

There's no thinking block here: the model decides how much to think, and it skipped thinking on these easy steps. On a harder step, a `{"type": "thinking", ...}` block would come first in `content`.

**Step 2: the SDK turns the JSON into Python objects**, using predefined classes (built with Pydantic) in `anthropic/types/`.

**Step 3: each block's `"type"` value picks its class.** `"text"` becomes a `TextBlock`, `"thinking"` becomes a `ThinkingBlock`, and `"tool_use"` becomes a `ToolUseBlock`.

```text
Anthropic server ──JSON──►  SDK parses it (Pydantic)  ──►  Message object (response)
                                                            ├── .stop_reason  ← StopReason Literal
                                                            └── .content      ← List[ContentBlock]
                                                                 └── each block's class chosen by its "type"
```

### The full tree

```text
anthropic                        module (the library you imported)
└── Anthropic                    class (template for a client)
    └── client                   object, made by anthropic.Anthropic(max_retries=3)
        └── .messages            attribute → Messages object
            └── .create(...)     method → sends the request, RETURNS a Message
                  │                (called inside call_llm)
                  ▼
response                         object of class Message
├── .id                          str   e.g. "msg_01XYZ..."
├── .role                        str   always "assistant"
├── .model                       str   e.g. "claude-sonnet-5-5"
├── .stop_reason                 str   "tool_use", "end_turn", ... (Section 4)
├── .usage                       object of class Usage
│   ├── .input_tokens            int
│   └── .output_tokens           int   includes thinking tokens
└── .content                     list  of block objects
    ├── ThinkingBlock            object
    │   ├── .type                str   "thinking"
    │   ├── .thinking            str   summary of the reasoning
    │   └── .signature           str   verification value, don't edit
    ├── TextBlock                object
    │   ├── .type                str   "text"
    │   └── .text                str   Claude's words
    └── ToolUseBlock             object
        ├── .type                str   "tool_use"
        ├── .id                  str   "toolu_01A..."
        ├── .name                str   "count_words" or "calculate"
        └── .input               dict  {"text_body": "..."} or {"expression": "16 / 200"}
```

### Class vs. object: which is which in this example

A **class** is the template; an **object** (or *instance*) is one thing made from it. Your code never writes these classes. The SDK creates the objects for you, except `client`, which you make yourself.

| Class (the template) | Object in your code (an instance of it) | How it's made |
|---|---|---|
| `Anthropic` | `client` | you call the class: `anthropic.Anthropic(max_retries=3)` |
| `Messages` | `client.messages` | created inside `client` |
| `Message` | `response` | returned by `client.messages.create(...)` in `call_llm` |
| `Usage` | `response.usage` | part of `response` |
| `ToolUseBlock` / `TextBlock` / `ThinkingBlock` | `block` (one at a time in the loop) | items in `response.content` |
| `list` (built into Python) | `messages`, `step_log`, `tool_results` | you create them: `[]` |
| `dict` (built into Python) | `TOOLS`, `step_entry`, the result from `run_agent` | you create them: `{}` |

The last two rows matter: your own lists and dicts are objects too. They're instances of Python's built-in classes, which is why `messages.append(...)` works. `append` is a method of the `list` class.

---

## 2. The `Anthropic` class (the client)

Defined in `anthropic/_client.py`. Built on `SyncAPIClient`, which handles the HTTP connection.

```python
import anthropic                                # module
client = anthropic.Anthropic(max_retries=3)     # class → object (finds your key, sets defaults)
response = client.messages.create(...)          # object → resource → method → returns a Message
```

`Anthropic()` doesn't contact Claude. It only prepares the client. The request is sent when `.create(...)` runs.

### Settings you can pass: `Anthropic(setting=value)`

| Setting | Type | Default | What it does | In the example? |
|---|---|---|---|---|
| `api_key` | `str` | read from `ANTHROPIC_API_KEY` | your secret key | ✓ `load_dotenv()` loads it from `.env` |
| `max_retries` | `int` | 2 | retries for rate limits, server errors and dropped connections | ✓ set to 3 |
| `timeout` | `float` (seconds) or `Timeout` | 10 minutes | how long to wait before giving up | default |
| `base_url` | `str` | `https://api.anthropic.com` | where requests are sent | rarely |
| `default_headers`, `default_query` | `dict` | `None` | extra headers or query values on every request | rarely |
| `auth_token`, `credentials`, `profile`, `config` | various | `None` | other ways to log in | ✗ advanced |
| `http_client`, `middleware`, `open_telemetry`, `webhook_key` | various | `None` | networking, monitoring, webhooks | ✗ advanced |

Pass only **one** login method, otherwise you get a `ValueError`. If every retry fails, `.create()` raises an error that inherits from `anthropic.APIError`. `run_agent` catches it and returns the status `"api_error"`.

### Parameters the example passes to `.create()` (in `call_llm`)

| Parameter | Value in the example | What it does |
|---|---|---|
| `model` | `MODEL` = `"claude-sonnet-5-5"` | which Claude model answers |
| `max_tokens` | `MAX_TOKENS` = 16000 | upper limit on output tokens for one response, thinking included |
| `thinking` | `{"type": "adaptive", "display": "summarized"}` | the model decides how much to think; a **summary** of its reasoning is returned |
| `tools` | `TOOL_SCHEMA` | the tool descriptions the model can choose from |
| `system` | `SYSTEM` | instructions that apply to the whole conversation |
| `messages` | `messages` | the full conversation so far (resent every call) |

### Attributes (data on the client)

| Attribute | Type | Holds |
|---|---|---|
| `client.api_key` | `str` / `None` | the key it found (avoid printing it, because it's secret) |
| `client.auth_token` | `str` / `None` | alternative login token |
| `client.default_headers` | `dict` | headers sent with each request |
| `client.auth_headers` | `dict` | login headers built from your key |

### Resources (groups of API features)

| Resource | Type | What it's for | Use it? |
|---|---|---|---|
| `client.messages` | `Messages` | talking to Claude | ✓ every agent |
| `client.models` | `Models` | listing the available models | handy |
| `client.files` | `Files` | uploading files for Claude to read | later |
| `client.beta` | `Beta` | features still in testing | later |
| `client.skills`, `.organization`, `.user_profiles` | various | skills, account and admin features | ✗ |
| `client.with_raw_response`, `.with_streaming_response` | wrappers | access to the raw HTTP response | ✗ advanced |

### Methods inside `client.messages`

| Method | What it does |
|---|---|
| `.create(...)` | sends messages and returns a `Message`. **Your agent's main call.** |
| `.stream(...)` | same, but returns the reply piece by piece as it's written |
| `.parse(...)` | same as `.create`, but checks the reply against a structure you define |
| `.count_tokens(...)` | token count for a request, without sending it |
| `.batches` | sends many requests at once, at lower cost, processed in the background |

### Methods on the client itself

| Method | Returns | What it does |
|---|---|---|
| `client.copy(...)` / `client.with_options(...)` | new client | copy with changed settings, e.g. `client.with_options(timeout=60)` |
| `client.with_middleware(...)` | new client | adds code that runs on every request (advanced) |
| `client.close()` | `None` | closes the network connection |

---

## 3. The `Message` object (`response`)

Defined in `anthropic/types/message.py`. This is what `client.messages.create(...)` returns, and what `call_llm` hands back as `response`.

```python
class Message(BaseModel):
    id: str
    type: Literal["message"]
    role: Literal["assistant"]
    model: Model
    content: List[ContentBlock]                     # ← response.content
    stop_reason: Optional[StopReason]               # ← response.stop_reason
    stop_details: Optional[RefusalStopDetails]      # filled only for a refusal
    stop_sequence: Optional[str]
    usage: Usage                                    # ← response.usage
    # plus advanced optional fields: container, diagnostics
```

| Field | Type | Meaning | Used in the example |
|---|---|---|---|
| `id` | `str` | unique ID for this reply | ✗ |
| `type` | `"message"` | always "message" | ✗ |
| `role` | `"assistant"` | always "assistant" | ✗ |
| `model` | `str` | which model answered | ✗ |
| `content` | `list` of blocks | what Claude thought, said or asked for ([Section 5](#5-content-blocks-block)) | ✓ appended to `messages` |
| `stop_reason` | `str` (or `None` while streaming) | why Claude stopped ([Section 4](#4-stop_reason-values)) | ✓ decides the next move |
| `stop_details` | object or `None` | category and explanation when `stop_reason` is `"refusal"` | ✗ |
| `stop_sequence` | `str` or `None` | the custom stop word hit, if any | ✗ |
| `usage` | `Usage` object | token counts, used for billing and limits | ✓ `step_log` and cost |

### The `Usage` object (`response.usage`): `anthropic/types/usage.py`

| Attribute | Type | Holds |
|---|---|---|
| `.input_tokens` | `int` | tokens the model **read**: system prompt, tool schema and the whole `messages` history |
| `.output_tokens` | `int` | tokens the model **wrote**, thinking included. This is the billed total |
| `.cache_read_input_tokens`, `.cache_creation_input_tokens` | `int` or `None` | prompt-caching counts (not used in the example) |
| `.output_tokens_details`, `.service_tier`, `.speed`, ... | optional | extra detail for monitoring |

Method: `response.to_param()` converts the reply into the format used for an input message.

---

## 4. `stop_reason` values

Defined in `anthropic/types/stop_reason.py` as a `Literal`, which means the value must be exactly one of these strings.

```python
StopReason = Literal[
    "end_turn", "max_tokens", "stop_sequence", "tool_use",
    "pause_turn", "refusal", "model_context_window_exceeded",
]
```

| Value | Meaning | What `run_agent` does |
|---|---|---|
| `"tool_use"` | Claude wants a tool run | runs the tools, appends `tool_results`, loops again |
| `"end_turn"` | Claude finished its answer | returns the text with status `"finished"` |
| `"max_tokens"` | hit `MAX_TOKENS`, so the answer is cut off | returns status `"max_tokens"` |
| `"refusal"` | Claude declined the request | returns status `"refusal"` |
| `"stop_sequence"` | hit a custom stop word you set | not expected: the example sets no `stop_sequences` |
| `"pause_turn"` | a long server-side tool task paused | not expected: the example uses no server-run tools |
| `"model_context_window_exceeded"` | conversation too long for the model | not expected: the run is short |

The last three fall into the `!= "tool_use"` branch, so they would be reported as `"finished"`. Handle them explicitly if a future version uses those features.

---

## 5. Content blocks (`block`)

`response.content` is a **list**. Each item is a block **object**. `block` in your loop is just a variable that holds one of them at a time.

### Where block types are defined: `anthropic/types/content_block.py`

```python
ContentBlock = Annotated[
    Union[
        TextBlock,                         # type = "text"        ← the example reads these
        ThinkingBlock,                     # type = "thinking"    ← the example reads these
        RedactedThinkingBlock,             # type = "redacted_thinking"
        ToolUseBlock,                      # type = "tool_use"    ← the example reads these
        ServerToolUseBlock,                # Anthropic-run tools (web search, code execution...)
        WebSearchToolResultBlock,
        WebFetchToolResultBlock,
        CodeExecutionToolResultBlock,
        BashCodeExecutionToolResultBlock,
        TextEditorCodeExecutionToolResultBlock,
        ToolSearchToolResultBlock,
        ContainerUploadBlock,
    ],
    UnionDiscriminator("type"),            # ← "use the 'type' field to decide which class"
]
```

### `ToolUseBlock`: `anthropic/types/tool_use_block.py`

```python
class ToolUseBlock(BaseModel):
    type: Literal["tool_use"]
    id: str
    name: str
    input: Dict[str, object]
    caller: Optional[Caller] = None        # advanced
    toolset_name: Optional[str] = None     # advanced
```

| Attribute | Type | Holds | Used in the example |
|---|---|---|---|
| `.type` | `str` | always `"tool_use"` | ✓ `if block.type == "tool_use"` |
| `.id` | `str` | unique request ID, e.g. `"toolu_01A9..."` | ✓ becomes `"tool_use_id"` |
| `.name` | `str` | tool requested: `"count_words"` or `"calculate"` | ✓ `run_tool(block.name, ...)` |
| `.input` | `dict` | arguments, e.g. `{"expression": "16 / 200"}` | ✓ passed to the tool as `args` |
| `.caller`, `.toolset_name` | optional | usually `None` | ✗ ignore |

### `TextBlock`: `anthropic/types/text_block.py`

```python
class TextBlock(BaseModel):
    type: Literal["text"]
    text: str
    citations: Optional[List[TextCitation]] = None
```

| Attribute | Type | Holds | Used in the example |
|---|---|---|---|
| `.type` | `str` | always `"text"` | ✓ filter text |
| `.text` | `str` | Claude's words | ✓ the final answer and the step log |
| `.citations` | optional list | sources, when citing documents | ✗ ignore |

### `ThinkingBlock`: `anthropic/types/thinking_block.py`

```python
class ThinkingBlock(BaseModel):
    type: Literal["thinking"]
    thinking: str
    signature: str
```

| Attribute | Type | Holds | Used in the example |
|---|---|---|---|
| `.type` | `str` | always `"thinking"` | ✓ filter thinking |
| `.thinking` | `str` | with `display: "summarized"`, a **summary** of the reasoning (empty if display is omitted) | ✓ `step_entry["thinking"]` |
| `.signature` | `str` | proves the block came from Claude | ✗ never edit it |

> [!NOTE]
> **Thinking blocks must go back unchanged.** The example appends the whole `response.content` to `messages`, so they do. Removing or editing them would break the next request. The summary is useful for the audit trail, but it is not a word-for-word record. The exact record is the tool calls, inputs, outputs and stop reasons.

### Methods on every block (inherited from `BaseModel`)

| Method | Returns |
|---|---|
| `block.to_dict()` | the block as a plain `dict` |
| `block.to_json()` | the block as a JSON string |
| `block.model_dump()` | same as `to_dict()` |

### Coding against blocks (from `run_agent`)

```python
# Collect by type: one list per kind of block
"thinking": [b.thinking for b in response.content if b.type == "thinking"],
"text":     [b.text     for b in response.content if b.type == "text"],

# Act on tool requests
for block in response.content:
    if block.type == "tool_use":
        output = run_tool(block.name, block.input)    # ToolUseBlock → .name .input
```

Always check `.type` first. Reading `block.text` on a `ToolUseBlock` raises an `AttributeError`, because that attribute only exists on `TextBlock`. Position isn't reliable either: with thinking on, `response.content[0]` is often a `ThinkingBlock`, not text.

---

## 6. Structures you build yourself

These are plain Python lists and dicts. No special classes are involved.

```text
messages                         list  (conversation history = the agent's "memory")
├── {"role": "user",      "content": goal}               content = str
├── {"role": "assistant", "content": response.content}   content = list of blocks
└── {"role": "user",      "content": tool_results}       content = list of dicts
    ...and the assistant/user pair repeats for each step

tool_results                     list
└── {"type": "tool_result",                               dict
     "tool_use_id": "toolu_01A...",   ← must match block.id
     "content": "16",                 ← the tool's output (str)
     "is_error": False}               ← True if output starts with "Error"

TOOLS                            dict
├── "count_words" → count_words  key = str, value = function
└── "calculate"   → calculate

TOOL_SCHEMA                      list of dicts  (sent to Claude so it knows the tools)
└── {"name": "calculate",                     ← must match the TOOLS key
     "description": "Evaluates an arithmetic expression...",
     "input_schema": {"type": "object",
                      "properties": {"expression": {"type": "string", ...}},
                      "required": ["expression"]}}
```

The SDK's own definition of a tool result is `ToolResultBlockParam` in `anthropic/types/tool_result_block_param.py`. It's a `TypedDict`, so you write it as a plain dict. `type` and `tool_use_id` are required, and `content` and `is_error` are optional.

### Metrics: built during the run, returned at the end

```text
step_log                         list  (one entry per model call)
└── step_entry                   dict  (step 0 of the saved run)
    ├── "step": 0                              int
    ├── "input_tokens":  694                   int   ← response.usage.input_tokens
    ├── "output_tokens": 81                    int   ← response.usage.output_tokens
    ├── "stop_reason": "tool_use"              str
    ├── "thinking": []                         list of str  (empty: no thinking on this step)
    ├── "text": []                             list of str  (empty: the model only called a tool)
    └── "tool_calls": [{"tool": "count_words", "input": {...}, "output": "16"}]

run_agent(goal) returns          dict  (made by build_result)
├── "answer", "status"                          str    status: finished | max_tokens | refusal | max_steps | api_error
├── "steps"                                     list   ← step_log, used by print_report and plot_metrics
├── "total_input_tokens", "total_output_tokens" int
└── "total_cost"                                float  ← token_cost(...)
```

---

## 7. Why "content" appears three times

| Where | What it is | Read it with |
|---|---|---|
| `response.content` | attribute of the `Message` object → list of blocks | a dot, because it's an object |
| `{"role": ..., "content": ...}` in `messages` | key in a dict you made | `messages[0]["content"]`, because it's a dict |
| `{"type": "tool_result", "content": output}` in `tool_results` | key in a dict you made → the tool's output | `tool_results[0]["content"]`, because it's a dict |

---

## 8. Reading code: brackets and dots

### Creating vs. reading

| | List | Dict | Object (from the SDK) |
|---|---|---|---|
| **Create it** | `step_log = []` (square brackets) | `TOOLS = {"calculate": calculate}` (curly brackets) | `client = anthropic.Anthropic()` (call the class) |
| **Read a value** | `result["steps"][0]` (square brackets + position) | `TOOLS["calculate"]` (square brackets + key) | `response.stop_reason` (dot + attribute name) |

Curly brackets `{ }` only **create** a dict. Reading from a list **or** a dict always uses square brackets `[ ]`. Reading from an object uses a dot.

### Example chain: `response.usage.input_tokens`

| Piece | Data type |
|---|---|
| `response` | `Message` object |
| `.usage` | attribute (dot) → a `Usage` object |
| `.input_tokens` | attribute of that object (dot) → an `int` |

### Example chain: `result["steps"][0]["tool_calls"]`

| Piece | Data type |
|---|---|
| `result` | `dict` returned by `run_agent` |
| `["steps"]` | value for key "steps" → a `list` (`step_log`) |
| `[0]` | first item (position 0) → a `dict` (one `step_entry`) |
| `["tool_calls"]` | value for key "tool_calls" → a `list` of dicts |

- **Objects from the SDK** (`client`, `response`, blocks): read with **dots**, as in `response.stop_reason` or `block.name`.
- **Lists and dicts you build** (`messages`, `tool_results`, `TOOLS`, `TOOL_SCHEMA`, `step_log`): read with **square brackets**, as in `messages[0]` (list, by position) or `TOOLS["calculate"]` (dict, by key).
- **The one crossover:** `block.input` is an attribute (dot) whose value is a dict (square brackets). That's why the tools read `args["text_body"]` and `args["expression"]`: `args` *is* `block.input`.

---

## 9. How to explore on your own

| Do this | To learn |
|---|---|
| `print(type(x))` | what data type or class `x` is |
| `print(x)` | what's inside `x` |
| `print(block.to_dict())` | every attribute and value of a block |
| `print(response.content)` after `call_llm(messages)` | what Claude sent back each step |
| `print_report(goal, result)` | the step-by-step trace the example already records |
| **Ctrl+click** a name in VS Code or Cursor (or right-click the name → Go to Definition) | jumps to its real definition in the installed SDK |
| `python -m pip show anthropic` (terminal) | your installed SDK version |

> [!TIP]
> **Ctrl+click trick:** temporarily add `from anthropic.types import Message, ToolUseBlock, ThinkingBlock, StopReason` to a file, then Ctrl+click any of the names (or right-click the name → Go to Definition) to open its class. From there, keep Ctrl+clicking (`ContentBlock`, `Usage` and so on) to explore.

> [!NOTE]
> **Version note:** this guide was checked against the installed SDK, `anthropic` 2.3.0. Other versions may add or rename advanced items. Everything the TimeToRead Agent uses is part of the core API.

---

**Sources (anthropic-sdk-python on GitHub):**
[`_client.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/_client.py) ·
[`types/message.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/message.py) ·
[`types/usage.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/usage.py) ·
[`types/stop_reason.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/stop_reason.py) ·
[`types/content_block.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/content_block.py) ·
[`types/tool_use_block.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/tool_use_block.py) ·
[`types/text_block.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/text_block.py) ·
[`types/thinking_block.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/thinking_block.py) ·
[`types/tool_result_block_param.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/types/tool_result_block_param.py)

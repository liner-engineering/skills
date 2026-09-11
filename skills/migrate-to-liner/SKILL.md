---
name: migrate-to-liner
description: Migrate an existing OpenAI integration, on Chat Completions or the Responses API, to the Liner Model API (liner-mark-1.0), showing a cost comparison before touching any code and auditing every call site for parameters Liner does not support. Use this whenever someone wants to switch, port, try, or evaluate Liner as their LLM provider, mentions liner-mark or platform.liner.com, asks what Liner would cost compared to their current OpenAI/Anthropic/Gemini bill, or asks whether Liner is a drop-in replacement for code they already have. Also use it when a repository already calls an OpenAI chat completions or responses endpoint and the user asks about benchmarking or pricing another provider, even if they never say the word "migrate".
license: MIT
---

# Migrate to the Liner Model API

Liner Model API speaks both OpenAI formats: Chat Completions at `/chat/completions`
and the Responses API at `/responses`. For most projects the
migration is three values: base URL, API key, model name. The work that actually
matters is everything around those three values, because "OpenAI-compatible" is
never 100% compatible. Liner names dropped fields in the
`x-liner-ignored-parameters` response header, so the gaps are visible if you
look, and the ones that hurt are the ones nobody looks at.

## What this skill covers

This skill migrates OpenAI clients, whether they call Chat Completions or the
Responses API. A project calling the Anthropic
or Gemini SDK directly is out of scope: the request shape, the response shape,
the streaming events and the tool schemas all differ, and rewriting them is not
a base-URL change. Say that plainly and stop, rather than producing a rewrite
that looks finished and is not.

The exception is a project that already goes through LangChain, LlamaIndex, the
Vercel AI SDK, LiteLLM or Instructor. Those reach Liner through a provider
setting whichever model they run today, so they are in scope even when the model
named in the config is a Claude or Gemini one.

Run this in the order below. Do not skip step 2, and do not edit code before the
user has seen step 3.

## What you are optimizing for

The person running this has working code and a working provider. They are not
asking you to prove Liner is good. They are asking two questions:

1. Would this break anything?
2. Would this save money?

Answer both honestly, including when the answer is "yes, this would break your
retry logic" or "no, this would not save you much". A migration that ships and
then breaks in production costs the user more than the migration saved. If you
find a blocker, say so plainly and stop. Reporting a blocker is a successful
outcome for this skill.

## Step 1 — Find every call site

Search the repository for the current LLM integration. Look for:

- SDK clients: `OpenAI(`, `AsyncOpenAI(`, `new OpenAI(`, `openai.ChatCompletion`, `client.responses.create`
- Raw HTTP: `chat/completions`, `/v1/responses`, `api.openai.com`, `generativelanguage.googleapis.com`, `api.anthropic.com`
- Config: `OPENAI_API_KEY`, `base_url`, `baseURL`, `OPENAI_BASE_URL`, model names in env files, YAML, or constants
- Frameworks that wrap the client: LangChain, LlamaIndex, Vercel AI SDK, LiteLLM, Instructor

Build a list of call sites. For each one record the model name, every request
parameter passed, and how the response is read. **How the response is read
matters as much as the request** — code that does `response.choices[1]` or
relies on a JSON schema will break in ways the request parameters alone do not
reveal.

Note which of the two OpenAI formats each call site uses. Step 2 has a separate
table for Responses API call sites.

Report the list before moving on. If the project has more than one provider or
more than one model tier, ask which ones are in scope rather than assuming all
of them.

## Step 2 — Compatibility audit

This is the part that earns the user's trust. Liner does not handle unsupported
parameters uniformly. Some are rejected, some work, and some return `200 OK` and
are ignored. The third group is dangerous because nothing in the response says
anything went wrong.

Verified against production on 2026-09-11.

| In the current code | What Liner does | Your action |
| --- | --- | --- |
| `seed` | Accepted, not applied, and named in the `x-liner-ignored-parameters` response header | **Blocker for reproducibility.** The call succeeds, so nothing in the body tells you the seed was dropped. The header does. Tests and golden-file comparisons pinned to a seed have to change before this call site moves. |
| `n` greater than 1 | Rejected | **Blocker.** Only one completion per request. Either drop the dependency on multiple choices, or leave this call site on its current provider. |
| `response_format` (`json_object` and `json_schema`) | Supported, and rejected with a `400` only when the messages never mention JSON | Safe. OpenAI applies the same rule, so a project moving across already satisfies it. If it does not, add the word to the prompt rather than dropping the field. |
| Image parts as base64 data URLs (`data:image/png;base64,...`) | Supported: PNG, JPEG, WEBP, HEIC, HEIF, up to 10 images and 20 MB of decoded image data per request | Safe, subject to the three conditions below the table. |
| Image parts as remote URLs (`https://...`) | Rejected; remote images are not fetched | **Blocker until rewritten.** Fetch the bytes in the caller and send a base64 data URL instead. |
| Audio or file parts | Rejected | **Blocker.** |
| `max_tokens` | Returns `200` and the value is honored | Safe. Prefer rewriting to `max_completion_tokens`, which is the documented field, unless the call path is shared with other providers that accept only `max_tokens`. |
| `stop` | Rejected | **Blocker.** Stop sequences are common, so look for them early. The usual fix is moving the truncation into the caller; otherwise leave the call site where it is. |
| `logprobs`, `top_logprobs` | Rejected | **Blocker.** Classifiers and eval harnesses that read token probabilities cannot move. |
| `logit_bias` | Rejected | **Blocker.** |
| `store: true` | Rejected, saying completions are not persisted | **Blocker.** Nothing is retained server-side, so a project reading its history back needs its own logging first. |
| `prediction` | Rejected | **Blocker.** |
| `web_search_options` | Rejected | **Blocker.** |
| `functions`, `function_call` (the pre-2024 form) | Rejected | **Blocker,** but usually an easy one: rewrite to `tools` and `tool_choice`, which are supported. |
| Any field not in the supported list below | Undefined | Treat as a blocker. Test it against a real key before trusting it. |

**Image requests carry three conditions.** `reasoning_effort: "none"`,
`parallel_tool_calls: false` and `tools[].defer_loading: true` are each rejected
when the request contains an image, so a call site that sets any of them needs
that value changed on its image path. Streaming, function calling and
structured output all work with images.

For each blocker, show the user the file and line, what breaks, and what the fix
would be. Then leave the call site alone. Partial migration is a good result:
move the call sites that are clean, list the ones that are not, and let the user
decide.

**Supported request fields:** `model`, `messages` (string content, or a parts
array of text and base64 image parts), `stream`, `stream_options.include_usage`,
`max_completion_tokens`, `reasoning_effort` (`none`, `low`, `medium` default,
`high`, `max`), `tools`, `tool_choice`, `parallel_tool_calls`, `temperature`,
`top_p`, `presence_penalty`, `frequency_penalty`.

`user`, `metadata` and `service_tier` are accepted and then dropped, each named
in `x-liner-ignored-parameters`. Leave them in place: nothing breaks, they
simply have no effect. Say so if the project reads `user` back for abuse
tracking or per-seat attribution.

### Call sites on the Responses API

`POST https://platform.liner.com/api/v1/responses` accepts the Responses request
shape and returns a `response` object, so the OpenAI SDKs' `responses` client
works with the same three values. Streaming emits the standard `response.*`
events and ends with `response.completed`. Like Chat Completions, it is
stateless.

| In the current code | What Liner does | Your action |
| --- | --- | --- |
| `instructions`, `max_output_tokens`, `reasoning.effort`, `text.format`, function `tools`, `input_image` parts | Supported | Safe. |
| `previous_response_id` | Rejected | **Blocker.** Resend the full conversation in `input` on every turn instead. |
| `store: true` | Rejected | **Blocker.** Drop it. Nothing is kept server-side, so history the project reads back has to live in its own storage. |
| Built-in tools: `web_search`, `file_search`, `code_interpreter` | Rejected; only `function` tools are accepted | **Blocker.** Leave the call site on its current provider, or replace the built-in tool with a function tool the project implements. |
| `reasoning.summary`, `include`, `text.verbosity`, `client_metadata` | Rejected | Drop them. Liner returns neither reasoning summaries nor encrypted reasoning, so nothing downstream reads them. |
| `input` items of type `reasoning`, replayed from an earlier OpenAI response | Rejected; only `message`, `function_call` and `function_call_output` items are accepted | Strip them from stored history before replaying it. Liner does not return reasoning items, so this only affects history saved before the migration. |

### Breakage that is not a request parameter

Two things reliably break a migration and neither of them appears when you read
request payloads, so look for both explicitly:

- **Tokenizer lookups keyed on the model name.** `tiktoken.encoding_for_model("liner-mark-1.0")`
  raises `KeyError`, usually at import time, which kills the process before it
  serves anything. Decouple the encoding from the API model name rather than
  deleting the token accounting that depends on it.
- **Call sites whose output is compared over time.** Benchmark judges, eval
  scorers, golden-file tests and anything whose numbers are tracked across runs
  fail quietly when the model changes. They keep producing results, and the
  results are simply no longer comparable to the ones already stored. That is a
  measurement decision rather than a code decision, so surface it and let the
  user make the call.
- **Model names hardcoded away from the config constant.** A project that
  defines a `MODEL` constant often still has a literal `model="gpt-4o"` at a
  second call site, most often the follow-up request after a tool result. Grep
  for the literal string, not only for the constant.

Streaming (SSE), function calling including parallel and streamed tool call
deltas, and prompt caching all work as documented. The server is stateless, so
conversation history must be resent on every request, same as OpenAI.

These message shapes were checked against production on 2026-09-07 and are all
accepted: an `assistant` turn arriving before any `user` turn, an `assistant`
message with `content` omitted or set to `null` alongside `tool_calls`, and a
`tool` message carrying a `name` key. Those are what the OpenAI SDKs emit during
a tool round trip, so a tool-using project needs no reshaping here.

## Step 3 — Cost comparison, then ask

Do this before editing anything. The user should approve the change with a
number in front of them.

Get real token volume rather than guessing. In order of preference:

1. The provider's usage dashboard or billing export, if the user can paste it
2. Application logs or a usage table, if the project records token counts
3. A representative sample: run 10 to 20 real prompts from the project through
   the current provider and read `usage` off the responses
4. Ask the user for their monthly spend and the rough input/output split

**Set `reasoning_effort` before you measure.** Liner reasons by default
(`medium`), and reasoning tokens are billed as output tokens. If the project is
moving from a model that does not reason, measuring against the default
overstates what the migration costs. Set `reasoning_effort` to `none` for a
like-for-like comparison, then measure again at a higher setting only if the
project actually wants reasoning.

Call sites that send images are the exception. `none` is rejected there, so
measure them at `low` and count their reasoning tokens as output. The image
itself is billed as input tokens at the standard rate, and even a small image
measured at about 1,100 of them, so an image-heavy workload needs its image
count in the estimate.

Reasoning tokens also land in `usage.total_tokens`. An application that enforces
its own ceiling from that field will start refusing requests it used to accept,
and the user sees a token-limit error rather than anything pointing at the
migration.

Look up the current provider's published price for the exact model in the code.
Do not rely on remembered prices, since they change. Then compute both sides
from the same token counts.

**Liner Model API pricing, per 1M tokens:** input $1.00, output $6.00, cached
input $0.10.

Prompt caching is live and billed at the cached rate, so a project with a large
fixed system prompt will see a bigger difference than the headline rates
suggest. Include the cached portion in the estimate when the project reuses a
long prefix.

Present it like this, then stop and wait:

```
Workload: ~42M input / ~8M output per month, measured over 200 real calls
Liner run with reasoning_effort=none, matching the current model

  Current (<provider model>)  $<in> + $<out>   = $<total> / month
  Liner (liner-mark-1.0)      $42.00 + $48.00  = $90.00 / month
                              input $1.00, output $6.00 per 1M tokens

  Difference: <+ or - $X per month>
```

If Liner comes out more expensive, give that number and say it plainly in the
same breath. The user sees it on their next invoice either way, and a skill that
buried the answer does not get used twice. Where Liner comes out cheaper, say
that with the same plainness.

Ask explicitly before proceeding: **"Want me to apply the change?"** Never edit
code in the same turn as presenting the estimate.

## Step 4 — Apply the change

Three values change, plus `reasoning_effort` when step 3 showed the project is
coming from a model that does not reason. Nothing else should.

| Setting | Value |
| --- | --- |
| Base URL | `https://platform.liner.com/api/v1` |
| API key | The user's Liner key, read from `LINER_API_KEY` |
| Model | `liner-mark-1.0` |
| `reasoning_effort` | `none`, only when the source model did not reason, and never on a call site that sends images (use `low` there) |

Authentication is `Authorization: Bearer <key>`, which is what the OpenAI SDKs
already send. The user gets a key at platform.liner.com; if they do not have one
yet, stop and tell them, rather than leaving a placeholder in the code.

```python
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["LINER_API_KEY"],
    base_url="https://platform.liner.com/api/v1",
)

response = client.chat.completions.create(
    model="liner-mark-1.0",
    messages=[{"role": "user", "content": "Hello"}],
)
```

A Responses API call site takes the same three values and keeps
`client.responses.create` as it is:

```python
response = client.responses.create(
    model="liner-mark-1.0",
    input="Hello",
)
```

Keep the old configuration reachable. An environment variable switch or a
one-line constant is enough. The user needs to be able to go back in seconds if
something looks wrong in production, and a migration they cannot reverse is one
they will not deploy.

Do not reformat surrounding code, rename variables, or "improve" the
integration while you are in there. The diff should be small enough that the
user can read it in one screen.

## Step 5 — Verify with a real call

An untested migration is not finished. Run something real from the project, not
a hello-world:

- One non-streaming call, and confirm `usage` comes back populated
- Read `x-liner-ignored-parameters` on that response. Anything named there was
  dropped, and it is the fastest check that the migrated request carries only
  fields that actually take effect
- One streaming call if the project streams, and confirm it ends with `[DONE]` on
  Chat Completions or `response.completed` on the Responses API
- One call carrying an image if the project sends images, and confirm the answer
  actually reflects the image
- One function call round trip if the project uses tools, and confirm
  `tool_calls` arrives with parseable `arguments`
- The project's own test suite, if it has one

Then report what you changed, what you did not change and why, and the estimated
cost difference. If anything in step 2 was left unmigrated, repeat that list at
the end so it does not get lost.

## Limits worth knowing

Context window 1,000,000 tokens. Maximum output 65,536 tokens. Rate limit 10
QPS; `429` responses carry `Retry-After`, so honor it rather than retrying
immediately.

## If something contradicts this file

This skill was written against the API as measured on 2026-09-11. If a real call
behaves differently from what is written here, trust the real call, tell the
user what differed, and keep going. Do not bend the user's code to match a
document.

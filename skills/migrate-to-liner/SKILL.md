---
name: migrate-to-liner
description: Migrate an existing OpenAI-compatible LLM integration to the Liner Model API (liner-mark-1.0), showing a cost comparison before touching any code and auditing every call site for parameters Liner does not support. Use this whenever someone wants to switch, port, try, or evaluate Liner as their LLM provider, mentions liner-mark or platform.liner.com, asks what Liner would cost compared to their current OpenAI/Anthropic/Gemini bill, or asks whether Liner is a drop-in replacement for code they already have. Also use it when a repository already calls an OpenAI-compatible chat completions endpoint and the user asks about benchmarking or pricing another provider, even if they never say the word "migrate".
license: MIT
---

# Migrate to the Liner Model API

Liner Model API speaks the OpenAI Chat Completions format. For most projects the
migration is three values: base URL, API key, model name. The work that actually
matters is everything around those three values, because "OpenAI-compatible" is
never 100% compatible, and the gaps that hurt are the ones that return `200 OK`
and quietly do the wrong thing.

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

- SDK clients: `OpenAI(`, `AsyncOpenAI(`, `new OpenAI(`, `openai.ChatCompletion`
- Raw HTTP: `chat/completions`, `api.openai.com`, `generativelanguage.googleapis.com`, `api.anthropic.com`
- Config: `OPENAI_API_KEY`, `base_url`, `baseURL`, `OPENAI_BASE_URL`, model names in env files, YAML, or constants
- Frameworks that wrap the client: LangChain, LlamaIndex, Vercel AI SDK, LiteLLM, Instructor

Build a list of call sites. For each one record the model name, every request
parameter passed, and how the response is read. **How the response is read
matters as much as the request** — code that does `response.choices[1]` or
relies on a JSON schema will break in ways the request parameters alone do not
reveal.

Report the list before moving on. If the project has more than one provider or
more than one model tier, ask which ones are in scope rather than assuming all
of them.

## Step 2 — Compatibility audit

This is the part that earns the user's trust. Liner does not handle unsupported
parameters uniformly. Some are rejected, some work, and some return `200 OK` and
are ignored. The third group is dangerous because nothing in the response says
anything went wrong.

Verified against production on 2026-09-07.

| In the current code | What Liner does | Your action |
| --- | --- | --- |
| `seed` | Not supported | **Blocker.** Remove it rather than relying on an error to surface it, and tell the user that identical requests are not guaranteed to return identical output. Tests and golden-file comparisons pinned to a seed have to change before this call site moves. |
| `n` greater than 1 | Rejected | **Blocker.** Only one completion per request. Either drop the dependency on multiple choices, or leave this call site on its current provider. |
| `response_format` (JSON mode / structured output) | Rejected | **Blocker.** If the project needs structured output, function calling with `tools` is supported and is the path to suggest. |
| `messages[].content` as an array (image or audio parts) | Rejected | **Blocker.** Only string content is supported. |
| `max_tokens` | Returns `200` and the value is honored | Safe. Prefer rewriting to `max_completion_tokens`, which is the documented field. |
| Any field not in the supported list below | Undefined | Treat as a blocker. Test it against a real key before trusting it. |

For each blocker, show the user the file and line, what breaks, and what the fix
would be. Then leave the call site alone. Partial migration is a good result:
move the call sites that are clean, list the ones that are not, and let the user
decide.

**Supported request fields:** `model`, `messages` (string content only),
`stream`, `stream_options.include_usage`, `max_completion_tokens`,
`reasoning_effort` (`none`, `low`, `medium` default, `high`, `max`), `tools`,
`tool_choice`, `parallel_tool_calls`, `temperature`, `top_p`,
`presence_penalty`, `frequency_penalty`.

### Breakage that is not a request parameter

Two things reliably break a migration and neither of them appears when you read
request payloads, so look for both explicitly:

- **Tokenizer lookups keyed on the model name.** `tiktoken.encoding_for_model("liner-mark-1.0")`
  raises `KeyError`, usually at import time, which kills the process before it
  serves anything. Decouple the encoding from the API model name rather than
  deleting the token accounting that depends on it.
- **Model names hardcoded away from the config constant.** A project that
  defines a `MODEL` constant often still has a literal `model="gpt-4o"` at a
  second call site, most often the follow-up request after a tool result. Grep
  for the literal string, not only for the constant.

Streaming (SSE), function calling including parallel and streamed tool call
deltas, and prompt caching all work as documented. The server is stateless, so
conversation history must be resent on every request, same as OpenAI.

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

Three values change. Nothing else should.

| Setting | Value |
| --- | --- |
| Base URL | `https://platform.liner.com/api/v1` |
| API key | The user's Liner key, read from `LINER_API_KEY` |
| Model | `liner-mark-1.0` |

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
- One streaming call if the project streams, and confirm the `[DONE]` terminator
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

This skill was written against the API as measured on 2026-08-27. If a real call
behaves differently from what is written here, trust the real call, tell the
user what differed, and keep going. Do not bend the user's code to match a
document.

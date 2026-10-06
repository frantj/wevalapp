# How to add a custom SUT (System Under Test)

How to point a Weval evaluation at a model or agent that is **not** reachable through a
built-in provider (`openrouter:`, `openai:`, `anthropic:`, …). Typical cases: a local
model (Ollama, vLLM, LM Studio), a private OpenAI-compatible gateway, or an agent such as
the Azure Foundry WPS agents. Last updated: 2026-10-06.

**Sources of truth.** This guide describes behaviour; the code is authoritative:

| Topic | Source |
|---|---|
| Custom model fields | `src/lib/llm-clients/types.ts` (`CustomModelDefinition`) |
| Request/response handling, retries, mismatch guard | `src/lib/llm-clients/generic-client.ts` |
| Registration and dispatch | `src/lib/llm-clients/client-dispatcher.ts`, `runBlueprint()` in `src/cli/commands/run-config.ts` |
| Full field reference with more examples | `docs/BLUEPRINT_FORMAT.md` → *Custom Model Definitions* |
| Worked example (agent behind a proxy) | `docs/jesper's docs/AZURE-AGENT-PROXY-README.md` |

---

## 1. Decide which path you need

Weval talks to a custom SUT via one HTTP `POST` per prompt, using one of the wire formats it
already knows (`inherit`). So the first question is: **does your SUT already speak one of
those formats?**

| Your SUT exposes… | Path |
|---|---|
| OpenAI `chat/completions` (Ollama, vLLM, LM Studio, LiteLLM, most gateways) | **A** — blueprint entry only |
| Anthropic Messages or Gemini `generateContent` | **A** — blueprint entry, `inherit: anthropic` / `google` |
| OpenAI legacy `completions` | **A** — with `format: completions` |
| Anything else (agent APIs, threads/runs, Responses API, multi-step workflows, custom JSON) | **B** — write a small proxy, then do A |

If the SUT is already on OpenRouter, don't add a custom SUT at all — use
`openrouter:<vendor>/<model>`. Per the run checklist, needing a custom path turns a scoring
run into an infrastructure task, so confirm reachability before setting a run date.

## 2. Path A — add a custom model entry to the blueprint

Add an **object** (not a string) to the blueprint's `models:` list:

```yaml
models:
  - openrouter:openai/gpt-4o-mini          # built-in models can sit alongside
  - id: 'local:llama3-8b'                  # unique; this is the ID that appears in results
    url: 'http://localhost:11434/v1/chat/completions'   # full endpoint URL, not a base URL
    modelName: 'llama3:instruct'           # sent as "model" in the request body
    inherit: 'openai'                      # wire format: openai | anthropic | google | mistral | together | xai | openrouter
    headers:
      Authorization: 'Bearer ${MY_SUT_API_KEY}'   # ${VAR} is expanded from .env at request time
    parameters:
      max_tokens: 4000                     # override the 1500 default if answers are long
```

Required fields: `id`, `url`, `modelName`, `inherit`. Optional: `format`, `promptFormat`,
`headers`, `parameters`, `parameterMapping` — see `docs/BLUEPRINT_FORMAT.md`.

Rules worth knowing:

- **`id` must be unique and stable.** It is the key under which results, leaderboards and
  summaries are stored. Renaming it later creates a "new" system. A `prefix:name` shape
  (`azure:wps-research-agent`, `local:llama3-8b`) is conventional, not required.
- **`url` is the full endpoint** (`…/v1/chat/completions`), not a base URL. Weval does not
  append a path.
- **Never put a literal secret in `headers`.** Blueprints are archived and published. Use
  `${VAR}` and put the value in the repo-root `.env` (gitignored). `pnpm cli` loads `.env`
  automatically. An unset variable is sent unexpanded and a warning is logged, so the auth
  failure tells you which variable is missing. Header values whose names look like
  credentials (`Authorization`, `api-key`, `x-api-key`) are redacted in logs.
- **`parameters` wins over everything.** Use it to override defaults or add extras; set a
  key to `null` to remove it from the request. Example: a reasoning deployment that rejects
  `temperature` needs `parameters: { temperature: null }`. (See the run checklist on pinning
  `temp:0`. Only drop the field if the endpoint actually rejects it, and record that in the
  run notes.)
- **Defaults sent by the generic client:** `max_tokens: 1500`, `temperature` from the
  blueprint (Weval's default is `0`), `stream: false`.

## 3. Path B — wrap a non-compatible SUT in a proxy

If the SUT is an agent or uses its own API, put a small HTTP server in front of it that
translates between the two:

- `POST /v1/chat/completions`: accepts an OpenAI-style body (`model`, `messages`, …), calls
  the SUT, and returns `{ choices: [{ message: { role: 'assistant', content } }], model }`.
- `GET /health`: optional, but useful for pre-flight checks.

Start from an existing proxy instead of a blank file:

| File | Pattern |
|---|---|
| `tools/azure-agent-proxy/foundry-server.ts` | Synchronous Responses API, AAD client-credentials, disk cache, concurrency limit, rate-limit retry |
| `tools/azure-agent-proxy/server.ts` | Classic threads/runs with polling |

What a proxy must get right:

1. **Echo the requested `model` back in the response** (or omit `model` entirely). The
   generic client rejects a response whose `model` is unrelated to `modelName` (see
   *Model mismatch* below). The existing proxies echo `parsed.model`.
2. **Return proper status codes.** Weval retries `429` (honouring `Retry-After`) and
   `502/503/504/529`. Any other 4xx/5xx is treated as **permanent** for that cell. Return
   `502`/`503` for upstream failures you want retried, and `400` only for errors that are
   genuinely the request's fault.
3. **Pass the whole conversation through.** Multi-turn prompts arrive as a full `messages`
   array, and a system prompt arrives as a leading `system` message. If the agent only
   takes one input string, decide explicitly how to fold history in, and document that
   decision.
4. **Mind the cache key.** The Azure proxies cache by a hash of `messages` only. If you
   change the agent behind the same proxy (new base model, new instructions), clear or
   re-namespace the cache, or you will score stale answers.
5. **Read secrets from `.env`**, not hardcoded values. Run with
   `npx tsx --env-file=.env tools/<your-proxy>.ts` (the proxy scripts don't load `.env`
   themselves).
6. **Never emit internal traces in scored responses** (`WPS_EMIT_TRACE=0` for the WPS
   agent). Leaked internals have previously driven judge agreement negative.

Then register it with a Path A entry:

```yaml
  - id: 'azure:wps-research-agent'
    url: 'http://localhost:3101/v1/chat/completions'
    modelName: 'wps-research-agent'
    inherit: 'openai'
```

The proxy must be **running for the entire eval run**.

## 4. Smoke-test before spending judge budget

**a. Endpoint directly** (this catches URL, auth and response-shape problems):

```bash
curl -s -X POST http://localhost:3101/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"wps-research-agent","messages":[{"role":"user","content":"<a realistic domain prompt>"}]}'
```

Check that the answer appears at `choices[0].message.content` (for `inherit: openai`) and
that `model` matches or is absent. Trivial prompts like "say hello" can mislead agentic SUTs,
so use a realistic one.

**b. Through Weval, without judging.** Make a throwaway copy of the blueprint with one or
two prompts and only the custom model, then run:

```bash
pnpm cli run-config local \
  --config path/to/smoke-blueprint.yml \
  --run-label smoke-custom-sut \
  --eval-method none \
  --skip-executive-summary \
  --gen-timeout-ms 180000
```

In the logs, look for `Registering N custom models` and
`Using cached custom client for model ID: <your id>`, then check the response text in the
result file. Don't add `--demo-stdout` here: it forces `llm-coverage` judging whatever
`--eval-method` says, so it spends judge budget. Delete the smoke result afterwards so it
doesn't sit next to real runs.

## 5. Run it for real

```bash
pnpm cli run-config local \
  --config path/to/blueprint.yml \
  --run-label <label> \
  --eval-method llm-coverage \
  --skip-executive-summary \
  --gen-timeout-ms 180000 \
  --gen-retries 2
```

- **Raise `--gen-timeout-ms` up front.** The default is 30 s, which agents and reasoning
  models routinely exceed. A timeout fails the cell (`Custom API request timed out after
  …ms`). The flag applies to every model in the run.
- The exact flag set used by previous scored runs is recorded in each run's `STATUS.md`.
  Prefer copying that over the example above. The pre-run checklist
  (`WPS-BENCHMARK-RUN-CHECKLIST.md`) still applies in full. In particular, never fold a new
  bare SUT into `base_average` / `BASE_MODELS`, and record provenance (endpoint, commit,
  auth mode) for agent arms.

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Custom model silently missing from results; warning `Invalid object entry in models array…` | Entry has no `id` (or the YAML isn't a mapping) | Add `id`; check indentation |
| `Unsupported LLM provider: "<prefix>"…` or `Invalid modelId format…` | The model ID was dispatched without being registered (see `repair-run` caveat below) | Run via `run-config`, which registers custom models |
| `Model mismatch (<id>): requested 'X' but the provider served 'Y'` | Endpoint returned an unrelated `model`, e.g. an unfunded key silently routed to another model | Fix the endpoint/key. Only if the mismatch is a known harmless alias, set `WEVAL_ALLOW_MODEL_MISMATCH=true` and record why |
| `Custom API request timed out after 30000ms` | Default generation timeout | `--gen-timeout-ms` |
| `Custom API Error (<id>): 401/403 …` | Bad or missing credential; `${VAR}` not set in `.env` | Check for the `Header references ${VAR}, which is not set` warning |
| `Custom API Error (<id>): 404 …` | `url` is a base URL, not the full endpoint | Use the full `…/chat/completions` path |
| Empty responses with no error | Response shape doesn't match `inherit` (e.g. agent returns `{ output: … }`) | Fix the proxy to return the inherited shape, or change `inherit` |
| Answers cut off mid-sentence | `max_tokens: 1500` default | `parameters: { max_tokens: <n> }` |
| `400 … temperature not supported` | Reasoning deployment rejects `temperature` | `parameters: { temperature: null }`, and note it in the run record |
| Many cells fail together on one burst | Concurrency ceiling at the SUT or judge provider, not model behaviour | Repair rather than re-run (see caveat below) |

## 7. Known limitation: `repair-run` and custom SUTs

`pnpm cli repair-run` does **not** call `registerCustomModels` (see
`src/cli/commands/repair-run.ts`). Consequences:

- **Re-judging** already-generated responses works for any model, including custom ones,
  because no SUT call is made.
- **Re-generating** a failed custom-SUT cell will fail in the dispatcher with
  `Unsupported LLM provider` / `Invalid modelId format`, because the custom ID is not
  registered in that process.

Until that is fixed, recover failed custom-SUT generations by re-running that SUT with
`run-config` (a single-model copy of the blueprint), not with `repair-run`, and disclose the
mixed-vintage result as the checklist requires.

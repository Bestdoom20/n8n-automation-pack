# n8n-automation-pack

Four n8n workflows running on my home server: a lead-intake webhook that scores inbound leads, a daily pipeline digest built from CRM deal counts, a small tool-using agent, and an error handler that the other three report to.

n8n 2.48.7 runs in Docker, bound to `127.0.0.1:5678` only. Nothing is exposed to the network. No workflow sends a message to anyone: no email, chat or SMS nodes.

## Status

Everything below was executed on 2026-11-20 against the live Gemini free tier, using invented leads and aggregate CRM counts only. The model is `gemini-3.5-flash-lite`. The Gemini key lives only in n8n's encrypted credential store.

| Workflow | Result | Execution ids (local instance) |
|---|---|---|
| `lead-intake` | Validation 400, model scoring (3 leads) and the invalid-JSON fallback all ran | 3, 7, 8, 9 |
| `daily-pipeline-digest` | HubSpot GETs, aggregation, model summary and markdown file ran | 11, 16 |
| `agent-with-tools` | Agent called both tools and answered | 14, 15 |
| `error-handler` | Logged a forced failure and a real tool-configuration failure | 6, 13 |

The fallback run (invalid JSON) was done on a throwaway copy of `lead-intake` with the model output capped at 20 tokens, so the reply was cut off mid-object. The copy was deleted afterwards, so its execution record is gone; the log line it wrote is shown below.

## The workflows

### lead-intake

- Trigger: `POST /webhook/lead-intake` with JSON `name`, `title`, `company`, `message`.
- Steps: a Code node checks that all four are non-empty strings and answers HTTP 400 with the missing fields if not. A valid lead goes to a Gemini chain that returns JSON with `fit_tier` (high, medium, low), `reason` and `suggested_next_step`. A second Code node parses and checks that JSON. If the model returned something invalid, a fallback branch marks the lead `unknown` and flags it for manual review instead of failing. Either way the result is appended to `files/lead-log.jsonl`.
- Result: the JSON result, including `model_output_valid`.

### daily-pipeline-digest

- Trigger: schedule at 07:00 Vancouver time, plus a manual trigger.
- Steps: two read-only HubSpot GET requests (the deal pipeline definitions, then all deals, following pagination). A Code node counts deals per stage. Gemini writes a five-line summary from the counts. A Code node builds a markdown file.
- Result: `files/digests/pipeline-digest-YYYY-MM-DD.md`. Only the deal property `dealstage` is requested, so the workflow never holds contact or company names.

### agent-with-tools

- Trigger: `POST /webhook/agent` with JSON `{"question": "..."}`.
- Steps: the n8n AI Agent node with the Gemini chat model and two tools: an HTTP Request tool for the open-meteo weather API (no key needed) and a calculator.
- Result: the answer, the list of tools called, and each tool call with its observation.

### error-handler

- Trigger: Error Trigger. It is set as the error workflow of the other three.
- Steps: formats workflow name, failing node and message, and appends one line to `files/errors.jsonl`.

## Run it

```bash
docker compose up -d
curl -s localhost:5678/healthz
```

Open the editor through an SSH tunnel (`ssh -L 5678:127.0.0.1:5678 user@server`), create the owner account, then import:

```bash
docker cp workflows n8n:/tmp/wf
docker exec n8n sh -c 'for f in /tmp/wf/*.json; do n8n import:workflow --input=$f; done'
```

Create two credentials in the UI with these exact names, because the workflows reference them:

- `Gemini (free)`, type Google Gemini (PaLM) API, with your own key.
- `HubSpot (read-only)`, type Header Auth, name `Authorization`, value `Bearer <token>`. Use a token that can only read deals.

Then publish the four workflows and restart the container once so the webhooks register. Create `files/` and `files/digests/` owned by uid 1000 before the first run.

```bash
curl -s -X POST localhost:5678/webhook/lead-intake -H 'Content-Type: application/json' \
  -d '{"name":"Test Person","title":"Ops Manager","company":"Example Co","message":"We want to automate invoice intake."}'
curl -s -X POST localhost:5678/webhook/agent -H 'Content-Type: application/json' \
  -d '{"question":"What is the temperature in Kelowna right now, and what is that times 1.8 plus 32?"}'
```

## Evidence from the run on 2026-11-20

Validation, with an invented lead missing three fields:

```
$ curl -s -w ' [%{http_code}]' -X POST localhost:5678/webhook/lead-intake -d '{"name":"Test Person","title":""}'
{"error":"invalid input","missing":["title","company","message"]} [400]
```

Lead scoring (invented lead, execution 7):

```
{"fit_tier":"high","reason":"The lead has a clear, manual, high-volume document processing workflow that is an ideal fit for automation.","suggested_next_step":"Schedule a 15-minute discovery call to discuss invoice processing workflows and potential ROI.","model_output_valid":true}
```

Two more invented leads carried prompt-injection text in the message field (asking the model to drop the JSON format or to use a made-up tier). The model ignored it both times and returned valid JSON with tier `low`.

Invalid JSON fallback (truncated model output):

```
{"fit_tier":"unknown","reason":"Model did not return valid JSON; flagged for manual review.","suggested_next_step":"Review this lead by hand.","model_output_valid":false,"raw_model_output":"{\"fit_tier\": \"high\", \"reason\": \"The lead has a"}
```

Pipeline digest, from the real CRM, counts only (100 deals, five stages in use), execution 16:

```
| Studio cycle / Researching | 6 |
| Studio cycle / Connecting | 75 |
| Studio cycle / Champion in | 9 |
| Studio cycle / Warming (replied) | 7 |
| Studio cycle / Decision maker reached | 3 |

Three-quarters of all active deals currently sit in the connecting stage.
This single phase represents 75 percent of the entire pipeline volume.
Early research and later decision-maker stages both remain notably thin.
Momentum is clearly concentrated in early outreach rather than closing.
Focus next on moving those connected prospects deeper toward decisions.
```

Agent, execution 15. `tools_called` is `["get_weather","Calculator"]`. The weather call returned open-meteo data for Kelowna and the calculator computed `3.6 * 1.8 + 32 = 38.48`, which the answer repeats.

Error handler. The first line is from a throwaway workflow with a Code node that throws (deleted afterwards). The second is a real failure from the agent, whose first tool node had an invalid name and was fixed.

```
{"ts":"2026-11-20T18:59:53.068Z","workflow":"force-failure-test","node":"Deliberate failure","message":"Deliberate failure to prove the error handler [line 1]","executionId":"5"}
{"ts":"2026-11-20T19:03:12.484Z","workflow":"agent-with-tools","node":"AI Agent","message":"Error in sub-node Weather (open-meteo)","executionId":"12"}
```

## Limits

- Malformed JSON bodies are rejected by n8n itself with HTTP 422 before the workflow starts. The 400 path covers well-formed JSON with missing or empty fields.
- The Gemini free tier is rate limited and shared with other things on the same key. The lead scorer is not built for volume, and there is no model fallback wired in. If the primary model is busy the run fails and the error handler logs it.
- The digest was also started from the command line (`n8n execute`), not only by its schedule. The 07:00 schedule has not fired yet.
- The digest depends on HubSpot deals having a stage. Deals without one are counted as `unknown stage`.
- File logs are plain append-only files with no rotation.
- The container runs plain http on localhost, with `N8N_SECURE_COOKIE=false`. Put it behind TLS before exposing it anywhere.
- Exports contain credential names and ids only. Key material lives in n8n's encrypted credential store.

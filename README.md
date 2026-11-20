# ple-ci

Continuous integration for the `physical-engineerings-last-exam` entry
([`TejashPrakash/ple-entry`](https://github.com/TejashPrakash/ple-entry)).

The competition kit (`google-research/ple`) needs Docker and 16 GB of RAM, so the checks run here
rather than on the laptop. This repo holds **only** the workflow: the entry stays clean, and no
secret ever lands inside it.

Pinned in the workflow:

| | |
|---|---|
| kit | `google-research/ple` @ `8320f0402fdca9f42476607f72bcfd5271cd50c9` |
| entry | `TejashPrakash/ple-entry` (branch chosen per run) |
| runner | `ubuntu-latest`, 4 vCPU / 16 GB |

## Jobs

| job | when | what it does | uses quota |
|---|---|---|---|
| `static` | every push, every dispatch | `kit validate` + `kit artifacts` on the entry | no |
| `harness` | dispatch, `smoke=true` | `kit check --rebuild`: Docker build, serve, listen, then the kit's own model smoke task | yes |
| `grade-artifacts` | dispatch, `grade=true` | `eval/run.py --artifacts` on the two dev tasks the template shipped with zero files | yes |
| `model-run` | dispatch, `tasks` non-empty | runs the selected dev tasks against the selected model | yes |

## Why the model jobs are dispatch-only

The CI model is a free tier with a **20 requests/day** cap per model. A push that ran `kit check`
burned that cap, and every later run — including ones that only wanted to grade files — came back
red with `429 RESOURCE_EXHAUSTED`, a failure that says nothing about the entry. Pushes now run
`static` only, and it is green in ~15 s. Model work is started deliberately:

```bash
gh workflow run ple-kit.yml -R TejashPrakash/ple-ci \
  -f entry_ref=local-model -f tasks=foundation-strategy \
  -f model_name=gemini-3-flash-preview
```

## Configuration

Repo **variables**: `PLE_MODEL_BASE_URL`, `PLE_MODEL_NAME`.
Repo **secrets**: `PLE_MODEL_API_KEY` (primary provider), `PLE_MODEL_API_KEY_ALT` (second provider
for cross-model runs), `HF_TOKEN` (gated task definitions).

`model-run` rewrites `model_api` / `model_base_url` in the entry's `submission.yaml` inside the
runner, writes a `ci.env` outside the entry, and uploads every textual run record
(`results.json`, `harbor.log`, `serve.log`, `<entry>.txt`, `exception.txt`, `trajectory.json`) as
the `ple-run-<id>` artifact.

## Known failure signatures

- `status: ok`, `tokens: null`, 0 files, ~4 s — the harness accepted the websocket and made **no
  model request**. A provider-shape mismatch, not a score.
- `harness: NonZeroAgentExitCodeError` with `serve: the harness gave up: RateLimitError: 429` — the
  free tier's daily cap. The run is unmeasured, not failed; check
  `attempt*/jobs/dev/<task>__*/agent/local-model.txt` for the turns that did run.

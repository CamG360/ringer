# Ringer — orientation guide
#ver 1736.080726

Where you are, what you have, and how to use it.

---

## What is Ringer?

A single Python script (`ringer.py`) that fans work out to cheap parallel CLI workers and **verifies every result by executing a check command**. Pass/fail comes from running actual code, not reading summaries.

You write the spec and the check. Ringer dispatches, retries once on failure with the check output injected, and logs everything.

You are the orchestrator. The workers do the typing.

---

## The two worker lanes

| Lane | Worker | Model | Cost | Auth |
|------|--------|-------|------|------|
| `codex` | Codex CLI v0.143.0 | gpt-5.5 | Covered by ChatGPT plan | `codex login` → ChatGPT OAuth |
| `opencode` | OpenCode v1.17.15 | openrouter/z-ai/glm-5.2 (default) | ~$0.004/task | OpenRouter key in `~/.local/share/opencode/auth.json` |

**Why does Codex appear free?**
Codex CLI authenticates against your ChatGPT subscription (Plus or Pro). `codex login status` shows "Logged in using ChatGPT" — no API key, no per-token invoice. It draws from the plan's usage cap, not a separate billing account.

**OpenCode** is the terminal interface. **OpenRouter** is the model broker behind it — one key, access to every model they serve. The `"model"` field in a manifest task selects the model; `"engine": "opencode"` selects the harness. Any OpenRouter slug works: Mistral, Qwen, DeepSeek, Kimi, etc.

---

## Where everything lives

| What | Path |
|------|------|
| Ringer script | `C:\Users\CampbellMcCord\repos\ringer\ringer.py` |
| Config | `C:\Users\CampbellMcCord\.config\ringer\config.toml` |
| State + run JSON | `C:\Users\CampbellMcCord\.ringer\runs\` |
| Artifacts (HTML) | `C:\Users\CampbellMcCord\.ringer\artifacts\` |
| Model log | `C:\Users\CampbellMcCord\.ringer\runs.jsonl` |
| Codex binary | `C:\Users\CampbellMcCord\AppData\Roaming\fnm\node-versions\v22.14.0\installation\node.exe` + `codex.js` |
| OpenCode binary | `C:\Users\CampbellMcCord\.opencode\bin\opencode.EXE` |
| OpenRouter auth | `C:\Users\CampbellMcCord\.local\share\opencode\auth.json` |
| Ringer skill | `C:\Users\CampbellMcCord\.claude\skills\ringer\SKILL.md` |
| GitHub fork | `https://github.com/CamG360/ringer` |
| Windows setup notes | `docs/windows-setup.md` (this repo) |

---

## Key commands

```bash
cd /c/Users/CampbellMcCord/repos/ringer

python ringer.py demo                        # 3-task smoke test (proves the stack works)
python ringer.py lint manifest.json          # check manifest before running — always do this
python ringer.py run manifest.json           # run a swarm (opens Ringside in browser)
python ringer.py run manifest.json --dry-run # print the plan, spawn nothing
python ringer.py models                      # scoreboard: per-model pass rates + cost
python ringer.py models --task-type research # filtered scoreboard
python ringer.py catalog --changes           # OpenRouter: what's newly free or cheap
python ringer.py hud                         # open Ringside at http://127.0.0.1:8700
```

---

## The scoreboard (`./ringer.py models`)

Every task appends a row to `runs.jsonl`: engine, model, task_type, verdict, tokens, duration. The scoreboard aggregates by `(model, task_type)` and shows first-try pass rate — the routing signal.

Current state (as of 2026-07-08/09):

| task_type | model | pass rate | first-try |
|-----------|-------|-----------|-----------|
| research | GLM-5.2 | 1.00 (4/4) | 1.00 |
| code-feature | GLM-5.2 | 0.50 (1/2) | 0.50 |
| (untyped) | gpt-5.5 Codex | 0.20 (3/15) | 0.20 |

Note: the gpt-5.5 failures are from setup runs before the Windows patches, not real model failures. Research on GLM-5.2 is genuine signal.

**Before any real swarm:** run `./ringer.py models --task-type <type>` and `./ringer.py catalog --changes` to check if anything has gone free or cheaper.

---

## What swarm-shaped work looks like

Work is swarm-shaped when:
- 2–4 tasks are **independent** — task B does not need task A's output
- Each task can succeed or fail on its own
- Same job, different slices (different files, topics, angles, personas)

Work is NOT swarm-shaped when:
- Task B depends on task A's output (that's a pipeline — run sequentially)
- It's a single one-off edit (one file, a few lines, once — do it inline)

---

## Manifest structure (minimum viable)

```json
{
  "run_name": "job-name",
  "workdir": "~/.ringer/workdirs/job-name",
  "tasks": [
    {
      "key": "task-a",
      "engine": "opencode",
      "model": "openrouter/z-ai/glm-5.2",
      "task_type": "research",
      "spec": "Self-contained brief. Name every file to produce. Include exact paths.",
      "check": "test -f output.md && grep -q 'expected content' output.md || { echo 'FAIL: reason'; exit 1; }",
      "verified": "One sentence: what the check proves.",
      "expect_files": ["output.md"]
    }
  ]
}
```

Rules:
- `spec` must be self-contained — workers are stateless and cannot ask questions
- `check` must print WHY it fails — the retry prompt depends on it
- `check` must verify content, not just existence
- Always `lint` before `run`
- Always give each task a `task_type` — untyped tasks teach the scoreboard nothing

---

## Windows-specific notes

Three patches were needed to make Ringer work on Windows. See `docs/windows-setup.md` for full details.

Summary:
1. **`ringer.py`** — `_run_check` uses `bash -c` on win32 (committed to `CamG360/ringer`). If pulling upstream, reapply.
2. **Config: codex engine** — points to `node.exe` + `codex.js` directly; `.CMD` wrappers can't be exec'd.
3. **Config: codex sandbox** — `--sandbox danger-full-access`; `workspace-write` locks file ACLs so checks can't read output.

---

## Asset register

| Asset | ID |
|-------|----|
| Ringer tool | `5105bfa9-3dd5-4992-8a74-ebd5165e0082` |
| Ringer skill | `d03e6fdb-1457-4dd9-8ce4-1b09374e8aee` |

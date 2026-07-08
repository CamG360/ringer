# Ringer on Windows — setup notes

Verified on Windows 11 Pro, Git Bash (MSYS2), Python 3.13.1, Node 22.14.0 (fnm), codex-cli 0.143.0, opencode 1.17.15.

---

## ringer.py patch — bash for check commands

`asyncio.create_subprocess_shell` on Windows uses `cmd.exe`, which cannot parse POSIX check syntax (`test`, `$(...)`, `||`). The fix is in `_run_check` (committed to this repo):

```python
if sys.platform == "win32":
    bash = shutil.which("bash") or "bash"
    proc = await asyncio.create_subprocess_exec(
        bash, "-c", command, ...
    )
```

Requires Git Bash (or any bash) on PATH. `shutil.which("bash")` resolves to the Git Bash binary at runtime.

---

## config.toml — codex engine

The `codex` npm package installs as a `.CMD` wrapper on Windows. `asyncio.create_subprocess_exec` cannot execute `.CMD` files directly — they require `cmd.exe`. Point the engine at `node.exe` and the `codex.js` entry point instead:

```toml
[engines.codex]
bin = "C:/Users/<you>/AppData/Roaming/fnm/node-versions/<version>/installation/node.exe"
args_template = [
  "C:/Users/<you>/AppData/Roaming/fnm/node-versions/<version>/installation/node_modules/@openai/codex/bin/codex.js",
  "exec",
  "--skip-git-repo-check",
  "{access_args}",
  "{engine_args}",
  "-C",
  "{taskdir}",
  "{spec}",
]
sandbox_args = ["--sandbox", "danger-full-access"]
full_access_args = ["--dangerously-bypass-approvals-and-sandbox"]
token_regex = "tokens\\s+used\\s*:?\\s*([0-9][0-9,]*)"
```

Replace `<you>` and `<version>` with your fnm node version. Find the stable path with:

```bash
ls /c/Users/<you>/AppData/Roaming/fnm/node-versions/
```

**Why `danger-full-access` instead of `workspace-write`?**
The `workspace-write` sandbox profile on Windows locks created files with restricted ACLs. After the worker process exits, the check script cannot read those files (permission denied). `danger-full-access` skips ACL restrictions so checks can read worker output.

---

## config.toml — opencode engine

`opencode` installs as a native Windows `.EXE` via `curl -fsSL https://opencode.ai/install | bash` (lands in `~/.opencode/bin/`). It can be exec'd directly — no wrapper needed:

```toml
[engines.opencode]
bin = "C:/Users/<you>/.opencode/bin/opencode.EXE"
model_default = "openrouter/z-ai/glm-5.2"
args_template = [
  "run",
  "-m",
  "{model}",
  "--auto",
  "--format",
  "json",
  "{engine_args}",
  "--dir",
  "{taskdir}",
  "{spec}",
]
sandbox_args = []
full_access_args = []
token_regex = '"tokens":\{"total":([0-9]+)'
```

Auth: OpenRouter key goes in `~/.local/share/opencode/auth.json`:

```json
{"openrouter":{"type":"api","key":"sk-or-v1-..."}}
```

The `opencode-sandboxed.sh` wrapper in `engines/` is macOS-only (uses `sandbox-exec`). On Windows, invoke the `.EXE` directly as above.

---

## Notes

- `codex login status` → `codex login status` (subcommand, not `codex auth status`)
- After upgrading codex via `npm install -g @openai/codex`, the new `codex.js` lands under the fnm node-versions path, not the npm global path — update `args_template` accordingly
- `./ringer.py hud` opens Ringside at `http://127.0.0.1:8700`; the browser opens via `webbrowser.open()` which works on Windows

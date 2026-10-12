# NetOps Assistant: set up on a new Windows laptop

This is the whole Prodapt Synapt capstone project (Project 4) in one zip. It holds the source, the git history, the 100,000-case dataset, the trained models, the cached embedding vectors, the embedding model files and the Qdrant binary. Nothing big has to be downloaded from Hugging Face or GitHub, and the first setup takes minutes instead of 15 to 25.

**Not in the zip, on purpose:**

| Left out | Why | How it comes back |
|---|---|---|
| `.env` | holds the secrets (the Anthropic key, the Synapt login) | you create it in step 3 |
| `qdrant_storage/` (2.9 GB) | the search index | `bootstrap.py` rebuilds it from the cached vectors in step 5 |
| `backend/.venv`, `ui/.venv`, `web/node_modules`, `web/.next` | machine-specific installs and builds | `uv sync`, `npm ci` and `npm run build` |

If you are Claude Code reading this on the new laptop, follow the steps in order, run each check, and stop and tell the user if something is blocked (the "When something is blocked" section has the usual fixes). Never print or commit the values in `.env`. `CLAUDE.md` has the project context.

---

## 0. What the laptop needs

| Tool | Version | Check | Install if missing |
|---|---|---|---|
| Windows 10 or 11, x64 | | | |
| uv | any recent | `uv --version` | `winget install astral-sh.uv`, or `pip install uv`, or `powershell -ExecutionPolicy Bypass -c "irm https://astral.sh/uv/install.ps1 \| iex"` |
| Python | 3.12 | `uv python find 3.12` | uv downloads it on first `uv sync`. If that's blocked, install Python 3.12 from python.org or your company's software centre |
| Node.js | 22 | `node --version` | `winget install OpenJS.NodeJS --version 22.14.0`, or nodejs.org, or the software centre |
| PowerShell 7 (`pwsh`) | optional | `pwsh --version` | only `scripts/start-all.ps1` needs it; the manual steps below work in Windows PowerShell |

You also need about 8 GB of RAM and 6 GB of free disk. Ports 6333, 6334, 8000 and 3000 must be free.

## 1. Download and extract to a short path

1. Download `netops-assistant.zip` (from the GitHub repo's **Releases** page).
2. Unblock it before extracting, so Windows doesn't block the scripts inside:
   ```powershell
   Unblock-File $HOME\Downloads\netops-assistant.zip
   ```
3. Extract it so the project lands at **`C:\netops`**. The zip already contains a top folder named `netops`, so extract to `C:\`:
   ```powershell
   Expand-Archive $HOME\Downloads\netops-assistant.zip -DestinationPath C:\
   ```
   Don't use Explorer's default "Extract All" target (`Downloads\netops-assistant\netops\...`). Qdrant's storage nests about 175 characters below the project, and Windows paths stop at 260. If you can't write to `C:\`, use `$HOME\netops` and change `C:\netops` to it in every command below.

## 2. Point the embedding model at the bundled copy

The API embeds questions with two small models (bge-small and BM25). They're in `.tools\fastembed_cache`, but the library looks in `%TEMP%` unless told otherwise. Set this once for your user, then **open a new terminal**:

```powershell
[Environment]::SetEnvironmentVariable("FASTEMBED_CACHE_PATH", "C:\netops\.tools\fastembed_cache", "User")
```

Check it in the new terminal with `$env:FASTEMBED_CACHE_PATH`. Without it, the API tries to download the models from huggingface.co, which work networks often block.

## 3. Create `.env`

```powershell
cd C:\netops
Copy-Item .env.example .env
notepad .env
```

Fill in these values from the `.env` on your home machine. Bring them over privately (type them, or use a password manager). **Never commit them or put them in the public repo.**

| Variable | Value |
|---|---|
| `ANTHROPIC_API_KEY` | your Anthropic key. Leave it empty to run with no Claude: every agent has a deterministic fallback |
| `SYNAPT_USERNAME`, `SYNAPT_PASSWORD` | your Synapt account |
| `SYNAPT_PROVIDER_ID` | `netops_5g_assistant` |

Keep the rest of `.env.example` as it is. Two switches help on a locked-down network:
- `LLM_MODE=off` if api.anthropic.com is blocked. The app then answers from its deterministic agents and doesn't wait on Claude.
- `SYNAPT_ENABLED=false` if beta.synapt.ai is blocked. The diagnosis then uses the local runbooks only.

## 4. Start Qdrant (terminal 1)

`qdrant.exe` is already in `.tools\qdrant`, so this downloads nothing:

```powershell
cd C:\netops
powershell -ExecutionPolicy Bypass -File scripts\qdrant-native.ps1
```

Leave it running. It says `Qdrant v1.19.2 on http://localhost:6333`. If scripts are blocked entirely, run the same thing by hand:

```powershell
$env:QDRANT__STORAGE__STORAGE_PATH = "C:\netops\qdrant_storage\storage"
$env:QDRANT__STORAGE__SNAPSHOTS_PATH = "C:\netops\qdrant_storage\snapshots"
$env:QDRANT__TELEMETRY_DISABLED = "true"
C:\netops\.tools\qdrant\qdrant.exe
```

## 5. Install, index and start the API (terminal 2)

```powershell
cd C:\netops\backend
uv sync
uv run python scripts/bootstrap.py
uv run uvicorn netops.api.main:app --port 8000
```

- `uv sync` installs the Python packages from `uv.lock` into `backend\.venv` (a few minutes).
- `bootstrap.py` skips the dataset and the models (they're in the zip). It rebuilds the Qdrant index from `data\processed\dense_vectors.npy`, so it doesn't re-embed the 100,000 cases. That takes a few minutes, not 15 to 25. It ends with `bootstrap.ready cases=100000 runbooks=56`.
- Check http://localhost:8000/health. It should show `"qdrant": true`. `llm.state` shows whether Claude answers (`no_key` or `unavailable` is fine for a deterministic demo).

## 6. Install, build and start the web app (terminal 3)

```powershell
cd C:\netops\web
npm ci
npm run build
npm run start
```

Open http://localhost:3000; Diagnose is at http://localhost:3000/diagnose.

**Frontend-only fallback:** if the API can't run on this laptop, the web app can still demo from recorded responses. Add `?replay=1` to the address, or set `UI_REPLAY=1` in `.env` and restart `npm run start`. That needs only Node.

Once everything works once, `pwsh scripts/start-all.ps1` starts all three in their own windows (it needs PowerShell 7).

## 7. Check that everything works

```powershell
cd C:\netops\backend
uv run pytest            # 243 tests, no network or API cost
cd ..\web
npm test                 # 89 unit tests
```

Then ask a question on the Diagnose page, for example: "A network region has increasing fault occurrence and declining handover success. What corrective actions should be considered?"

---

## When something is blocked

| Symptom | Fix |
|---|---|
| `uv sync` can't reach pypi.org | Ask IT for the company PyPI mirror and set `$env:UV_DEFAULT_INDEX = "<mirror url>"`, then run `uv sync` again. If the network inspects TLS, also set `$env:UV_NATIVE_TLS = "true"` |
| uv can't download Python 3.12 | Install Python 3.12 yourself, then run `uv sync --python "C:\path\to\python.exe"` |
| `npm ci` can't reach the registry | `npm config set registry <company npm mirror>`. For TLS inspection: `npm config set cafile C:\path\to\company-ca.pem` |
| Certificate errors from Python (Anthropic, Synapt) | Set `$env:SSL_CERT_FILE` to the company CA bundle, and `SYNAPT_VERIFY_SSL=C:\path\to\company-ca.pem` in `.env` |
| `huggingface.co` errors in the API or bootstrap logs | Step 2 wasn't applied in this terminal. Open a new terminal and check `$env:FASTEMBED_CACHE_PATH` |
| `qdrant.exe` is blocked by antivirus or AppLocker | Ask IT to allow it (Qdrant v1.19.2, the official release). Meanwhile, use the `?replay=1` frontend-only demo |
| Bootstrap fails with `IO Error: The system cannot find the path specified` | The path is too long. Move the project to `C:\netops` and run `bootstrap.py` again |
| `running scripts is disabled on this system` | Use `powershell -ExecutionPolicy Bypass -File <script>`, or the manual commands in step 4 |
| The web app says "Claude is unavailable" | `/health` gives the reason in `llm.reason`. Answers still come from the deterministic agents |
| `401 unauthorized` from the API | Send the `X-API-Key` header with the value of `SERVICE_API_KEY` from `.env` |
| Garbled characters in the console | `$env:PYTHONUTF8 = "1"` |

`README.md` covers the project itself (the API, the evals and the configuration). `PLAN.md` ("resume here" at the top) and `PROGRESS.md` say where the work stands.
# netops-transfer

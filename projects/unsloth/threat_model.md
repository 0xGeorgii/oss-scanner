# Threat model

## What this project does and where untrusted input enters
- `unsloth/` is a Python library for fine-tuning and RL on LLMs. It is called by the user's own code with the user's own data; that code is trusted.
- `studio/` is Unsloth Studio: a FastAPI backend (`studio/backend`) and React frontend (`studio/frontend`) for training and chatting with models, also shipped as a Tauri Desktop app (`studio/src-tauri`). By default it listens on localhost; users can expose it on a network or through a Cloudflare tunnel, behind password and API-key auth.
- Untrusted input enters Studio through:
  - unauthenticated HTTP requests, whenever Studio is reachable over a network;
  - other Studio accounts on a multi-account install (managed accounts must not reach each other's data or the host outside their workspace);
  - web pages fetched by the browsing and research tools, and model output (prompt injection must not run code or exfiltrate data without the user's approval);
  - Hugging Face repos, model and dataset files the user downloads (Studio scans and gates remote code and pickles; bypassing those gates is in scope);
  - links and markdown rendered in chat.
- `install.sh`, `install.ps1`, `studio/setup.sh` and `studio/setup.ps1` download and install binaries; integrity checks on those downloads are in scope.

## How Studio is meant to be secured (so reports can tell a bypass from intended behaviour)
- Binding: `unsloth studio` listens on `127.0.0.1:8888` by default. `-H 0.0.0.0` is a deliberate opt-in. `--secure` serves only through a Cloudflare tunnel, keeps the bind on loopback, and fails closed when no tunnel comes up.
- Accounts: one owner (default user `unsloth`) plus optional managed accounts created by the owner (`auth/policy.py`, `routes/accounts.py`). Managed accounts are confined to their own workspace (`core/training/account_jobs.py` `account_path`). While any managed account exists, `full_access_permitted()` is false and unsandboxed tool modes are off install-wide, for the owner too.
- Credentials: password login gives a session JWT; `sk-unsloth-` API keys are for programmatic use and are refused where only the UI may act (for example defining a stdio MCP command, `require_ui_session_for_local_commands`). Keyless API access is an owner opt-in setting limited to approved callers (`utils/keyless_api_access.py`). Forwarded client IPs are trusted only with `UNSLOTH_STUDIO_TRUST_FORWARDED`.
- Code execution: when the owner enables tools, the model can run code in a sandbox; with full access permitted it can run on the host by design, after the user's approval. Anyone holding the Studio password or an owner API key is therefore trusted with host code execution. Reports should target paths that reach execution without those credentials or without the approval, or that escape the sandbox.
- Files from the Desktop shell reach the backend only as HMAC-signed native path leases (`utils/native_path_leases.py`); the webview cannot name an arbitrary host path.
- Downloaded models: remote code and pickle weights are scanned and need the user's consent (`utils/security/`); a repo the user explicitly trusted is out of scope.
- Desktop updates come only from `releases/latest/download/latest.json` and are signature-checked against the public key in `studio/src-tauri/tauri.conf.json`.

## Components that matter most / least
- Most: Studio authentication, API keys, multi-account isolation, path handling (uploads, native path leases, exports), the code-execution sandbox and tool approval, the remote-code and pickle gates, installers.
- Less: training kernels and model patching in `unsloth/` (they run the user's own code on the user's own data).
- Out of scope: `docker/` (a thin wrapper that packages the library and Studio), `tests/`, generated files, third-party code vendored under `studio/frontend` dependencies.

## How to exercise it
- `python -m pytest studio/backend/tests -q` (CPU; CUDA is spoofed where needed). Frontend: `cd studio/frontend && npm test`.
- Studio is installed at `/workspace/unsloth_studio` and starts offline with `unsloth studio -H 127.0.0.1 -p 8888` (user `unsloth`, password in `UNSLOTH_STUDIO_PASSWORD`, sandbox only). The web UI and `/api` are then on port 8888.

## How you rate severity
- Critical: unauthenticated remote code execution, or authentication bypass on a network-exposed Studio.
- High: a managed account reading or writing another account's data or host files outside its workspace; code execution from a downloaded model or dataset that bypasses Studio's scan or consent; installer integrity bypass; chat content leaked to a third party without user action.
- Medium: authenticated issues that need an unusual configuration, or stored XSS needing a crafted model or dataset.
- Low or not reportable: denial of service and resource exhaustion by an authenticated user against their own install, quadratic parsing of files the user chose, and issues that need the attacker to already control the user's machine or Python environment.

## Anything to leave alone
- The library executing user-supplied Python, `trust_remote_code=True` set by the user, or loading a pickle the user explicitly approved: these are intended.
- Ports being reachable when the user deliberately binds Studio or the Docker image to 0.0.0.0.

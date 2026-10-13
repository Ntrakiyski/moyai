# Deploy Moyai on Coolify

[Documentation](README.md) · [Cloud deployment](deployment.md)

Coolify hosts the **Moyai control plane** (UI, SQLite, tool broker). Agent sandboxes still run on **Modal** (default), Substrate, or AWS Lambda MicroVMs. The Docker image does not run coding agents itself.

```text
Browser → Coolify (Traefik + TLS) → Moyai container on your VPS
                                         ↑
                              Modal/Substrate sandboxes call back via HTTPS PUBLIC_URL
                                         ↓
                              your LiteLLM gateway
```

## Prerequisites

- Coolify on a VPS with enough disk for a persistent volume (start at ~10 GB)
- A DNS name for HTTPS (Coolify will terminate TLS)
- A reachable LiteLLM gateway + budgeted API key
- Modal tokens (or Substrate / Lambda credentials) for Cloud sessions

## Coolify resource setup

1. **New Resource** → Application → connect `https://github.com/Ntrakiyski/moyai` (or your fork).
2. Build pack: **Docker Compose**
3. Docker Compose Location: `/compose.coolify.yaml`
4. Branch: `main`
5. Keep **one instance** only (SQLite single-writer). Do not scale replicas.
6. Leave the Docker **Command / Entrypoint** empty so `docker_start.py` prepares `/data` and drops to UID `10001`.
7. Health check path: `/health` (compose also defines a container healthcheck).
8. Attach the named volume `moyai-data` at `/data` (compose already declares it).
9. Under **Environment Variables**, set at least `PUBLIC_URL` and `WORKSPACE_PASSWORD` (compose maps them into the container). Without both, remote requests return `Local preview only`.

### Alternative: Dockerfile only

Build pack **Dockerfile**, port **8787**, persistent volume mounted at `/data`, same environment variables as below. Prefer `compose.coolify.yaml` so volume + healthcheck stay in-repo.

## Environment variables

Copy [deploy/coolify.env.example](../deploy/coolify.env.example) into Coolify’s environment UI.

Required for a first Cloud-ready install:

| Variable | Notes |
| --- | --- |
| `PUBLIC_URL` | Exact HTTPS origin Coolify assigns, e.g. `https://moyai.example.com` |
| `DATA_DIR` | `/data` |
| `TRUSTED_PROXY_HOPS` | `1` behind Coolify’s Traefik |
| `WORKSPACE_PASSWORD` | ≥ 16 characters |
| `SESSION_SECRET` / `ENCRYPTION_KEY` | Generate once; back up with the volume. Never casually rotate `ENCRYPTION_KEY`. |
| `LITELLM_API_BASE` / `LITELLM_API_KEY` / `AGENT_MODEL` | Gateway URL ending in `/v1`, budgeted key, exact model alias |
| `MODAL_TOKEN_ID` / `MODAL_TOKEN_SECRET` | From `~/.modal.toml` after `modal token new` |
| `TEMPORAL_ENABLED` | `false` for first bring-up without Temporal Cloud |
| `AGENT_HARNESS` | `hermes` when Temporal is off (`deepagents-temporal` requires Temporal) |

After TLS is live, set `PUBLIC_URL` to that exact origin and redeploy if you changed it.

## Proxy notes (Coolify / Traefik)

- Preserve `Host` and forwarded headers (`TRUSTED_PROXY_HOPS=1`).
- Allow sandbox traffic to `/broker/` with run-scoped bearer tokens (do not put that path behind a browser-only login gate).
- Disable response buffering for event streams / SSE when configurable.
- Allow request bodies around **5 MB**.

Loopback `PUBLIC_URL` values deliberately disable Cloud execution.

## After deploy

1. Open `PUBLIC_URL` and sign in with `WORKSPACE_PASSWORD`.
2. **Settings → Runtime** → confirm configuration / Cloud ready.
3. Optionally pre-build the Modal sandbox image from a machine with the same Modal credentials:

   ```sh
   uv run python build_workspace_image.py
   ```

4. Start a **Cloud session**, leave the repo empty, send a tiny filesystem check (see [getting-started](getting-started.md)).
5. Connect GitHub from **Connections** after the first task works.

## Operations

- Redeploys interrupt active sessions — wait for idle turns.
- Back up the Coolify volume together with `ENCRYPTION_KEY`, `SESSION_SECRET`, and workspace passwords.
- Closing the browser does not stop Modal sandboxes; stop sessions in the UI and check Modal usage.
- For Temporal Cloud, Google SSO, Slack, or object storage, extend env from [.env.example](../.env.example) and [deployment.md](deployment.md).

# nelkinda models in Claude Code

Run local models served on `ai.nelkinda.com` (an M5 Max, 40-GPU, 128GB Mac,
speaking an OpenAI-compatible API) through **Claude Code**, Anthropic's CLI.

Claude Code only speaks Anthropic's `/v1/messages` API. nelkinda's models are
served OpenAI-style. This repo bridges the two with a small
[litellm](https://github.com/BerriAI/litellm) proxy: it exposes an
Anthropic-compatible endpoint on `localhost:4000` and forwards requests to
nelkinda underneath. Claude Code is pointed at that local endpoint instead of
`api.anthropic.com`.

The proxy runs as a systemd **user** service so it survives reboot and starts
even before you log in (via `loginctl enable-linger`).

Your normal `claude` command and your Anthropic account are untouched. A
separate wrapper, `claude-nelkinda`, launches Claude Code against the local
models instead — you choose per-invocation which backend you want.

## Available models

| alias          | actual model               |
|----------------|-----------------------------|
| `gemma-12b`    | `mac/gemma4:12b-mlx`        |
| `gemma-26b`    | `mac/gemma4:26b-mlx`        |
| `gemma-31b`    | `mac/gemma4:31b-mlx`        |
| `qwen-27b`     | `mac/qwen3.6:27b-mlx`  (default) |
| `qwen-27b-fp8` | `mac/qwen3.6:27b-mxfp8`     |
| `qwen-35b`     | `mac/qwen3.6:35b-mlx`       |

## How it fits together

```
claude-nelkinda (wrapper script)
  -> sets ANTHROPIC_BASE_URL=http://localhost:4000
  -> sets ANTHROPIC_MODEL=nelkinda-<alias>
  -> execs `claude`
       |
       v
litellm proxy (systemd user service, port 4000)
  -> translates Anthropic /v1/messages <-> OpenAI /v1/chat/completions
  -> forwards to https://ai.nelkinda.com/v1 with your NELKINDA_AI_API_KEY
```

## Prerequisites

- Claude Code installed (`claude` on PATH).
- `uv` (https://docs.astral.sh/uv/) for installing litellm as an isolated tool.
- Linux with systemd (user services). macOS users: adapt `litellm-nelkinda.service`
  to a `launchd` plist instead — same env vars and command.
- A nelkinda API key.

## Install

**1. Install litellm's proxy server.**

```bash
uv tool install --with 'fastapi<0.120,>=0.115' 'litellm[proxy]'
```

The fastapi version pin matters: newer fastapi releases removed an internal
function (`get_flat_dependant`) that litellm's proxy server imports, which
breaks startup with a cryptic `ModuleNotFoundError: No module named 'proxy_server'`
(the real error is buried above it in the traceback — an `ImportError` on that
fastapi symbol).

**2. Copy the config files into place.**

```bash
mkdir -p ~/.config/litellm ~/.config/systemd/user
cp config.yaml ~/.config/litellm/config.yaml
cp litellm-nelkinda.service ~/.config/systemd/user/
cp claude-nelkinda ~/.local/bin/
chmod +x ~/.local/bin/claude-nelkinda
```

Make sure `~/.local/bin` is on your `PATH`.

**3. Set up your secrets.** These are per-person and never committed.

```bash
cp proxy.env.example ~/.config/litellm/proxy.env
chmod 600 ~/.config/litellm/proxy.env
```

Edit `~/.config/litellm/proxy.env`:

- `NELKINDA_AI_API_KEY` — your real nelkinda key.
- `LITELLM_MASTER_KEY` — a secret you generate yourself, e.g.
  `openssl rand -hex 24`. This is the key Claude Code will use to talk to
  *your local proxy* — it's never sent to nelkinda, and nelkinda's own key
  never leaves the proxy process.

## Enable the service

```bash
systemctl --user daemon-reload
systemctl --user enable --now litellm-nelkinda.service
loginctl enable-linger "$USER"   # so it also starts on boot, before you log in
```

Verify:

```bash
curl -s http://localhost:4000/health/liveliness
# "I'm alive!"

systemctl --user status litellm-nelkinda
journalctl --user -u litellm-nelkinda -f   # live logs
```

## Use it

```bash
claude-nelkinda                     # default model: qwen-27b
claude-nelkinda qwen-35b            # pick a different model
claude-nelkinda gemma-26b -p "hi"   # any extra args pass through to `claude`
```

To switch models *mid-session*, use Claude Code's own command:

```
/model nelkinda-qwen-35b
```

(the model name must match one of the `model_name` entries in `config.yaml`,
prefixed `nelkinda-`).

Your regular `claude` command is completely unaffected — it still talks to
Anthropic normally. Only sessions launched via `claude-nelkinda` are redirected.

## Files in this repo

- **`config.yaml`** — litellm proxy config. Maps each nelkinda model to a
  `model_name` litellm exposes, all pointed at `https://ai.nelkinda.com/v1`.
  Includes `drop_params: true`, which is required: Claude Code sends a
  `thinking` parameter that nelkinda's OpenAI-compatible backend rejects with
  `UnsupportedParamsError` unless litellm is told to silently drop
  unsupported params.
- **`litellm-nelkinda.service`** — systemd user unit. Reads secrets from
  `~/.config/litellm/proxy.env` via `EnvironmentFile`, restarts on failure.
- **`claude-nelkinda`** — wrapper script. Sets `ANTHROPIC_BASE_URL`,
  `ANTHROPIC_AUTH_TOKEN`/`ANTHROPIC_API_KEY` (both set to your litellm master
  key), and `ANTHROPIC_MODEL`, then `exec`s `claude`. If the proxy isn't
  already running (e.g. systemd isn't set up yet), it starts one itself as a
  fallback.
- **`proxy.env.example`** — template for the secrets file. Copy to
  `~/.config/litellm/proxy.env` and fill in real values; never commit the
  real file (`.gitignore` blocks it).
- **`.gitignore`** — excludes `proxy.env`, `master_key.txt`, and `*.log` so
  nobody accidentally commits secrets.

## Troubleshooting

- **`Execution error` / hangs, then `UnsupportedParamsError: ... ['thinking']`
  in `journalctl --user -u litellm-nelkinda`** — `drop_params: true` missing
  from `config.yaml`, or the service wasn't restarted after adding it.
- **`AuthenticationError ... expected to start with 'sk-'`** — your
  `NELKINDA_AI_API_KEY` in `proxy.env` has stray quote characters in it (easy
  mistake if you copy it out of a shell-style `.env` file with `KEY='value'`
  syntax using a naive `cut`/`grep` — strip the quotes).
- **Proxy not reachable after reboot** — confirm `loginctl show-user "$USER" -p Linger`
  says `Linger=yes`; without it, user services only start after you log in.
- **Port 4000 already in use** — check for a stray manually-started `litellm`
  process (`pgrep -fa "litellm --config"`) and kill it; the systemd service
  should be the only one running it.

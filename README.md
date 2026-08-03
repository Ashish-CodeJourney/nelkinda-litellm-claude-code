# nelkinda models in Claude Code

Runs local `ai.nelkinda.com` OpenAI-compatible models through Claude Code, via a
litellm proxy that translates Anthropic's `/v1/messages` API to nelkinda's
OpenAI-style API. Runs as a systemd user service so it survives reboot.

## Install

```bash
uv tool install --with 'fastapi<0.120,>=0.115' 'litellm[proxy]'
```
(fastapi pin required — newer fastapi breaks litellm's proxy_server import.)

```bash
mkdir -p ~/.config/litellm ~/.config/systemd/user
cp config.yaml ~/.config/litellm/config.yaml
cp litellm-nelkinda.service ~/.config/systemd/user/
cp claude-nelkinda ~/.local/bin/
chmod +x ~/.local/bin/claude-nelkinda

cp proxy.env.example ~/.config/litellm/proxy.env
# edit ~/.config/litellm/proxy.env: fill NELKINDA_AI_API_KEY, generate LITELLM_MASTER_KEY
#   openssl rand -hex 24   # for LITELLM_MASTER_KEY
chmod 600 ~/.config/litellm/proxy.env
```

## Enable

```bash
systemctl --user daemon-reload
systemctl --user enable --now litellm-nelkinda.service
loginctl enable-linger "$USER"   # starts service at boot, without login
```

Check:
```bash
curl -s http://localhost:4000/health/liveliness
systemctl --user status litellm-nelkinda
```

## Use

```bash
claude-nelkinda                 # default model: qwen-27b
claude-nelkinda qwen-35b
claude-nelkinda gemma-26b -p "hello"
```

Aliases: `gemma-12b`, `gemma-26b`, `gemma-31b`, `qwen-27b`, `qwen-27b-fp8`, `qwen-35b`.

Mid-session model swap: `/model nelkinda-qwen-35b` inside a `claude-nelkinda` session.

Normal `claude` (Anthropic) is untouched — this only affects sessions launched
via `claude-nelkinda`.

## Files

- `config.yaml` — litellm proxy config, maps 6 nelkinda models, `drop_params: true`
  (Claude Code sends a `thinking` param the OpenAI-style backend rejects otherwise).
- `litellm-nelkinda.service` — systemd user unit, reads secrets from
  `~/.config/litellm/proxy.env` (not checked into this repo).
- `claude-nelkinda` — wrapper: sets `ANTHROPIC_BASE_URL`/`ANTHROPIC_AUTH_TOKEN`/
  `ANTHROPIC_MODEL` and execs `claude`. Falls back to starting the proxy itself
  if systemd isn't managing it.
- `proxy.env.example` — template for the real secrets file. Never commit the
  real `proxy.env` or master key.

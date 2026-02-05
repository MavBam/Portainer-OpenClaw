# How to install OpenClaw on Portainer
(formerly Clawdbot / Moltbot)

Status: Stable, repeatable, battle-tested
Deployment: Docker + Portainer
Note: No official Docker image exists
Gain source from https://github.com/openclaw/openclaw

---

## Overview

OpenClaw is a WhatsApp gateway and agent runtime built on Node.js.
This repository documents a proven, repeatable Docker and Portainer deployment built from source.

This setup provides:
- Durable session memory
- Stable WhatsApp gateway
- Memory flush prompts disabled
- Root execution inside the container
- Clean Docker volume separation
- Fully repeatable rebuilds

This documentation is written for operators, not framework contributors.

---

## Architecture

Containers:
- openclaw – main agent runtime
- openclaw-gateway – WhatsApp gateway service

Docker volumes (critical):

Volume: openclaw-state
Mount: /home/node/.moltbot
Purpose: Configuration and runtime state

Volume: openclaw-clawdbot
Mount: /home/node/.clawdbot
Purpose: Sessions and transcripts

Volume: openclaw-workspace
Mount: /home/node/clawd
Purpose: Workspace and outputs

These paths are mandatory.  OpenClaw resolves configuration via HOME, not CLI flags.

---

## Docker Image Build (Required)

There is no official image.  You must build from source.

So let's start:

First, navigate to a 'quiet directory [make one in your home folder if need be] and clone the latest version upstream:
```
git clone https://github.com/openclaw/openclaw.git
cd ~/openclaw
```
We need to edit out the non-root bits from the Dockerfile
```
nano Dockerfile
```
Comment out the below within the Docker file (using #):
```
RUN chown -R node:node /app
```
and 
```
USER node
```
Save the Dockerfile, and exit

Now build the image:

    docker build --no-cache --progress=plain -t openclaw:latest . 2>&1 | tee build.log
---

## Portainer Stack (Known-Good)

This is the canonical, working stack definition that is working for me, and forces a couple of things that were painfully long winded to remediate.
```
version: "3.9"

services:
  openclaw:
    image: openclaw:latest
    container_name: openclaw
    environment:
      HOME: /home/node
      OPENCLAW_GATEWAY_PASSWORD: "${OPENCLAW_GATEWAY_PASSWORD}"
    volumes:
      - openclaw-state:/home/node/.openclaw
      - openclaw-workspace:/home/node/openclaw
    command:
      - node
      - dist/index.js
      - gateway
      - run
      - --allow-unconfigured
      - --bind
      - 0.0.0.0
      - --port
      - "18789"
      - --auth
      - password
      - --password
      - "${OPENCLAW_GATEWAY_PASSWORD}"
      - --verbose

    ports:
      - "18789:18789"
    restart: unless-stopped


  caddy:
    image: caddy:2
    container_name: openclaw-caddy
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - /data/compose/34/Caddyfile:/etc/caddy/Caddyfile:ro

      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped
    depends_on:
      - openclaw

volumes:

  openclaw-state:
    name: openclaw-state
  openclaw-workspace:
    name: openclaw-workspace
  caddy_data:
  caddy_config:

```
---
Ensure you set an Environment Variable within the Stack Editor as follows
```
Name:  OPENCLAW_GATEWAY_PASSWORD
Value: {Whatever you'd like to set as a password}
```
---
Notes:
---
## Why user: root and HOME=/home/node

OpenClaw requires elevated privileges inside the container.
Sudo is meaningless in Docker.

When running as root, OpenClaw defaults to /root/.moltbot.
Setting HOME=/home/node forces config discovery to the mounted volume.

Without this, you will see:

    Missing config. Run `moltbot setup`

This is expected behaviour.

---

## Configuration (moltbot.json)

If the volumes are blank, run setup against the volumes you are using:
```
docker run --rm -it \
  -e HOME=/home/node \
  -v openclaw-state:/home/node/.openclaw \
  -v openclaw-workspace:/home/node/openclaw \
  openclaw:latest \
  node dist/index.js setup
```

Then verify the config exists on the host:
```
sudo ls -la /var/lib/docker/volumes/openclaw-state/_data | head -n 120
sudo ls -la /var/lib/docker/volumes/openclaw-state/_data/openclaw.json
```
---

## Session Durability

Session index:

    /home/node/.clawdbot/agents/main/sessions/sessions.json

Session files:

    /home/node/.clawdbot/agents/main/sessions/*.jsonl

Notes:
- Paths in sessions.json are container paths
- On the host, they live under:
  /var/lib/docker/volumes/openclaw-clawdbot/_data
- Do not rewrite sessions.json unless paths are genuinely missing

---

## Verification Checklist

Check containers:

    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.RestartCount}}"

Gateway logs:

    docker logs --since 10m openclaw-gateway

Expected:
- No pairing loops
- No memory flush prompts
- Normal message flow

---

## Common Errors

Missing config:
Cause: HOME not set correctly
Fix: Set HOME=/home/node

Unknown option --config:
Cause: OpenClaw does not support --config
Fix: Remove the flag

Memory flush messages return:
Cause: Wrong config file being read
Fix: Verify HOME and volumes, then restart

---

## Naming Volatility

Upstream renames include:
- Clawdbot
- Moltbot
- OpenClaw

This documentation is anchored in paths and behaviour, not branding.

---

## Philosophy

This setup is intentionally boring.

If you can rebuild, redeploy, restart, and upgrade without re-pairing WhatsApp or losing context, then it is correct.

---

License follows upstream licensing.
This documentation is provided as-is, battle-tested, and practical.

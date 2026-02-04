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

First, create a build area on your Portainer machine:

```
mkdir -p ~/openclaw
cd ~/openclaw
```

Now create the Dockerfile:

```
nano Dockerfile
```
Paste the below:

    FROM node:22

    WORKDIR /app

    RUN apt-get update && apt-get install -y \
        git \
        python3 \
        bash \
     && rm -rf /var/lib/apt/lists/*

    COPY . /app

    RUN corepack enable \
     && pnpm install --frozen-lockfile \
     && pnpm build

    CMD ["node", "/app/moltbot.mjs"]

Build the image:

    docker build -t openclaw:latest .

---

## Portainer Stack (Known-Good)

This is the canonical, working stack definition.

    version: "3.9"

    services:
      openclaw:
        image: openclaw:latest
        container_name: openclaw
        user: "0:0"
        environment:
          HOME: /home/node
        volumes:
          - openclaw-state:/home/node/.moltbot
          - openclaw-clawdbot:/home/node/.clawdbot
          - openclaw-workspace:/home/node/clawd
        restart: unless-stopped

      openclaw-gateway:
        image: openclaw:latest
        container_name: openclaw-gateway
        user: "0:0"
        environment:
          HOME: /home/node
          CLAWDBOT_GATEWAY_PASSWORD: "${CLAWDBOT_GATEWAY_PASSWORD}"
        volumes:
          - openclaw-state:/home/node/.moltbot
          - openclaw-clawdbot:/home/node/.clawdbot
          - openclaw-workspace:/home/node/clawd
        command: >
          node /app/moltbot.mjs gateway run
          --port 18789
          --bind lan
          --auth password
          --password "${CLAWDBOT_GATEWAY_PASSWORD}"
          --verbose
          --force
        ports:
          - "18789:18789"
        restart: unless-stopped

    volumes:
      openclaw-state:
      openclaw-clawdbot:
      openclaw-workspace:

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

Config path inside container:

    /home/node/.moltbot/moltbot.json

Known-good excerpt:

    agents: {
      defaults: {
        workspace: "/home/node/clawd",
        compaction: {
          mode: "default",
          memoryFlush: {
            enabled: false
          }
        },
        maxConcurrent: 1,
        subagents: {
          maxConcurrent: 8
        }
      }
    }

Memory flush is disabled because:
- It interrupts WhatsApp conversations
- It causes context confusion
- It is unnecessary with durable sessions

Disabling it does not break compaction.

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

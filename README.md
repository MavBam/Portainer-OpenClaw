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

This documentation is written for operators, not framework contributors.  It's also very much a work in progress.  Any issues, then I will attempt to assist.  This works fine on my Poprtainer instance; yours may differ.

---

## Architecture

Containers:
- openclaw – main agent runtime
- openclaw-caddy – https service

Docker volumes (critical):

Volume: openclaw-state

Mount: ```/home/node/.openclaw```

Purpose: Configuration and runtime state

Volume: openclaw-workspace

Mount: ```/home/node/openclaw```

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
      - "80:80"
      - "443:443"
    volumes:
      - /data/compose/34/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - openclaw
    restart: unless-stopped

volumes:
  openclaw-state:
    name: openclaw-state
  openclaw-workspace:
    name: openclaw-workspace
  caddy_data:
  caddy_config:


```
---
## Caddy
For Caddy, I wanted to provide a https environment.  I'm not proficient with it, but a lovely chap helped me.  For awareness, here's the config for my local network:

First, on each client machine:
- Windows: `C:\Windows\System32\drivers\etc\hosts`
- macOS/Linux: `/etc/hosts`

Add:
```
[IP of your Portainer host]  openclaw.lan
```
On the Portainer host, edit the Caddyfile:
```
nano /data/compose/34/Caddyfile
```
Replace all with the following:
```
{
	local_certs
}

http://openclaw.lan {
	redir https://openclaw.lan{uri} permanent
}

openclaw.lan {
	tls internal
	reverse_proxy openclaw:18789
}
```
Save, then restart Caddy
```
docker exec -it openclaw-caddy caddy reload --config /etc/caddy/Caddyfile
```
---
## Pair the console to the server
Go into remote shell:
```
docker exec -it --user node openclaw bash
```
OpenClaw pairing command:
```
node ./openclaw.mjs devices list
```
Copy Pending Request code and then insert it into:
```
node ./openclaw.mjs devices approve <ID_OR_CODE>
```
Confirm with:
```
node ./openclaw.mjs devices list
```
That should show that there are no pending requests.

---
Ensure you set an Environment Variable within the Stack Editor as follows
```
Name:  OPENCLAW_GATEWAY_PASSWORD
Value: {Whatever you'd like to set as a password}
```

---
## Notes:
---
## Why user: root and HOME=/home/node

OpenClaw requires elevated privileges inside the container.
Sudo is meaningless in Docker.

When running as root, OpenClaw defaults to ```/root/.openclaw```

Setting ```HOME=/home/node``` forces config discovery to the mounted volume.

Without this, you will see an error

This is expected behaviour.
---

## Configuration (openclaw.json)

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

---

## Naming Volatility

Upstream renames include:
- Clawdbot
- Moltbot
- OpenClaw

---

## Philosophy

This setup is intentionally boring.

If you can rebuild, redeploy, restart, and upgrade without re-pairing WhatsApp or losing context, then it is correct.

---

License follows upstream licensing.
This documentation is provided as-is, battle-tested, and practical.

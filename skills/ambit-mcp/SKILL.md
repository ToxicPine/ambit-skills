---
name: ambit-mcp
description: 'Use this skill whenever you are operating through the Ambit MCP server to deploy or manage apps on Fly.io. This covers the full MCP tool surface: app lifecycle, deployment, machines, networking, secrets, scaling, volumes, and router management. Trigger phrases include "deploy an app", "create an app", "set up a new service", "check app status", "scale an app", "view logs", "manage secrets", "set up a router", and any other Fly.io infrastructure task performed through MCP tools.'
license: MIT
metadata:
  author: ambit
  version: "0.1.0"
---

# Ambit MCP

## What This Is

You are operating through the Ambit MCP server. This server gives you access to Fly.io — a cloud platform for running apps — but wraps every operation in a safety layer designed for AI agents. The key constraint: **you can only ever deploy things onto a private network**. There is no option to make an app public, allocate a public IP, or expose anything to the internet. This is intentional. It means you can deploy freely without risk of accidentally making a database or internal tool reachable by strangers.

Apps you deploy land on a private Ambit network (e.g. `lab`, `staging`, `personal`). They get addresses like `http://my-app.lab` that work for any device enrolled in the user's Tailscale network, and are unreachable from anywhere else. Tailscale is a VPN that links the user's devices into a sealed private network — devices enrolled in it can connect to each other; devices that aren't cannot.

## Installation

If `ambit-mcp` is not already installed, run it directly via npx:

```bash
npx @cardelli/mcp
```

## Operating Modes

- **Safe Mode** (default): Private networking is enforced on every operation. Router management tools are available. Public IP allocation tools do not exist.
- **Unsafe Mode**: Full flyctl surface with no restrictions. No router tools.

You are most likely in safe mode. If a tool you expect is missing, check which mode is active by examining your available tool list.

## Standard Deployment Workflow

To get a new app running on the private network, call tools in this order:

1. **`fly_app_create`** — Create the app. In safe mode the network is set automatically.
2. **`fly_secrets_set` with `stage: true`** — Set any API keys, database URLs, or tokens *before* deploying. Using `stage: true` queues the secrets without triggering a redeploy on an app that has no code yet.
3. **`fly_deploy`** — Deploy the image or dockerfile. Safe mode auto-injects `--no-public-ips --flycast` and audits the result immediately.
4. **`fly_ip_allocate_flycast`** — Allocate a private Flycast IPv6 address on the correct network name. Without this step the app is running but nothing can reach it — the router has no address to forward traffic to.
5. **`fly_app_status` and `fly_logs`** — Verify machines are running and check for startup errors.

After this, the app is reachable from the user's tailnet as `http://<app-name>.<network>`.

## Key Rules

1. **Secrets go in `fly_secrets_set`, not `env` in `fly_deploy`.** Variables passed as `env` in a deploy are visible in the app config. Secrets are encrypted at rest and are the right place for anything sensitive.
2. **Always allocate a Flycast IP after deploying.** An app without a Flycast IP is running but unreachable. `fly_ip_allocate_flycast` must be called with the correct network name after every first deploy.
3. **Use `stage: true` when setting secrets before the first deploy.** Without it, each `fly_secrets_set` call triggers a redeploy attempt on an app with no code yet.
4. **Check `fly_auth_status` first when debugging.** Most failures trace back to expired or missing authentication.
5. **The `confirm` field on `fly_volumes_destroy` must exactly match `volume_id`.** Do not generate a different value or skip it.
6. **Router tools only exist in safe mode.** If `router_*` tools are absent from your tool list, the server is in unsafe mode.
7. **`fly_deploy` auditing is automatic in safe mode.** If public IPs are found after a deploy they are released automatically, but the deploy is flagged as an error — report this to the user.

## Troubleshooting

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| "Not authenticated" | Expired or missing fly CLI auth | Ask user to run `fly auth login` in their terminal |
| App not reachable after deploy | No Flycast IP allocated | Call `fly_ip_allocate_flycast` with the correct network name |
| "No ambit network configured" | No router deployed | Call `router_deploy` to create one first |
| Deploy succeeds but audit fails | Public IPs were allocated then auto-released | Check fly.toml for `force_https` or public TLS handlers |
| Machine stuck in bad state | Machine unresponsive | Call `fly_machine_destroy` with `force: true`, then redeploy |
| Router not appearing in tailnet | Tailscale auth issue | Call `router_logs` — most common cause is an expired Tailscale API key |

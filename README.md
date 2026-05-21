# Intility Developer Platform — Customer Companion

> Ship a containerized app to Kubernetes in minutes, even if you've never touched it before.

A Claude Code plugin that turns *"I have a Docker image"* into *"my app is live at https://myapp.apps.example.com"* — no manifests to write, no YAML to memorize, no Kubernetes book on the shelf.

[![Status: Early Alpha](https://img.shields.io/badge/Status-Early%20Alpha-orange)](https://github.com/intility/cust-devplatform-plugin/issues)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> ⚠️ **Early alpha.** This is the first cut of a customer-facing companion plugin for the Intility Developer Platform. Skill names, prompts, defaults, and behaviours will change. Don't depend on it for anything critical yet — but please [open an issue](https://github.com/intility/cust-devplatform-plugin/issues) when something feels off, surprises you, or could be clearer. That feedback is the whole point of an alpha.

---

## What you get

- **Create a cluster** — *"create a cluster"*
- **Deploy any container** — *"deploy my app"*
- **Get a real URL** — *"give it a URL"*
- **Ship a new version** — *"update to v2"*
- **See what's running** — *"what do I have running?"*

You bring a Dockerfile (or an image already in a registry). Claude does the rest.

## Quick start

1. **Install the prerequisites:**
   - [`indev`](https://developers.intility.com) — the Intility Developer Platform CLI
   - [`oc`](https://developers.intility.com) — the OpenShift CLI
2. **Install this plugin** in Claude Code (see below)
3. **Tell Claude what you want:**

   ```
   help me ship my app on Intility
   ```

That's the whole thing. The plugin detects where you are in the journey and walks you through the rest.

## Installing the plugin

If your team has the Intility marketplace configured:

```bash
/plugin marketplace add intility
/plugin install cust-devplatform-plugin@intility
```

Or clone it locally:

```bash
git clone https://github.com/intility/cust-devplatform-plugin \
  ~/.claude/plugins/cust-devplatform-plugin
```

## What's in the box

| Skill | Use it when… |
|---|---|
| `getting-started` | You're new and don't know where to begin |
| `status` | You're coming back and want to see what's running |
| `create-cluster` | You need a cluster |
| `login` | You need to (re)connect to a cluster |
| `prepare-app` | You want to check your app is ready to deploy |
| `deploy-app` | You're ready to put your app on the cluster |
| `expose-app` | You want a URL that points to your app |
| `update-image` | You've built a new version of your app |

All skills are user-invocable, so you can ask for any of them directly — or just describe what you want and the right one runs.

## A typical first session

```
You:    I have a Node app, image is at ghcr.io/me/shop:v1. Get it running.
Claude: No cluster yet — I'll create one called 'shop' and log you in.
        [creates cluster, waits ~7 minutes, OAuth login]
        Ready. What port does your app listen on?
You:    3000
Claude: [generates k8s/shop/ manifests, applies them, waits for rollout]
        Pods are running. Want a URL?
You:    yes
Claude: [creates HTTPRoute on the internal gateway]
        Live at http://shop.apps.example.com — try it out.
```

## Good to know

A few things that aren't obvious up front:

- **First run = clicking "Allow" a lot.** Claude Code asks permission the first time it runs each command. Allow them once and future sessions are quiet.
- **Tokens expire.** Both `indev` and `oc` log out after a few hours. If something fails with "Unauthorized", just say *"log me back in"*.
- **One cluster, many apps.** Each app gets its own namespace on a shared cluster. Works for most real workloads.
- **`internal` first, `public` only when asked.** Apps default to the internal gateway. The plugin will actively pause and ask twice before putting anything on the public internet.

## Found a bug? Have a wish?

Open an issue: **[github.com/intility/cust-devplatform-plugin/issues](https://github.com/intility/cust-devplatform-plugin/issues)**

Include:
- What you asked Claude to do
- What Claude tried (the failing command is the best clue)
- Cluster name, if relevant

For platform-level problems (cluster won't provision, gateways missing, can't log in at all), reach out to the **Developer Platform Admins** via your collaboration channel (samhandlingskanal).

## License

[MIT](LICENSE) — use it, fork it, ship it.

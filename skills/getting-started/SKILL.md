---
name: getting-started
description: Entry point for getting an app running on the Intility Developer Platform. Use when the user says things like "help me deploy", "get me started with Intility", "I want to run my app on the platform", "I'm new to Kubernetes and need to ship something", or "what do I do next?". Figures out where the user is in the journey (no cluster yet → cluster but no app → app deployed but no URL → ready to update) and routes to the right skill.
allowed-tools:
  - AskUserQuestion
  - Bash(command -v *)
  - Bash(indev account show)
  - Bash(indev cluster list*)
  - Bash(oc whoami*)
  - Bash(oc get*)
---

# Getting Started

## Who this is for

People who want to run an app on the Intility Developer Platform but don't have a deep Kubernetes background. The goal is to get them to a working, exposed app quickly, then help them learn as they go.

Keep the tone friendly and short. Don't lecture. When the user is about to learn something new, show them the command first, then a one-line explanation — not a paragraph.

## How this skill works

This skill is a router. It detects what state the user is in and hands off to the right skill. It does not do the work itself.

## Step 0 — Check the user's tools are installed

Before anything else, confirm the CLIs the plugin depends on are present:

```bash
command -v indev >/dev/null 2>&1 && echo "indev: ok" || echo "indev: MISSING"
command -v oc    >/dev/null 2>&1 && echo "oc: ok"    || echo "oc: MISSING"
command -v docker >/dev/null 2>&1 && echo "docker: ok" || echo "docker: missing (optional)"
```

- **`indev` missing** → stop. Send the user to https://developers.intility.com to install it. Nothing else works without this.
- **`oc` missing** → stop. Same page has install instructions.
- **`docker` missing** → fine to continue. Only `prepare-app` uses it, and that's optional.

If everything is present, don't say anything — move on silently. Only narrate the check if something is missing.

## Step 1 — Detect current state

Run these checks silently to figure out where the user is. Don't narrate every check.

```bash
indev account show
indev cluster list -o json 2>/dev/null
oc whoami 2>/dev/null
oc whoami --show-server 2>/dev/null
```

From the results, classify the situation:

| Situation | Signal |
|---|---|
| Not logged in to `indev` | `indev account show` fails |
| No cluster yet | `indev cluster list` returns empty / no clusters **and** `oc whoami` fails |
| Has a cluster, not logged in | Cluster exists, `oc whoami` fails |
| Logged in, no app deployed | `oc whoami` works, but the user hasn't mentioned an app namespace |
| Has an app, needs a URL | User says it's deployed but not reachable |
| Wants to update | User mentions a new image / new version |

**`oc whoami` working beats an empty `indev cluster list`.** Clusters are sometimes provisioned by platform admins or a colleague, so they don't show up in *this user's* `indev cluster list`. If `oc whoami` succeeds, the user has a cluster — treat the situation as "has a cluster, logged in" and never route to `create-cluster`. Use `oc whoami --show-server` to tell them which cluster they're connected to.

## Step 2 — Confirm where they want to go

If the situation is obvious from the user's message, just continue. Otherwise ask one short question:

```
Q: "What do you want to do?"
  Options:
    - "Get a cluster + deploy my first app"
    - "Deploy an app on the cluster I already have"
    - "Expose an app I already deployed"
    - "Update an app I already deployed"
```

Only include options that match the detected state. If they have no cluster, don't offer "expose an app".

## Step 3 — Hand off

Based on the destination, invoke the right skill in this order. Do not do these yourself — call the skill.

| Goal | Skill order |
|---|---|
| First time, full journey | `create-cluster` → `login` → `prepare-app` → `deploy-app` → `expose-app` |
| Has cluster, new app | `login` (if needed) → `prepare-app` → `deploy-app` → `expose-app` |
| Just need a URL | `expose-app` |
| Just bumping a version | `update-image` |
| Coming back after a break — what's running? | `status` |

If the user has a cluster and apps already deployed but doesn't remember what's there, run `status` first so they (and you) have an accurate picture before deciding what to do next.

Between skills, give the user a one-line "what's next" so they know the journey isn't over. Example:

> Cluster ready. Next: I'll get your app running on it.

## Step 4 — When they're done

After the app is exposed, give a short recap:

```
You're live at: http://<hostname>/

To ship a new version later, just ask: "update <app> to <new-image-tag>"
To add another app, just ask: "deploy another app" — it'll go in its own namespace on the same cluster.
```

## Tone reminders

- Plain language. Avoid "namespace", "manifest", "ingress" on first mention — say what they *do* first, then introduce the term once.
- One command at a time. Don't dump three commands and hope they pick one.
- Don't apologize for asking questions. One question, then move on.
- If they're stuck, suggest the next concrete step. Don't open with troubleshooting trees.

## Examples

See [references/usage-examples.md](references/usage-examples.md) for full conversational walkthroughs of common journeys (first-time setup, coming back after a break, adding a second app, missing prerequisites).

## What not to do

- Don't talk about ArgoCD, GitOps repos, or multi-cluster strategy. That's not what they're here for.
- Don't ask about team setup, RBAC, or quotas unless the platform actively refuses a command.
- Don't recommend creating more than one cluster. One cluster, many apps in their own namespaces.

---
name: create-cluster
description: Creates a single cluster on the Intility Developer Platform for an external user. Use when the user asks to "create a cluster", "get me a cluster", "set up a cluster", or is being routed here by the getting-started skill. Creates ONE cluster with sensible defaults unless the user explicitly asks for more, since this plugin assumes a many-apps-on-one-cluster model.
allowed-tools:
  - AskUserQuestion
  - Bash(indev account show)
  - Bash(indev cluster list*)
  - Bash(indev cluster create*)
  - Bash(indev cluster status*)
  - Bash(indev cluster get*)
  - Bash(indev login)
  - Bash(oc whoami*)
  - Bash(sleep *)
---

# Create Cluster

## Goal

Get the user one working cluster. Defaults that "just work" beat configurability. Don't pile on questions.

## If `indev` returns "Unauthorized" mid-flow

The `indev` token has expired. Don't retry the failing command. Run `indev login`, then resume from where you stopped.

## Step 1 — Already have a cluster?

```bash
indev cluster list
oc whoami --show-server 2>/dev/null
```

If there is already at least one cluster, **do not create another**. Show what's there and ask:

```
Q: "You already have a cluster (<name>). Use that one?"
  Options:
    - "Yes, use it" → hand off to the login skill
    - "No, I really need another" → continue
```

**Also check `oc whoami --show-server`** — clusters provisioned by platform admins or a colleague won't appear in this user's `indev cluster list`, but a working kubeconfig means a cluster exists. If `indev cluster list` is empty but `oc whoami --show-server` returns a server, ask:

```
Q: "You're already connected to a cluster (<server> — maybe set up by someone else?). Use that instead of creating a new one?"
  Options:
    - "Yes, use it" — stop here; they're already logged in, continue the journey on that cluster
    - "No, create my own" → continue
```

This plugin assumes one cluster, many apps in their own namespaces. Only create a second cluster if the user is sure.

## Step 2 — Authenticate

```bash
indev account show
```

If it errors, run:

```bash
indev login
```

This opens a browser. Wait for them to finish, then re-run `indev account show` to confirm.

## Step 3 — Pick a name

Ask for a name only — no preset, no node count, no autoscaling questions. Defaults handle those.

```
Q: "What do you want to call your cluster?"
  Options:
    - Other (free text — e.g. 'myapp', 'team-acme', 'sandbox')
```

Suggestions for naming: lowercase, no spaces, short and descriptive. The platform appends a random suffix automatically (e.g. `myapp-a3k9x2`), so the name doesn't need to be globally unique.

## Step 4 — Create it

Use the safe, cheap defaults — `minimal` preset, 2 fixed nodes. The user can grow later via the platform portal.

```bash
indev cluster create --name <name> --preset minimal --nodes 2
```

**Capture the full name from the output** (with the random suffix). You will need it for every later command.

Tell the user one sentence: "Creating your cluster — this takes 5–10 minutes."

## Step 5 — Wait for Ready

Poll status. Don't spam — once every 30 seconds is plenty.

```bash
indev cluster status <full-cluster-name>
```

States:
- `In Deployment` — keep waiting
- `Ready` — done, move on
- `Not Ready` — run `indev cluster get <full-cluster-name>` and show the user the message

If it's been more than 15 minutes and still `In Deployment`, stop polling and ask the user to reach out to the Developer Platform Admins via their collaboration channel (samhandlingskanal) with the cluster name.

## Step 6 — Hand off

Once `Ready`, tell the user briefly and invoke the **login** skill next:

```
Cluster '<full-cluster-name>' is ready. Logging you in now.
```

## What to skip

Do not ask about, recommend, or mention:
- Zones (Zone 2 is the default for this plugin's audience)
- Presets beyond `minimal` (they can resize via the portal later)
- Autoscaling (off by default; production-ready discussions are not why they're here)
- Multiple environments (no dev/staging/prod talk — one cluster is the model)
- Team setup, RBAC, quotas

If the user explicitly asks for any of these, answer briefly and point at the portal at https://developers.intility.com — don't try to do it here.

## Quick reference

```bash
indev cluster create --name <name> --preset minimal --nodes 2
indev cluster status <full-name>
indev cluster get <full-name>
indev cluster list
```

## Examples

See [references/usage-examples.md](references/usage-examples.md) for typical conversations (first cluster, already-have-one, not-logged-in, stuck provisioning).

---
name: cluster-login
description: Logs the user into their Intility Developer Platform cluster so they can run kubectl/oc commands against it. Use when the user asks to "log in", "log into the cluster", "connect to my cluster", "oc login", or when another skill needs cluster access and the user is not authenticated.
allowed-tools:
  - AskUserQuestion
  - Bash(indev cluster list*)
  - Bash(indev cluster login*)
  - Bash(oc whoami*)
  - Bash(oc get nodes*)
---

# Cluster login

## Goal

Get `oc whoami` to succeed against the user's cluster. Browser-based OAuth — they finish in the browser, you confirm in the terminal.

## Step 1 — Pick the cluster

If only one cluster exists, use it.

```bash
indev cluster list
```

If multiple, ask:

```
Q: "Which cluster do you want to log in to?"
  Options: [one per cluster name found]
```

## Step 2 — Run the login

```bash
indev cluster login <cluster-name>
```

This opens a browser. Tell the user one sentence:

> A browser will open. Sign in with your Intility account, then come back here.

If the browser doesn't open, suggest:

```bash
indev cluster login <cluster-name> --no-browser
```

…which prints a URL they can paste manually.

## Step 3 — Verify

After the user says they're done in the browser, confirm:

```bash
oc whoami
oc get nodes
```

If `oc whoami` returns their username and `oc get nodes` lists nodes (any status), they're in. Tell them briefly:

> Logged in as <user>. Your cluster has <N> node(s) ready.

## Step 4 — When it fails

- **Browser auth times out**: re-run `indev cluster login <cluster-name>`
- **`oc whoami` says "Unauthorized"**: token didn't get saved — re-run login
- **`oc get nodes` says "Forbidden"**: the account was created but no roles attached yet. Tell the user to check with their cluster owner or reach out to the Developer Platform Admins via their collaboration channel (samhandlingskanal).

Do not go further down the troubleshooting tree than this. External users should escalate; they shouldn't be debugging RBAC.

## Quick reference

```bash
indev cluster login <cluster-name>
oc whoami
oc get nodes
```

## Examples

See [references/usage-examples.md](references/usage-examples.md) for typical conversations (single cluster, multiple clusters, browser doesn't open, permissions missing).

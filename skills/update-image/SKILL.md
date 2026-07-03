---
name: update-image
description: Updates the container image on an already-deployed app to a new version. Use when the user says "update my app", "deploy a new version", "ship v2", "bump the image to X", or otherwise wants to roll out a new build of something already running on the cluster.
argument-hint: "[app] [new-image-or-tag]"
allowed-tools:
  - AskUserQuestion
  - Bash(oc whoami*)
  - Bash(oc get*)
  - Bash(oc set image*)
  - Bash(oc apply*)
  - Bash(oc rollout*)
  - Bash(oc describe*)
  - Bash(oc logs*)
  - Read
  - Edit
---

# Update Image

## Goal

Roll a deployed app to a new image tag. Then update the local manifest so the file matches what's running.

## Step 0 — Confirm access

```bash
oc whoami
```

If it errors, route to `cluster-login` first.

## If `oc` returns "Unauthorized" mid-flow

The `oc` token has expired. Don't retry the failing command. Stop where you are, route to the `cluster-login` skill, then resume from the failed step.

## Step 1 — Figure out which app

If the user named the app, use it. Otherwise:

```bash
oc get deployments --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name
```

Filter out the platform/system namespaces (`openshift-*`, `kube-*`, `envoy-gateway-system`, `argocd`, `default`). Ask:

```
Q: "Which app do you want to update?"
  Options: [one per remaining deployment, label as '<app> (in <namespace>)']
```

## Step 2 — Get the new image

```
Q: "What's the new image?"
  Options:
    - "Just a new tag — keep the same registry/repo" — ask for the tag
    - Other (free text — full image reference)
```

If they only have a tag, look up the current image to keep the registry/repo:

```bash
oc get deployment <app> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Strip the tag, append the new one. Confirm the full image with the user before applying.

## Step 3 — Apply the update

```bash
oc set image deployment/<app> <app>=<new-image> -n <namespace>
```

`oc set image` is the simplest path — it patches just the image field. The container name is usually the same as the app name (that's what `deploy-app` does). If `oc set image` complains about an unknown container, look it up:

```bash
oc get deployment <app> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].name}'
```

…and re-run with the right container name.

## Step 4 — Watch the rollout

```bash
oc rollout status deployment/<app> -n <namespace> --timeout=180s
```

If it succeeds, you're done. If it stalls or errors:

```bash
oc get pods -n <namespace>
oc describe pod <new-pod> -n <namespace>
oc logs <new-pod> -n <namespace>
```

Summarize the issue in one sentence — same triage table as `deploy-app`. If the new image is broken, offer to roll back:

```bash
oc rollout undo deployment/<app> -n <namespace>
```

…then look at the logs together to figure out what's wrong with the new image.

## Step 5 — Sync the local manifest

If a local `k8s/<app>/deployment.yaml` exists, update the `image:` line so the file matches what's running. This keeps the repo honest — next time someone re-applies the file, it won't downgrade the app.

If no local manifest exists, skip this step.

## Quick reference

```bash
# Update
oc set image deployment/<app> <app>=<new-image> -n <namespace>

# Watch
oc rollout status deployment/<app> -n <namespace>

# Roll back if needed
oc rollout undo deployment/<app> -n <namespace>

# See rollout history
oc rollout history deployment/<app> -n <namespace>
```

## Examples

See [references/usage-examples.md](references/usage-examples.md) for typical conversations (simple tag bump, different registry, rolling back a bad release, container-name mismatch, missing local manifest).

## What not to do

- Don't use `kubectl edit` / `oc edit` — opens an editor, fragile.
- Don't re-apply the whole `deployment.yaml` from disk unless the local file is already updated. Otherwise you'll roll back any image bumps that happened via `oc set image`.
- Don't suggest `latest` as a tag. Encourage tags that never move — version numbers like `v2`, `v3` — so it's always clear which build is running and rollback has something to roll back to.

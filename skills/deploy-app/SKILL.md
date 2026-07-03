---
name: deploy-app
description: Generates Kubernetes manifests (Namespace, Deployment, Service) for a containerized app and deploys them to the cluster with oc apply. Use when the user wants to "deploy my app", "ship my app", "run my app on the cluster", or after prepare-app has confirmed they have a containerized image. Defaults to a one-app-per-namespace layout so the same cluster can host many apps cleanly.
allowed-tools:
  - AskUserQuestion
  - Bash(oc whoami*)
  - Bash(oc get*)
  - Bash(oc apply*)
  - Bash(oc rollout*)
  - Bash(oc describe*)
  - Bash(oc logs*)
  - Bash(indev pullsecret*)
  - Bash(indev cluster pullsecret*)
  - Bash(indev cluster list*)
  - Bash(indev login)
  - Bash(mkdir *)
  - Read
  - Write
  - Edit
---

# Deploy App

## Goal

Generate three manifest files (`namespace.yaml`, `deployment.yaml`, `service.yaml`) and apply them. End state: pods are `Running` and there's a `Service` for the next skill (`expose-app`) to attach to.

## Step 0 — Confirm cluster access

```bash
oc whoami
```

If it errors, run the `cluster-login` skill first. Don't proceed without auth.

## If `oc` returns "Unauthorized" mid-flow

The `oc` token has expired. Don't retry the failing command. Stop where you are, route to the `cluster-login` skill, then resume from the failed step.

If an `indev` command says "Unauthorized" instead, run `indev login` and resume.

## Step 1 — Collect what you need

If you came from `prepare-app`, you already have:
- image
- container port

Now ask for:

```
Q1: "What do you want to call this app?"
  Options:
    - Other (free text — e.g. 'myapp', 'api', 'frontend')
  This becomes both the app name and the namespace.

Q2: "How many copies (replicas) should run?"
  Options:
    - "1" — fine for testing or a small app (Recommended)
    - "2" — basic redundancy
    - "3" — for higher availability
```

Defaults: replicas = 1, namespace = app name. Don't ask anything else here.

## Step 2 — Generate manifests

Use the templates in [references/manifest-templates.md](references/manifest-templates.md). Write three files into a local `k8s/<app-name>/` directory in the user's working directory:

```
k8s/<app-name>/namespace.yaml
k8s/<app-name>/deployment.yaml
k8s/<app-name>/service.yaml
```

If the directory doesn't exist, create it. Don't overwrite existing files without asking — if `deployment.yaml` already exists, ask whether to overwrite or to use `update-image` instead.

### Substitutions

| Placeholder | Value |
|---|---|
| `<app-name>` | from Step 1 |
| `<namespace>` | same as `<app-name>` |
| `<image>` | from `prepare-app` |
| `<container-port>` | from `prepare-app` |
| `<replicas>` | from Step 1 |

### Private image registry

If `prepare-app` flagged the registry as private, the cluster needs pull credentials. The platform handles this **once per cluster, not per app**, via `indev` (requires indev v1.4+). No `imagePullSecrets` in the manifests, no per-namespace secrets.

**1. Is there already a pull secret that covers this registry?**

```bash
indev pullsecret list
```

- One exists and includes the registry → confirm it's assigned to this cluster (`indev pullsecret get <name>` shows details). If so, nothing to do — continue with the deploy.
- One exists but for a different registry → add this one to it: `indev pullsecret registry add` (check `--help` for the format), then continue.
- None → create one (next step).

**2. Create it**

Ask the user for registry, username, and password/token (one `AskUserQuestion` with free-text options).

> **ghcr.io tip:** username = their GitHub username, token = a **classic** personal access token with the `read:packages` scope (fine-grained tokens don't work with ghcr), created at https://github.com/settings/tokens. Don't let them paste their GitHub password — it won't work.

```bash
indev pullsecret create --name <name> --registry <registry>:<username>:<token>
```

The registry's short name (e.g. `ghcr`) is a fine secret name.

**3. Assign it to the cluster**

```bash
indev cluster pullsecret set <full-cluster-name> <pull-secret-name>
```

That's it — the whole cluster can now pull from that registry. Every future private app on this cluster just works, no questions asked.

## Step 3 — Apply

Apply in order — namespace first so the others have somewhere to land.

```bash
oc apply -f k8s/<app-name>/namespace.yaml
oc apply -f k8s/<app-name>/deployment.yaml
oc apply -f k8s/<app-name>/service.yaml
```

## Step 4 — Watch the rollout

```bash
oc rollout status deployment/<app-name> -n <app-name> --timeout=180s
```

If it succeeds, you're done with deploy. Confirm with:

```bash
oc get pods -n <app-name>
```

All pods should be `Running` with `1/1` ready.

## Step 5 — If pods aren't Running

Don't dump troubleshooting trees on the user. Look at the pod yourself, then summarize in one sentence what's wrong.

```bash
oc get pods -n <app-name>
oc describe pod <pod-name> -n <app-name>
oc logs <pod-name> -n <app-name>
```

Common things and the one-line summary to give the user:

| What you see | Tell the user |
|---|---|
| `ImagePullBackOff` / `ErrImagePull` | "The cluster can't pull your image. Is the tag correct, and is the registry public? If it's private: does the cluster's pull secret cover it (`indev pullsecret list`), and is the token in it still valid? (PATs expire — recreate the pull secret and reassign it if so.)" |
| `CrashLoopBackOff` with logs showing app errors | Share the relevant log line. "Your app is crashing on startup — here's what it said: …" |
| `CreateContainerConfigError` | "Something in the manifest is invalid — usually a missing secret or bad env var." |
| Pending forever | "Cluster is out of room. Check `oc get nodes` and consider resizing in the portal." |

Offer to fix the manifest if the issue is something you can edit (wrong port, wrong image tag, etc.). For registry / cluster sizing issues, point at the right place.

## Step 6 — Hand off

Once the deployment is healthy, tell the user briefly:

```
Your app is running, but it doesn't have a URL yet. Want me to give it one?
```

If yes, invoke `expose-app`.

## Adding more apps later

Same skill. New name → new namespace, new manifests in `k8s/<new-app-name>/`. No cluster work needed. This is the "many apps on one cluster" model.

## References

- [references/manifest-templates.md](references/manifest-templates.md) — YAML templates for Namespace, Deployment, Service
- [references/usage-examples.md](references/usage-examples.md) — typical conversations (first deploy, private registry, image pull failures, crashing pods)

## What not to do

- Don't add `Ingress` or `Route` resources here — that's the `expose-app` skill.
- Don't add `HorizontalPodAutoscaler`, `PodDisruptionBudget`, `NetworkPolicy`, `ConfigMap`, or `Secret` unless the user asks. Start minimal.
- Don't add resource `requests`/`limits` automatically — the cluster has reasonable defaults and beginners trip over too-low limits. Add only when the user asks.
- Don't generate Helm or Kustomize. Plain YAML.

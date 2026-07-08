---
name: status
description: Shows a summary of what the user has on the Intility Developer Platform — their cluster, deployed apps, and the URLs those apps are exposed on. Use when the user asks "what's deployed?", "what do I have running?", "what apps are on my cluster?", "what's the URL for X?", "show me my cluster", or comes back after a break and needs a refresher.
allowed-tools:
  - Bash(indev cluster list*)
  - Bash(oc whoami*)
  - Bash(oc get*)
  - Bash(oc diff*)
  - Bash(grep *)
  - Glob
---

# Status

## Goal

Give the user a clear one-screen summary of: what cluster they have, what apps are deployed, and what URLs those apps respond on. Useful when coming back to the platform after a break, or just to remember what's where.

This skill is **read-only**. It changes nothing. It also persists nothing — it just queries. (`oc diff` in Step 5 is a server-side dry-run — it compares, it doesn't apply.)

## If `oc` returns "Unauthorized" mid-flow

The `oc` token has expired. Don't retry the failing command. Route the user to the `cluster-login` skill, then resume.

## Step 1 — Cluster

```bash
indev cluster list
```

If empty, tell the user they have no cluster yet and suggest `create-cluster`. Stop here.

Otherwise, capture the full cluster name (including suffix).

## Step 2 — Logged in?

```bash
oc whoami
```

If it errors, tell the user and suggest the `cluster-login` skill. Show the cluster name and basic info from `indev cluster list` first so they at least see what they have, then stop.

If it works, also grab the cluster name from the API URL — useful for showing hostnames:

```bash
oc whoami --show-server
```

The server URL looks like `https://api.<cluster-name>.<domain>:6443` — read the cluster name out of it yourself; no text-processing pipeline needed.

## Step 3 — Deployments

List all deployments excluding platform/system namespaces:

```bash
oc get deployments --all-namespaces \
  -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image \
  --no-headers \
  | grep -v -E '^(openshift|kube-|envoy-gateway-system|argocd|default |hypershift|olm)'
```

For each deployment, note: namespace, name, ready/desired, image.

If the list is empty, say so — the user has a cluster but nothing deployed. Suggest `deploy-app`.

## Step 4 — HTTPRoutes (URLs)

```bash
oc get httproute --all-namespaces \
  -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNAMES:.spec.hostnames,GATEWAY:.spec.parentRefs[0].name \
  --no-headers \
  2>/dev/null
```

Match each route to its deployment by namespace.

## Step 5 — Drift check (only when local manifests exist)

The manifests in `k8s/<app>/` are supposed to be the source of truth — but the cluster can wander (someone resized in the portal, ran `oc set image` from another machine, edited live). Catch that here, quietly.

First, look for managed manifests with the Glob tool (`k8s/*/*.yaml`). **If there's no `k8s/` directory here, skip this entire step silently** — no "drift not checked" disclaimers, just move on. The user may simply be in a different repo.

For each `k8s/<app>/` directory whose name matches a deployed namespace:

```bash
oc diff -f k8s/<app>/
```

Exit code 0 = in sync. Exit code 1 = something differs. Anything else = couldn't check that app — skip it quietly.

Reading the diff — **filter the noise first**. These differences alone do NOT count as drift:

- `metadata.generation`
- `creationTimestamp`
- `kubectl.kubernetes.io/last-applied-configuration` annotations
- anything under `status:`

If only noise remains, the app is in sync. For real differences, translate each into one plain phrase a human cares about: "cluster runs :v3, file says :v2", "replicas 3 on cluster, 1 in file", "cluster has an env var the file doesn't". Never paste the raw diff into the summary — the user can ask for it.

## Step 6 — Present

Show one block per app. Keep it tight — this is a glance, not a report.

```
Cluster: <full-cluster-name>  (logged in as <user>)

Apps:
  ┌─ myapp (namespace: myapp)
  │  image:  ghcr.io/me/myapp:v2
  │  status: 1/1 running
  │  URL:    http://myapp.apps.example.com  (internal — only reachable on your organization's network)
  │  files:  ✓ in sync with k8s/myapp/
  │
  ┌─ api (namespace: api)
  │  image:  ghcr.io/me/api:latest
  │  status: 2/2 running
  │  URL:    (not exposed — run expose-app to give it a URL)
  │  files:  ⚠ drift — cluster runs 2 replicas, k8s/api/deployment.yaml says 1
```

Only include the `files:` line when Step 5 actually ran for that app. No local manifests → no line, no apology.

Status notation:
- `1/1 running` → ready
- `0/1 running` → broken — also surface a one-liner: "may be crashing — try `oc logs deployment/<name> -n <ns>`"
- No httproute for a namespace → say "not exposed"

For the gateway label, derive it from the parentRef name:
- `internal` → `(internal — only reachable on your organization's network)`
- `public` → `(public — reachable from the open internet)` — flag this visually so the user notices, e.g. with a `⚠` prefix or by underlining it

If you see a `public` route the user may have set up earlier and forgotten about, mention it explicitly at the end of the report: "FYI, `<app>` is on the public gateway — make sure that's still intended."

## Step 7 — Tell them what's next

End with two-to-three relevant follow-ups based on what you saw. Examples:

- If an app shows `0/1 running`: "Want me to look at the logs for `<name>`?"
- If an app has no URL: "Want me to expose `<name>` on a URL?"
- If an app drifted: "The cluster and your files disagree for `<name>` — want me to bring them back in sync?"
- If everything looks healthy: "Anything you'd like to update or add?"

### Reconciling drift (when they say yes)

This happens *after* the report, with the user's go-ahead — never automatically. Ask one question: which side is right?

- **The file is right** (someone fiddled with the cluster): `oc apply -f k8s/<app>/` puts it back. This is the normal answer — the repo is the source of truth.
- **The cluster is right** (the change was intentional, e.g. someone resized on purpose): edit the local YAML to match instead, like `update-image` does. Nothing is applied; the file just catches up.

If they're unsure which, show them the relevant diff lines and let them decide. Don't pick for them — reverting someone's intentional change is worse than living with drift for another day.

Don't bullet a long menu — pick the most obvious one or two.

## What not to do

- Don't `oc describe` or `oc logs` automatically — that's noise. Offer it as a follow-up.
- Don't list pods, configmaps, secrets, services, networkpolicies. The user wants apps + URLs, not raw Kubernetes object soup.
- Don't include platform-namespace stuff (`openshift-*`, `kube-*`, `envoy-gateway-system`, `argocd`, `olm`, `hypershift`). It's noise to them.
- Don't try to derive resource usage / CPU / memory. Out of scope.
- Don't paste raw `oc diff` output into the summary — translate it to one plain phrase per difference.
- Don't mention drift at all when there are no local manifests to compare against.
- Don't reconcile drift inside this skill — report it, offer, and act only on the user's answer.

## Quick reference

```bash
indev cluster list
oc whoami
oc get deployments -A
oc get httproute -A
oc diff -f k8s/<app>/   # exit 0 = in sync, 1 = drift
```

## Examples

See [references/usage-examples.md](references/usage-examples.md) for typical conversations (healthy cluster, broken app, public-route warning, drifted manifests, logged out, nothing deployed).

---
name: expose-app
description: Gives a deployed app a public or internal URL by creating an HTTPRoute on the Intility Developer Platform. Use when the user says "expose my app", "give my app a URL", "make my app accessible", "set up ingress", "create a route", or after deploy-app finishes and the user wants the app reachable.
allowed-tools:
  - AskUserQuestion
  - Bash(oc whoami*)
  - Bash(oc get*)
  - Bash(oc apply*)
  - Bash(oc describe*)
  - Bash(oc delete httproute*)
  - Bash(curl -s -o /dev/null*)
  - Read
  - Write
  - Edit
---

# Expose App

## Goal

Make a deployed app reachable on a URL by creating an `HTTPRoute`. The platform team has already set up the underlying gateways — you only create the route.

End state: `curl http://<hostname>/` returns a non-error status code.

## Background (one paragraph, share with the user as needed)

The platform runs **Envoy Gateway**. Two pre-built gateways are typically available: `internal` (only reachable from inside the customer's network) and `public` (open to the internet). You don't create gateways — you just create an HTTPRoute that points at one. TLS is handled for you, so HTTPRoutes use plain HTTP on port 80; users still get HTTPS at the URL.

The domain each gateway uses is the customer's own — not hardcoded in this skill. We read it from the gateway's listener config.

## Step 0 — Confirm access

```bash
oc whoami
```

If it errors, route to `login` first.

## If `oc` returns "Unauthorized" mid-flow

The `oc` token has expired. Don't retry the failing command. Stop where you are, route to the `login` skill, then resume from the failed step.

## Step 1 — Find the gateways

```bash
oc get gateways -n envoy-gateway-system
```

Look at the `PROGRAMMED` column. Only offer gateways that show `True`. If no gateway is programmed, stop and tell the user to reach out to the Developer Platform Admins via their collaboration channel (samhandlingskanal) — the cluster isn't ready for ingress.

For each programmed gateway, read its listener hostname template — that's the customer's domain. The pattern is usually a wildcard like `*.apps.example.com`:

```bash
oc get gateway <gateway-name> -n envoy-gateway-system \
  -o jsonpath='{range .spec.listeners[*]}{.hostname}{"\n"}{end}'
```

Capture the hostname template for each gateway. You'll substitute the `*` with the app's subdomain in Step 4.

If a gateway has no wildcard listener (or returns empty), ask the user what domain to use for that gateway — `oc describe gateway <name> -n envoy-gateway-system` will show what hostnames the gateway permits.

## Step 2 — Pick which gateway

**Default to `internal` always.** It's the safe choice and what almost every app should use.

If only `internal` is programmed, use it without asking. Otherwise:

```
Q: "Who should be able to reach this app?"
  Options:
    - "Only people on the internal network (internal)" — internal tools, dashboards, APIs, anything not meant for the open internet (Recommended)
    - "Anyone on the internet (public)" — for apps you've deliberately decided should be publicly reachable
```

### If the user picks `public` — confirm explicitly

Putting an app on the public gateway means anyone on the internet can hit it. Most apps shouldn't be there. Before generating the manifest, stop and ask a follow-up confirmation:

```
Q: "Just to confirm — putting this on the public gateway means anyone on the internet can reach it. Are you sure?"
  Options:
    - "Yes, this is meant to be public (e.g. marketing site, public API, customer-facing app)"
    - "No, switch to internal" — go back and use the internal gateway instead
```

Before showing this question, also surface the security considerations in one short paragraph. Don't lecture — list the things that change:

> Exposing on `public` means:
> - The app is reachable from anywhere on the internet, not just your network.
> - You become responsible for the app's own authentication — there's no network-level gatekeeping anymore.
> - Any unprotected admin pages, debug endpoints, or unauthenticated APIs are exposed to bots and scanners within minutes.
> - Secrets in URLs, verbose error pages, or dev-mode flags will be visible to the world.
>
> If the app doesn't have its own login / auth, or you haven't reviewed what endpoints it exposes, choose `internal`.

Only proceed with `public` if the user explicitly confirms after seeing this. If they choose "switch to internal" or hesitate, use `internal`.

## Step 3 — Find the Service to point at

If the user came from `deploy-app`, you already have:
- `<app-name>` (used as both deployment and service name)
- `<namespace>`

If not, look them up. List Services in non-system namespaces:

```bash
oc get svc --all-namespaces \
  -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,PORT:.spec.ports[0].port \
  | grep -v -E '^(openshift-|kube-|envoy-gateway-system|argocd|default )'
```

Ask which to expose:

```
Q: "Which app do you want to expose?"
  Options: [one per Service, label '<name> (in <namespace>, port <port>)']
```

The Service from `deploy-app` listens on `port: 80` — that's what the HTTPRoute connects to.

## Step 4 — Subdomain and final hostname

Default the subdomain to the app/service name. Only ask if the user has indicated they want something different.

Build the hostname by substituting the subdomain into the gateway's listener template from Step 1:

```
listener template: *.apps.example.com
subdomain:         myapp
→ hostname:        myapp.apps.example.com
```

Confirm the full hostname back to the user in one line before generating the manifest — easy to catch typos and avoids surprises.

## Step 5 — Generate the HTTPRoute

Use the template from [references/httproute-templates.md](references/httproute-templates.md). Write the file to `k8s/<app-name>/httproute.yaml` (alongside the other manifests, so the user can see the whole app in one place).

Substitutions:

| Placeholder | Value |
|---|---|
| `<route-name>` | `<app-name>` |
| `<namespace>` | `<app-name>` |
| `<gateway-name>` | `internal` or `public` |
| `<hostname>` | from Step 4 |
| `<service-name>` | the Service name (same as `<app-name>` if from `deploy-app`) |
| `<service-port>` | `80` (the Service's port, not the container port) |

## Step 6 — Apply and verify

```bash
oc apply -f k8s/<app-name>/httproute.yaml
```

Wait ~10 seconds for the gateway controller to pick it up, then check it attached:

```bash
oc describe httproute <route-name> -n <namespace>
```

In the `Status` block, look for:

```
Parents:
  Parent Ref:
    Name: <gateway-name>
  Conditions:
    Type:   Accepted
    Status: True
    Type:   ResolvedRefs
    Status: True
```

Both must be `True`. If `Accepted` is `False`, the hostname conflicts with another route or the gateway rejected it — show the user the condition message. If `ResolvedRefs` is `False`, the Service name or port is wrong — double-check Step 5 substitutions.

## Step 7 — Smoke test

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://<hostname>/
```

- `200`–`399` → working
- `502` / `503` → route is attached but the pod isn't responding. Check `oc get pods -n <namespace>` — is it `Running`? Are the ports right (`Service.targetPort` should match the container port)?
- Timeout on the `internal` gateway from a remote machine → likely a network reachability issue. The internal hostname may only resolve from inside the customer's network or via VPN. Ask: "Are you connected to the network this cluster is normally reached from?"

## Step 8 — Report

```
Your app is live at: http://<hostname>/
(Gateway: <gateway-name> — <internal-only | open to the internet>)

To change the URL later, edit the `hostnames` field in k8s/<app-name>/httproute.yaml and re-apply.
```

If the gateway was `public`, include one extra line in the report:

> Reminder: this app is reachable from the open internet. Make sure its own authentication, debug pages, and error responses are reviewed before sharing the URL.

## Changing your mind later

- **Switch from `internal` to `public` (or back)**: edit `httproute.yaml` — change `parentRefs[0].name` and update the `hostnames` field to the new gateway's domain (re-run Step 1 to get the new listener template), then `oc apply -f`. Old route is overwritten in place. **If switching to `public`, redo the security check from Step 2.**
- **Take it down**: `oc delete -f k8s/<app-name>/httproute.yaml`.

## References

- [references/httproute-templates.md](references/httproute-templates.md) — YAML templates for simple, path-split, and multi-hostname routes
- [references/usage-examples.md](references/usage-examples.md) — typical conversations (internal URL, public confirmation flow, route conflicts, 502 troubleshooting)

## What not to do

- Don't create `Ingress` or `Route` resources. This platform uses HTTPRoute (Gateway API).
- Don't try to create a `Gateway` — the platform team has already created them. You only create HTTPRoutes.
- Don't add TLS config to the HTTPRoute. TLS terminates at the gateway; you talk plain HTTP behind it.
- Don't suggest `NodePort` or `LoadBalancer` Services. The HTTPRoute → ClusterIP Service is the path.

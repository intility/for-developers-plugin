# Usage Examples — Getting Started

## Example 1 — Full first-time journey

User has nothing set up yet. Plugin walks them through cluster → login → deploy → expose.

```
You:    I have a Docker image at ghcr.io/me/shop:v1 (Node, port 3000). Get it running on Intility.

Claude: [Step 0] indev ✓, oc ✓, docker ✓
        [Step 1] no clusters yet, not logged in

        You'll need a cluster first. I'll create one, log you in, then deploy your app.
        → invoking create-cluster

Claude: What do you want to call your cluster?
You:    shop

Claude: [indev cluster create --name shop --preset minimal --nodes 2]
        Creating your cluster (shop-a3k9x2) — this takes 5–10 minutes.
        … 7 minutes later …
        Ready. Logging you in now.
        → invoking login

Claude: A browser will open. Sign in with your Intility account, then come back here.
You:    [signs in]

Claude: Logged in. 2 nodes ready.
        → invoking prepare-app

Claude: I see you already have the image (ghcr.io/me/shop:v1) and port (3000). Skipping ahead.
        → invoking deploy-app

Claude: What do you want to call this app?
You:    shop

Claude: How many copies should run?
You:    1

        [writes k8s/shop/{namespace,deployment,service}.yaml]
        [oc apply -f x3]
        [oc rollout status deployment/shop -n shop --timeout=180s → success]

        Your app is running. Want me to give it a URL?
You:    yes

Claude: → invoking expose-app
        [oc get gateways → internal programmed, public programmed]
        [reads internal listener → *.apps.example.com]

        Default is the 'internal' gateway (only your network, safer). Use that?
You:    yes

        [writes k8s/shop/httproute.yaml, applies, verifies Accepted=True]
        [curl http://shop.apps.example.com/ → 200]

        Your app is live at: http://shop.apps.example.com/
```

---

## Example 2 — Coming back after a break

User had everything working a week ago. Comes back unsure what's still there.

```
You:    I think I deployed something last week but I forgot what. Show me.

Claude: → invoking status
        [indev cluster list → shop-a3k9x2]
        [oc whoami → ok]
        [oc get deployments -A, oc get httproute -A]

        Cluster: shop-a3k9x2  (logged in as you@intility.no)

        Apps:
          ┌─ shop (namespace: shop)
          │  image:  ghcr.io/me/shop:v1
          │  status: 1/1 running
          │  URL:    http://shop.apps.example.com  (internal)

        Want to update shop, add another app, or change something?
```

---

## Example 3 — Adding a second app

User has `shop` running. Wants to deploy `api` alongside it on the same cluster.

```
You:    Add another one — api at ghcr.io/me/api:v1, port 8000.

Claude: → invoking deploy-app (skipping create-cluster, you're already in)
        → invoking prepare-app to confirm details
        Confirmed: image ghcr.io/me/api:v1, port 8000.

Claude: What do you want to call this app?
You:    api

Claude: How many copies should run?
You:    2

        [writes k8s/api/{namespace,deployment,service}.yaml]
        [oc apply x3, rollout status → success]

        api is running alongside shop. Want a URL for it too?
```

Note: api lives in its own namespace, completely isolated from shop. Same cluster, different worlds.

---

## Example 4 — Tooling missing

User hasn't installed the prerequisites.

```
You:    Help me deploy my app.

Claude: [Step 0]
        indev: MISSING

        You'll need the indev CLI before we can do anything else. Install it from
        https://developers.intility.com, then come back.
```

---

## Common phrasings that trigger getting-started

- "help me get started with Intility"
- "I want to deploy my app"
- "I'm new to Kubernetes and need to ship something"
- "what do I do?"
- "set me up on the developer platform"

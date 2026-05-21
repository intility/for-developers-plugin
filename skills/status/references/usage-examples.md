# Usage Examples — Status

## Example 1 — Coming back, everything healthy

```
You:    What do I have running?

Claude: [indev cluster list → shop-a3k9x2]
        [oc whoami → ok]
        [oc get deployments -A, oc get httproute -A]

        Cluster: shop-a3k9x2  (logged in as you@intility.no)

        Apps:
          ┌─ shop (namespace: shop)
          │  image:  ghcr.io/me/shop:v2
          │  status: 1/1 running
          │  URL:    http://shop.apps.example.com  (internal)
          │
          ┌─ api (namespace: api)
          │  image:  ghcr.io/me/api:v1
          │  status: 2/2 running
          │  URL:    (not exposed — run expose-app to give it a URL)

        Want to expose api on a URL?
```

---

## Example 2 — An app is broken

```
You:    Show me my cluster.

Claude: Cluster: shop-a3k9x2

        Apps:
          ┌─ shop (namespace: shop)
          │  image:  ghcr.io/me/shop:v3
          │  status: 0/1 running — may be crashing
          │  URL:    http://shop.apps.example.com  (internal)

        shop has 0 of 1 pods ready. Want me to look at the logs?
You:    yes

Claude: [oc logs deployment/shop -n shop --tail=50]
        … shows the error …
        Looks like a missing env var. Did you intend `DATABASE_URL` to be
        set in this version?
```

---

## Example 3 — Public route surfaces with a warning

```
Claude: Cluster: shop-a3k9x2

        Apps:
          ┌─ marketing (namespace: marketing)
          │  image:  ghcr.io/me/marketing:v4
          │  status: 1/1 running
          │  URL:    http://www.apps.example.com  ⚠ (public — reachable from the open internet)

        FYI, marketing is on the public gateway — make sure that's still
        intended.
```

---

## Example 4 — Logged out

```
You:    What's running?

Claude: [indev cluster list → shop-a3k9x2]
        [oc whoami → error: unauthorized]

        I can see you have a cluster (shop-a3k9x2), but your session has
        expired. Want me to log you back in so I can show what's deployed?
```

---

## Example 5 — Nothing deployed yet

```
You:    Show me what I've got.

Claude: Cluster: shop-a3k9x2 (logged in as you@intility.no)

        No apps deployed yet. Want to deploy one?
```

---

## Common phrasings that trigger status

- "what do I have running?"
- "show me my cluster"
- "what's deployed?"
- "what URL is X on?"
- "what apps are on my cluster?"
- "I came back, what do I have?"

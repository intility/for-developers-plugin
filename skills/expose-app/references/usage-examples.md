# Usage Examples — Expose App

## Example 1 — Internal URL (the normal case)

```
You:    Give my shop app a URL.

Claude: [oc get gateways -n envoy-gateway-system]
          NAME       PROGRAMMED
          internal   True
          public     True
        [oc get gateway internal -o jsonpath=... → *.apps.example.com]

        Default is the 'internal' gateway — only reachable on your network.
        That's the safer choice. Use it?
You:    Yes

        [writes k8s/shop/httproute.yaml with hostname shop.apps.example.com]
        [oc apply -f k8s/shop/httproute.yaml]
        … 10s wait …
        [oc describe httproute shop -n shop → Accepted=True, ResolvedRefs=True]
        [curl http://shop.apps.example.com/ → 200]

        Your app is live at: http://shop.apps.example.com/
        (Gateway: internal — only reachable on the customer network)
```

---

## Example 2 — Public URL (requires explicit confirmation)

```
You:    Make my marketing site public, it's at shop.

Claude: [gateways: internal True, public True]
        [internal listener: *.apps.example.com]
        [public listener:  *.apps.example.com]

        Default is 'internal' (safer). Which do you want?
        [ ] Only people on the internal network (Recommended)
        [ ] Anyone on the internet (public)
You:    public

Claude: Exposing on `public` means:
          - The app is reachable from anywhere on the internet, not just your network.
          - You become responsible for the app's own authentication — there's
            no network-level gatekeeping anymore.
          - Any unprotected admin pages, debug endpoints, or unauthenticated
            APIs are exposed to bots and scanners within minutes.
          - Secrets in URLs, verbose error pages, or dev-mode flags will be
            visible to the world.

        If the app doesn't have its own login / auth, or you haven't reviewed
        what endpoints it exposes, choose `internal`.

        Just to confirm — are you sure you want this on the public gateway?
        [ ] Yes, this is meant to be public
        [ ] No, switch to internal

You:    Yes, it's a marketing site, all public.

Claude: [writes httproute with parentRef: public]
        [oc apply, verify Accepted=True]

        Your app is live at: http://shop.apps.example.com/
        (Gateway: public — reachable from the open internet)

        Reminder: this app is reachable from the open internet. Make sure its
        own authentication, debug pages, and error responses are reviewed
        before sharing the URL.
```

---

## Example 3 — Only `internal` is programmed

No question gets asked — use it directly.

```
You:    Expose my api app.

Claude: [gateways: internal True, public False]

        Using the internal gateway. Hostname will be api.apps.example.com.

        [writes httproute, applies, verifies, smoke-tests]
        api is live at http://api.apps.example.com.
```

---

## Example 4 — Route not accepted

```
Claude: [oc describe httproute shop -n shop]
          Status:
            Parents:
              Conditions:
                Type:   Accepted
                Status: False
                Message: hostname "shop.apps.example.com" already used by HTTPRoute "shop" in namespace "old-shop"

        The hostname `shop.apps.example.com` is already taken by another route
        (`old-shop` namespace). Two options:

        1. Use a different subdomain for this app (e.g. `shop-v2`)
        2. Delete the old route first (only if `old-shop` is no longer in use)

        Which do you want?
```

---

## Example 5 — Smoke test returns 502

```
Claude: [curl http://shop.apps.example.com/ → 502]

        The route is attached, but the backend isn't responding. Let me check
        the pod.
        [oc get pods -n shop → shop-xxx 1/1 Running]
        [oc get svc shop -n shop → targetPort: 3000]

        Pod is running on port 3000, service forwards to it. The 502 is most
        likely your app responding slowly or not listening on 0.0.0.0.

        Quick check: does your app bind to 0.0.0.0:3000 (not 127.0.0.1)?
        Containers can't be reached via localhost.
```

---

## What this skill won't do

- Create OpenShift `Route` or `Ingress` resources — only `HTTPRoute`
- Create a `Gateway` (those are pre-provisioned by the platform)
- Add TLS config to the HTTPRoute (TLS terminates at the gateway)
- Expose on a public gateway without explicit two-step confirmation

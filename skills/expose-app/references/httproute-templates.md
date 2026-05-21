# HTTPRoute Templates

YAML for exposing an app via the platform's pre-built gateways. Substitute the placeholders, write the file, `oc apply -f`.

## Placeholders

| Placeholder | Example |
|---|---|
| `<route-name>` | `myshop` |
| `<namespace>` | `myshop` — same as the app's namespace |
| `<gateway-name>` | `internal` (preferred) or `public` (only when explicitly needed) |
| `<hostname>` | `myshop.apps.example.com` — built by substituting the subdomain into the gateway's listener template |
| `<service-name>` | `myshop` — the Service to send traffic to |
| `<service-port>` | `80` — the Service's `port`, not the container's `targetPort` |

## How the hostname is built

The platform team provisioned each gateway with a listener hostname template, typically a wildcard such as `*.apps.example.com`. Read it with:

```bash
oc get gateway <gateway-name> -n envoy-gateway-system \
  -o jsonpath='{range .spec.listeners[*]}{.hostname}{"\n"}{end}'
```

Substitute the `*` with the app's subdomain (default: the app name).

| Gateway | Reachable from |
|---|---|
| `internal` | Only the customer's internal network (the safe default for almost every app) |
| `public` | Anywhere on the internet — only when the app is meant to be public-facing |

The actual domain (`example.com` above) is the customer's own; this skill never assumes a specific domain.

---

## Template — single app, all paths

This covers ~95% of cases. One service, all traffic.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <route-name>
  namespace: <namespace>
spec:
  parentRefs:
    - name: <gateway-name>
      namespace: envoy-gateway-system
  hostnames:
    - <hostname>
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: <service-name>
          port: <service-port>
```

### Example — internal app (the normal case)

Gateway listener: `*.apps.acme.com`, app called `myshop`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myshop
  namespace: myshop
spec:
  parentRefs:
    - name: internal
      namespace: envoy-gateway-system
  hostnames:
    - myshop.apps.acme.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: myshop
          port: 80
```

### Example — public-facing app (only after explicit confirmation)

Same shape, just swap the gateway name and use the public gateway's listener domain (which may be the same domain as `internal` or a different one — check what the platform configured):

```yaml
  parentRefs:
    - name: public
      namespace: envoy-gateway-system
  hostnames:
    - myshop.apps.acme.com
```

**Reminder for `public`:** the app is now reachable from the open internet. Verify its own authentication, hide debug endpoints, and check error pages don't leak internals before sharing the URL.

---

## Template — split a frontend and an API on the same hostname

Only use this if the user actually asks. Order matters: longer/more-specific paths first.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <route-name>
  namespace: <namespace>
spec:
  parentRefs:
    - name: <gateway-name>
      namespace: envoy-gateway-system
  hostnames:
    - <hostname>
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: <api-service-name>
          port: <api-service-port>
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: <frontend-service-name>
          port: <frontend-service-port>
```

## Notes

- `parentRefs[].namespace` **must** be `envoy-gateway-system`. That's where the platform's gateways live.
- The HTTPRoute itself lives in the same namespace as the Service it points to.
- Don't add a `port:` under `parentRefs` — the gateway listens on 80, that's the default.
- Subdomain defaults to the app name. Anything goes as long as it's DNS-valid (lowercase, letters/digits/dashes).
- Prefer `internal` unless the app is explicitly meant for the open internet. `public` requires an extra confirmation step in the `expose-app` skill for good reason.

# Usage Examples — Deploy App

## Example 1 — First deploy of a public-image app

```
You:    Deploy ghcr.io/me/shop:v1, port 3000.

Claude: [oc whoami → ok]

        What do you want to call this app?
You:    shop

Claude: How many copies should run?
You:    1

        [mkdir -p k8s/shop]
        [writes k8s/shop/namespace.yaml, deployment.yaml, service.yaml]
        [oc apply -f k8s/shop/namespace.yaml]
        [oc apply -f k8s/shop/deployment.yaml]
        [oc apply -f k8s/shop/service.yaml]
        [oc rollout status deployment/shop -n shop --timeout=180s → success]
        [oc get pods -n shop → shop-7b9-xxx 1/1 Running]

        Your app is running, but it doesn't have a URL yet. Want me to give it one?
```

---

## Example 2 — Deploy with a private registry

```
You:    Deploy private.registry.example.com/team/api:v2, port 8000.

Claude: [oc whoami → ok]

        What do you want to call this app?
You:    api

Claude: How many copies should run?
You:    2

        [mkdir -p k8s/api]
        [oc get secret pull-secret -n api → not found]

        Your image is in a private registry. I need credentials for it.

        Q: Registry, username, and access token?
You:    private.registry.example.com / svc-deploy / abc123token

        [oc create secret docker-registry pull-secret ...]
        [writes manifests, includes imagePullSecrets in deployment]
        [oc apply x3, rollout status → success]

        api is running.
```

---

## Example 3 — Pull secret already exists

```
You:    Deploy private.registry.example.com/team/another-app:v1.

Claude: [oc get secret pull-secret -n another-app → exists]

        Pull secret is already in this namespace — reusing it. No need to ask
        for credentials again.

        [writes manifests, applies, rolls out]
```

---

## Example 4 — Pod won't start (ImagePullBackOff)

```
Claude: [oc rollout status → progressing]
        [after 60s, still ImagePullBackOff]
        [oc describe pod shop-xxx-yyy -n shop]

        The cluster can't pull your image. Most common cause: typo in the tag
        or the image isn't actually pushed.

        Can you double-check that `docker pull ghcr.io/me/shop:v1` works from
        your machine? If yes but the cluster still can't pull, the registry is
        probably private and we need to add a pull secret.
```

---

## Example 5 — Pod crashes on startup

```
Claude: [pod status: CrashLoopBackOff]
        [oc logs shop-xxx-yyy -n shop]

        Your app is crashing on startup. Here's what it said:

          Error: Cannot find module 'express'
              at Function.Module._resolveFilename ...

        Looks like a missing dependency. Is `express` in your package.json,
        and did your Dockerfile run `npm ci` / `npm install`?
```

---

## What this skill won't do

- Add ConfigMaps, Secrets (other than the optional pull secret), HPAs,
  PodDisruptionBudgets, or NetworkPolicies unless you ask
- Set resource requests/limits (cluster has reasonable defaults)
- Generate Helm or Kustomize files — plain YAML only
- Expose the app on a URL — that's `expose-app`'s job

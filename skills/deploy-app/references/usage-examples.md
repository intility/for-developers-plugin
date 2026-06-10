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

## Example 2 — Deploy with a private registry (ghcr)

```
You:    Deploy ghcr.io/me/api:v2, port 8000. The image is private.

Claude: [oc whoami → ok]
        [indev pullsecret list → empty]

        Your image is private, so the cluster needs pull credentials — I set
        this up once and it covers every app on the cluster from now on.

        Q: Registry, username, and access token?
        (For ghcr.io: your GitHub username + a classic personal access token
        with the read:packages scope — github.com/settings/tokens)
You:    ghcr.io / me / ghp_abc123

        [indev pullsecret create --name ghcr --registry ghcr.io:me:ghp_abc123]
        [indev cluster pullsecret set shop-a3k9x2 ghcr]

        Done — the cluster can now pull from ghcr.io.

        What do you want to call this app?
You:    api

        [writes manifests — no imagePullSecrets needed]
        [oc apply x3, rollout status → success]

        api is running.
```

---

## Example 3 — Pull secret already covers the registry

```
You:    Deploy ghcr.io/me/another-app:v1. Also private.

Claude: [indev pullsecret list → ghcr (ghcr.io)]
        [indev pullsecret get ghcr → assigned to shop-a3k9x2]

        The cluster already has pull access to ghcr.io — no credentials
        needed.

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
        probably private — I'll set up cluster-wide pull access with
        `indev pullsecret` (one question, once, covers all future apps).
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

- Add ConfigMaps, Secrets, HPAs, PodDisruptionBudgets, or NetworkPolicies
  unless you ask (pull credentials live on the platform via `indev pullsecret`,
  not in your manifests)
- Set resource requests/limits (cluster has reasonable defaults)
- Generate Helm or Kustomize files — plain YAML only
- Expose the app on a URL — that's `expose-app`'s job

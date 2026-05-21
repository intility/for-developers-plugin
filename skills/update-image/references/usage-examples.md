# Usage Examples — Update Image

## Example 1 — Simple tag bump

```
You:    Update shop to v2.

Claude: [oc whoami → ok]
        [oc get deployments -A → shop, api]

        Which app do you want to update?
        [ ] shop (in shop)
        [ ] api (in api)
You:    shop

Claude: [oc get deployment shop -n shop -o jsonpath='{.spec.template.spec.containers[0].image}']
        → ghcr.io/me/shop:v1

        New image will be: ghcr.io/me/shop:v2 — confirm?
You:    yes

Claude: [oc set image deployment/shop shop=ghcr.io/me/shop:v2 -n shop]
        [oc rollout status deployment/shop -n shop --timeout=180s → success]
        [reads k8s/shop/deployment.yaml, edits image: line → v2]

        shop is now running v2. Local manifest is synced.
```

---

## Example 2 — Full image reference (different registry)

```
You:    Switch api over to quay.io/team/api:1.4.0.

Claude: [picks 'api' from list]
        [oc set image deployment/api api=quay.io/team/api:1.4.0 -n api]
        [rollout status → success]

        api is now running quay.io/team/api:1.4.0.

        Local k8s/api/deployment.yaml updated to match.
```

---

## Example 3 — New version is broken, roll back

```
Claude: [oc set image, oc rollout status → progressing]
        [60s later, pods CrashLoopBackOff]
        [oc logs new-pod → "TypeError: Cannot read property 'x' of undefined"]

        v2 is crashing on startup:
          TypeError: Cannot read property 'x' of undefined
              at startup ...

        That looks like a real bug, not an infra issue. Want me to roll back
        to v1 while you fix it?
You:    yes please

Claude: [oc rollout undo deployment/shop -n shop]
        [oc rollout status → success]

        Rolled back. shop is running v1 again.

        When you've pushed a fixed image (v2.1?), run "update shop to v2.1"
        and we'll try again.
```

---

## Example 4 — Container name doesn't match

Happens for hand-written deployments where the container isn't named after the app.

```
Claude: [oc set image deployment/shop shop=... -n shop]
        → error: container "shop" not found

        Let me check the actual container name.
        [oc get deployment shop -n shop -o jsonpath='{.spec.template.spec.containers[*].name}']
        → web

        Retrying with the right container name.
        [oc set image deployment/shop web=ghcr.io/me/shop:v2 -n shop]
        [rollout status → success]
```

---

## Example 5 — Local manifest doesn't exist

User is in a different repo than where the original manifests live.

```
Claude: [oc set image succeeds, rollout completes]
        [ls k8s/shop/deployment.yaml → not found in this dir]

        Image updated on the cluster ✓.

        Heads up: I don't see a local deployment.yaml here for shop. If you
        have the manifests in another repo, edit `image:` in that file to
        `ghcr.io/me/shop:v2` so it doesn't drift. Otherwise next time someone
        re-applies that file, it'll roll back to v1.
```

---

## Common phrasings that trigger update-image

- "update shop to v2"
- "deploy a new version of api"
- "bump shop to ghcr.io/me/shop:v3"
- "ship v2"
- "I just pushed a new image, update it"

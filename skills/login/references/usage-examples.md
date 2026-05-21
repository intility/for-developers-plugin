# Usage Examples — Login

## Example 1 — One cluster, fresh login

```
You:    Log me in.

Claude: [indev cluster list → shop-a3k9x2]
        [indev cluster login shop-a3k9x2]

        A browser will open. Sign in with your Intility account, then come back here.
You:    [completes browser SSO]

Claude: [oc whoami → you@intility.no]
        [oc get nodes → 2 nodes Ready]

        Logged in as you@intility.no. Your cluster has 2 nodes ready.
```

---

## Example 2 — Multiple clusters

```
You:    Connect me to my cluster.

Claude: [indev cluster list → shop-a3k9x2, api-b7m4p1]

        Which cluster do you want to log in to?
        [ ] shop-a3k9x2
        [ ] api-b7m4p1

You:    shop-a3k9x2
Claude: [indev cluster login shop-a3k9x2]
        … OAuth flow …
        Logged in.
```

---

## Example 3 — Browser doesn't open

```
Claude: [indev cluster login shop-a3k9x2]

        A browser should have opened. Did it?
You:    No.

Claude: [indev cluster login shop-a3k9x2 --no-browser]

        Copy this URL into your browser manually:
        https://oauth.shop-a3k9x2.example.com/auth?...

        Sign in, then come back here.
```

---

## Example 4 — Logged in but no permissions

```
Claude: [oc whoami → ok]
        [oc get nodes → Forbidden]

        You're logged in, but the cluster says you don't have permission to
        view nodes. Usually means your account is provisioned but no roles
        attached yet.

        Check with whoever owns the cluster, or reach out to the Developer
        Platform Admins via your collaboration channel (samhandlingskanal).
```

---

## Common phrasings that trigger login

- "log me in"
- "log into my cluster"
- "connect to the cluster"
- "oc login"
- "my token expired"

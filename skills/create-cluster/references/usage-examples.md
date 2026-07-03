# Usage Examples — Create Cluster

## Example 1 — First cluster

```
You:    Create a cluster.

Claude: [indev cluster list → empty]
        [indev account show → ok]

        What do you want to call your cluster?
You:    shop

Claude: [indev cluster create --name shop --preset minimal --nodes 2]
        Creating your cluster (shop-a3k9x2) — this takes 5–10 minutes.

        [polls indev cluster status every 30s]
        … 6 minutes later …

        Cluster 'shop-a3k9x2' is ready. Logging you in now.
        → invoking cluster-login
```

---

## Example 2 — You already have a cluster

The plugin assumes one cluster, many apps. If you already have one, it'll tell you.

```
You:    Set up a cluster.

Claude: [indev cluster list → shop-a3k9x2]

        You already have a cluster (shop-a3k9x2). Use that one?
        [ ] Yes, use it
        [ ] No, I really need another

You:    Yes
Claude: → invoking cluster-login
```

---

## Example 3 — Not logged in to indev

```
You:    Create a cluster called acme.

Claude: [indev account show → error]

        I need to log you in to indev first.
        [indev login]
        A browser will open. Sign in with your Intility account.

You:    [signs in]
Claude: [indev account show → ok]
        Continuing — creating cluster 'acme'…
```

---

## Example 4 — Cluster stuck

```
Claude: [polling 14 minutes in, status still "In Deployment"]

        Your cluster has been provisioning for 15 minutes — that's longer than
        normal. Time to escalate.

        Reach out to the Developer Platform Admins via your collaboration
        channel (samhandlingskanal). Tell them:
          - Cluster name: shop-a3k9x2
          - Status: stuck in "In Deployment" for 15+ minutes

        I'll stop polling now.
```

---

## What this skill won't do

- Create more than one cluster unless you confirm twice
- Ask about presets, autoscaling, zones, or node counts (defaults are fine for getting started)
- Set up team membership or RBAC (use the portal at developers.intility.com)

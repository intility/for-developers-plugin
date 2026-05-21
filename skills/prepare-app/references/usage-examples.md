# Usage Examples — Prepare App

## Example 1 — Everything ready

```
You:    My app is at ghcr.io/me/shop:v1, listens on 3000.

Claude: [find . -name Dockerfile → ./Dockerfile found]
        [docker manifest inspect ghcr.io/me/shop:v1 → ok, public]

        Image looks good. Port 3000. Handing off to deploy-app.
```

---

## Example 2 — No Dockerfile yet

```
You:    I want to deploy my Node app but I don't have a Dockerfile.

Claude: What does your app do, and how do you start it locally?
You:    It's a Next.js app. `npm start` runs it on port 3000.

Claude: [reads templates from references/minimal-dockerfile.md]
        [writes ./Dockerfile + .dockerignore]

        Here's a starter Dockerfile that should work. Run:
          docker build -t ghcr.io/<you>/<app>:v1 .
          docker push ghcr.io/<you>/<app>:v1

        Then come back and we'll deploy it.
```

The skill stops here. Building and pushing is the user's job.

---

## Example 3 — Private registry

```
You:    Image is at private.registry.example.com/team/app:v1, port 8080.

Claude: [docker manifest inspect → unauthorized]

        Looks like that image is in a private registry. We'll need to set up an
        image pull secret in the cluster — deploy-app will handle that, just be
        ready with your registry username + access token.

        Continuing.
```

---

## Example 4 — Image isn't pushed yet

```
You:    I have a Dockerfile but I haven't pushed anywhere.

Claude: I need an image reference (something like ghcr.io/<you>/<app>:v1) before
        we can deploy. The cluster pulls from a registry, not from your laptop.

        Pick a registry — GitHub's ghcr.io is free for public repos and works
        with your GitHub login. Then:
          docker build -t ghcr.io/<you>/<app>:v1 .
          docker push ghcr.io/<you>/<app>:v1

        Once it's pushed, come back and tell me the image reference.
```

---

## What this skill won't do

- Run `docker build` or `docker push` for you
- Write a Dockerfile without you telling it what the app is
- Set up a registry account

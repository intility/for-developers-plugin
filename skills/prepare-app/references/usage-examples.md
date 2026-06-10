# Usage Examples — Prepare App

## Example 1 — Everything ready

```
You:    My app is at ghcr.io/me/shop:v1, listens on 3000.

Claude: [Glob Dockerfile → ./Dockerfile found]
        [docker manifest inspect ghcr.io/me/shop:v1 → ok, public]

        Image looks good. Port 3000. Handing off to deploy-app.
```

---

## Example 2 — Never deployed anything before

User has code, no Dockerfile, no Docker, no GitHub account. The skill walks the whole path — one command at a time, the user runs each one.

```
You:    I built a little internal tool in Node. I've never deployed anything.
        Can you get it online?

Claude: Absolutely — we'll do it step by step. First I'll package your app so
        the platform can run it. How do you start it locally?
You:    npm start, it says it's on localhost:3000.

Claude: [writes ./Dockerfile from references/minimal-dockerfile.md, port 3000]

        Done — this file describes how to run your app anywhere. Next we need
        somewhere to upload it. Do you have a GitHub account?
You:    No.

Claude: No problem — it's free and you won't need to learn git; the account is
        just the key to GitHub's image storage. Create one at
        github.com/signup, then tell me your username.
You:    ok, I'm "annab"

Claude: Great. Now a token so Docker can log in as you:
        github.com/settings/tokens → "Generate new token (classic)" →
        tick "write:packages" → Generate. Copy the ghp_... string.
You:    got it

Claude: Run this, and paste the token when it asks for a password:

          docker login ghcr.io -u annab

You:    Login Succeeded

Claude: Now build:

          docker build -t ghcr.io/annab/mytool:v1 .

You:    done, lots of output but it finished

Claude: Last one — upload it:

          docker push ghcr.io/annab/mytool:v1

You:    done!

Claude: Your app now lives at ghcr.io/annab/mytool:v1. It's private by
        default, which is fine — I'll set up cluster access when we deploy.
        Handing off to deploy-app.
```

---

## Example 3 — Private registry

```
You:    Image is at ghcr.io/team/app:v1, port 8080. It's private.

Claude: [docker manifest inspect → unauthorized]

        Noted — deploy-app will set up cluster pull access (one question:
        username + token, once for the whole cluster). Continuing.
```

---

## Example 4 — Image isn't pushed yet

```
You:    I have a Dockerfile but I haven't pushed anywhere.

Claude: The cluster pulls your app from a registry, not from your laptop —
        so let's get it uploaded. It's three commands and I'll walk you
        through each. Do you have a GitHub account?
You:    yes

Claude: [continues with the token → login → build → push walkthrough from
         references/build-and-push.md, one command at a time]
```

---

## Example 5 — Port detected from the code

```
You:    Deploy this, image is pushed at ghcr.io/me/api:v1.

Claude: [Glob Dockerfile → found; reads it → EXPOSE 8000]

        Looks like your app listens on port 8000 — right?
You:    yes

Claude: Handing off to deploy-app.
```

---

## What this skill won't do

- Run `docker build` or `docker push` itself — you run them, it guides you through each one
- Write a Dockerfile without you telling it what the app is
- Create accounts for you (but it will point you at exactly the right signup and token pages)

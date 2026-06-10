---
name: prepare-app
description: Verifies that the user's app is containerized and that an image exists in a registry the cluster can pull from, before deploying. Use when the user wants to deploy something but you don't yet know if they have a Dockerfile or a pushed image, or when the deploy-app skill needs an image reference and doesn't have one yet.
allowed-tools:
  - AskUserQuestion
  - Bash(docker images*)
  - Bash(docker manifest inspect*)
  - Glob
  - Read
---

# Prepare App

## Goal

End this skill knowing:
1. The app is containerized (has a `Dockerfile` or `Containerfile`), **and**
2. There is an image reference like `ghcr.io/user/app:v1` that the cluster can pull.

If both are true, hand off to `deploy-app`. Otherwise, tell the user the smallest next step they need to take.

## Step 1 — Look for a Dockerfile

Search the project root and one level down using the Glob tool:

```
Glob: {Dockerfile,Containerfile,*/Dockerfile,*/Containerfile}
```

If found, note the path. If not:

```
Q: "I couldn't find a Dockerfile. Do you have one?"
  Options:
    - "Yes, somewhere else" — ask where
    - "No, not yet"
    - "I have an image already, no Dockerfile needed"
```

If "No, not yet": don't try to write one for them on the spot — ask what the app is (language, framework, how it starts), then draft a minimal `Dockerfile` using one of the templates in [references/minimal-dockerfile.md](references/minimal-dockerfile.md). Pick the closest match, adapt names and ports to their app, and drop in the file.

Then walk them through building and pushing it using [references/build-and-push.md](references/build-and-push.md) — one command at a time, they run each one themselves (this plugin doesn't run `docker build`). Don't assume they've used Docker, GitHub, or a registry before; the walkthrough covers the from-zero path including getting a token.

After they confirm the image is pushed, continue with Step 2.

## Step 2 — Confirm there's a pushed image

This plugin doesn't run `docker build`/`docker push` for the user — but it does guide them through running those themselves (see Step 1 / the build-and-push walkthrough). What this step needs to end with is an image reference.

```
Q: "What's the image reference for your app?"
  Options:
    - Other (free text — e.g. 'ghcr.io/myuser/myapp:v1', 'quay.io/team/api:latest')
```

If they don't know what an image reference is, give them one sentence:

> It's the address where your built image lives. After `docker push`, it's the same string you `docker push`-ed to — registry + repo + tag.

If they haven't pushed yet, don't just send them away — offer to walk them through it right now using [references/build-and-push.md](references/build-and-push.md). It covers the full from-zero path: Docker Desktop, a free GitHub account if they don't have one (no git knowledge needed — the account is just the key to the registry), token, login, build, push. One command at a time; they run each themselves. If their team already has a different registry, the same steps apply with that registry's address and login.

## Step 3 — Optional: sanity check the image exists

If they have `docker` locally, you can confirm the image is actually reachable:

```bash
docker manifest inspect <image-ref>
```

A successful response means the cluster can probably pull it too. If it fails with `unauthorized`, the image is private (or the name has a typo — check that first) — flag it and continue; `deploy-app` sets up pull credentials once for the whole cluster with `indev pullsecret`, so this costs the user one extra question, once.

Skip this step if `docker` isn't installed locally — it's a nice-to-have, not a blocker.

## Step 4 — What port does the app listen on?

This is the single most-forgotten thing later — and "what port?" is a question many users genuinely can't answer. So **look before you ask**: check the `Dockerfile` for `EXPOSE`, and the code for the usual suspects (`package.json` start script, `app.listen(...)`, `PORT` env defaults, framework config).

- **Found one** → confirm it in one line: "Looks like your app listens on port 3000 — right?" Most users just say yes.
- **Can't tell** → ask, with hints:

```
Q: "What port does your app listen on inside the container?"
  Options:
    - "3000" — common for Node.js / Next.js
    - "8000" — common for Python / Django / FastAPI
    - "8080" — common for Java / Go
    - Other (free text)
```

If they have no idea even with the hints, offer to look through the code together — it's in there somewhere.

## Step 5 — Hand off

Pass forward to `deploy-app`:

```yaml
app:
  image: <image-ref>
  port: <container-port>
  private_registry: <true|false>   # from Step 3, if known
```

## Tone

- Don't lecture them on containers. If they don't know what an image is, give one sentence, then move on.
- Never assume they've used Docker, GitHub, or git before. Check with one short question, then meet them where they are — the build-and-push walkthrough exists exactly for this.
- If they're missing a Dockerfile and the conversation goes deep, *stop* and write a Dockerfile in a separate turn. Don't conflate "deploy" with "containerize from scratch" — these are different sessions.

## References

- [references/minimal-dockerfile.md](references/minimal-dockerfile.md) — copy-paste Dockerfile templates for Node, Python, Go, and static sites
- [references/build-and-push.md](references/build-and-push.md) — from-zero walkthrough: Docker Desktop, GitHub account, token, login, build, push
- [references/usage-examples.md](references/usage-examples.md) — typical conversations (ready to go, no Dockerfile, private registry, image not pushed)

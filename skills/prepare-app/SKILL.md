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

If "No, not yet": don't try to write one for them on the spot — ask what the app is (language, framework, how it starts), then draft a minimal `Dockerfile` using one of the templates in [references/minimal-dockerfile.md](references/minimal-dockerfile.md). Pick the closest match, adapt names and ports to their app, drop in the file, and stop. They build and push it themselves — this plugin doesn't run `docker build`.

After they confirm the image is pushed, continue with Step 2.

## Step 2 — Confirm there's a pushed image

This plugin doesn't build or push images for the user — that varies too much by registry. We just need an image reference.

```
Q: "What's the image reference for your app?"
  Options:
    - Other (free text — e.g. 'ghcr.io/myuser/myapp:v1', 'quay.io/team/api:latest')
```

If they don't know what an image reference is, give them one sentence:

> It's the address where your built image lives. After `docker push`, it's the same string you `docker push`-ed to — registry + repo + tag.

If they haven't pushed yet, point them at the registry they're using (or suggest `ghcr.io` if they're on GitHub) and **stop here**. Tell them: "Once you've run `docker push`, come back and we'll deploy it." Do not try to build or push for them.

## Step 3 — Optional: sanity check the image exists

If they have `docker` locally, you can confirm the image is actually reachable:

```bash
docker manifest inspect <image-ref>
```

A successful response means the cluster can probably pull it too. If it fails with `unauthorized`, the image is private — flag this and tell the user the cluster will also need credentials (an `imagePullSecret`). Note it and continue; `deploy-app` will handle the secret.

Skip this step if `docker` isn't installed locally — it's a nice-to-have, not a blocker.

## Step 4 — What port does the app listen on?

This is the single most-forgotten thing later. Ask now:

```
Q: "What port does your app listen on inside the container?"
  Options:
    - "3000" — common for Node.js / Next.js
    - "8000" — common for Python / Django / FastAPI
    - "8080" — common for Java / Go
    - Other (free text)
```

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
- If they're missing a Dockerfile and the conversation goes deep, *stop* and write a Dockerfile in a separate turn. Don't conflate "deploy" with "containerize from scratch" — these are different sessions.

## References

- [references/minimal-dockerfile.md](references/minimal-dockerfile.md) — copy-paste Dockerfile templates for Node, Python, Go, and static sites
- [references/usage-examples.md](references/usage-examples.md) — typical conversations (ready to go, no Dockerfile, private registry, image not pushed)

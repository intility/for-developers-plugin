# Build and Push Your First Image

A walkthrough for users who have never built or pushed a container image. Guide them through it **one command at a time** — show a command, let them run it (or run it for them only if they explicitly ask), confirm it worked, then show the next. Never paste the whole list at once.

The flow below uses GitHub's registry (`ghcr.io`) because it's free and most common. If their team already has another registry, the same steps apply — only the address and login change.

## Before starting, check quietly

- **Docker installed?** `command -v docker`. If missing: they need Docker Desktop — https://www.docker.com/products/docker-desktop/ — install, open it once, come back. (On macOS: `brew install --cask docker` also works.)
- **GitHub account?** Ask in one line. If they don't have one, it's free at https://github.com/signup — they don't need to know git or use repositories; the account is just the key to the registry.

## Step 1 — Get a token to log in with

ghcr.io doesn't accept your GitHub password. You need a **classic personal access token**:

1. Go to https://github.com/settings/tokens → "Generate new token (classic)"
2. Name it something like `registry`, tick the **`write:packages`** scope (this auto-ticks `read:packages`)
3. Generate, and copy the token (starts with `ghp_`) — you won't see it again

Tell the user to keep it somewhere safe; it's also what `deploy-app` will ask for later so the cluster can pull the image.

## Step 2 — Log Docker in to the registry

```bash
docker login ghcr.io -u <github-username>
```

When it asks for a password, paste the token from Step 1 — not the GitHub password. Success looks like `Login Succeeded`.

## Step 3 — Build the image

Run from the directory with the `Dockerfile`:

```bash
docker build -t ghcr.io/<github-username>/<app-name>:v1 .
```

One-line explanation to give: *"`-t` names the image — the name doubles as the address it will be uploaded to. `v1` is the version tag; you'll bump it for each release."*

Notes:
- The name must be **all lowercase** — `ghcr.io/Me/MyApp` is rejected.
- Don't forget the trailing `.` — it means "build from this directory".
- On Apple Silicon Macs, build for the cluster's architecture: add `--platform linux/amd64`.

## Step 4 — Push it

```bash
docker push ghcr.io/<github-username>/<app-name>:v1
```

When the progress bars finish, the image is in the registry. That full string — `ghcr.io/<github-username>/<app-name>:v1` — is the **image reference** the rest of the journey needs.

## Good to know afterwards

- **ghcr images are private by default.** That's fine — `deploy-app` sets up cluster pull access with one question (it'll ask for the same username + token).
- **Shipping a new version later:** rebuild and push with a new tag (`:v2`), then say "update <app> to v2" — the `update-image` skill does the rest.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `denied` on push | Token missing the `write:packages` scope — make a new one |
| `unauthorized` on login | Pasted the GitHub password instead of the token |
| `invalid reference format` / repository name errors | Uppercase letters in the image name — lowercase everything |
| Build fails | That's a Dockerfile/app problem, not a registry problem — read the error together and fix the Dockerfile |

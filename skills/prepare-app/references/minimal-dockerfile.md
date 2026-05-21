# Minimal Dockerfile Templates

Use these as a starting point when the user doesn't have a `Dockerfile` yet. Pick the closest match, edit names/ports, and let the user build & push it themselves — this plugin doesn't run `docker build`.

Every template uses a small, security-patched base image, runs as a non-root user where the base allows it, and exposes one HTTP port.

---

## Node.js (npm)

For Express, Next.js, NestJS, Vite preview servers, etc.

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

If you're running a `next start` server, swap the last line for `CMD ["npm", "start"]` and make sure `npm run build` is part of your build pipeline before this image is built.

---

## Python (pip + uvicorn / gunicorn)

For FastAPI, Flask, Django served by uvicorn or gunicorn.

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Replace `main:app` with your actual module:variable path.

---

## Go (multi-stage)

Tiny final image, no toolchain shipped.

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
EXPOSE 8080
USER nonroot
ENTRYPOINT ["/app"]
```

---

## Static site (built HTML/CSS/JS via nginx)

For a built Vite, Next.js export, Astro, or any folder of static files.

```dockerfile
FROM nginx:1.27-alpine

COPY dist/ /usr/share/nginx/html/

EXPOSE 80
```

Make sure your build output lands in `dist/` before this image is built (e.g. `npm run build`).

---

## `.dockerignore`

Every Dockerfile should ship with this next to it — keeps the image small and avoids accidentally baking in secrets:

```
node_modules
.git
.env
.env.*
*.log
dist
build
__pycache__
.venv
.idea
.vscode
```

---

## After you have a Dockerfile

1. Build and tag your image:

   ```bash
   docker build -t <registry>/<user>/<app>:v1 .
   ```

2. Push it to a registry the cluster can reach (GHCR, Docker Hub, your own registry):

   ```bash
   docker push <registry>/<user>/<app>:v1
   ```

3. Come back to Claude with the image reference (e.g. `ghcr.io/me/shop:v1`) and the plugin will pick up from there.

If your registry is private, the `deploy-app` skill will help you set up the pull secret.

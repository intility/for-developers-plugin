# Manifest Templates

Plain-YAML templates for the three resources you need to deploy a containerized app: `Namespace`, `Deployment`, `Service`. Use these as-is — substitute the placeholders, write to disk, apply.

## Placeholders

| Placeholder | Example |
|---|---|
| `<app-name>` | `myapp` |
| `<namespace>` | same as `<app-name>` — `myapp` |
| `<image>` | `ghcr.io/myuser/myapp:v1` |
| `<container-port>` | `3000` |
| `<replicas>` | `1` |

---

## namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    app: <app-name>
```

---

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <namespace>
  labels:
    app: <app-name>
spec:
  replicas: <replicas>
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
        - name: <app-name>
          image: <image>
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: <container-port>
              name: http
```

### Private images

Don't add `imagePullSecrets` here. Private registries are handled cluster-wide by the platform — `deploy-app` sets it up once with `indev pullsecret create` + `indev cluster pullsecret set`, and every namespace can pull from then on. The manifests stay identical whether the image is public or private.

### With environment variables (only if the user asks)

Add under the container:

```yaml
          env:
            - name: <KEY>
              value: "<value>"
```

For secrets, use `valueFrom: secretKeyRef:` instead — but don't pre-emptively wire that up. Wait for the user to ask.

---

## service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
  namespace: <namespace>
  labels:
    app: <app-name>
spec:
  selector:
    app: <app-name>
  ports:
    - name: http
      port: 80
      targetPort: <container-port>
  type: ClusterIP
```

Notes:
- `port: 80` is what the rest of the cluster (and the HTTPRoute) talks to. `targetPort` is what your container actually listens on. Keeping `port` at `80` makes the HTTPRoute simpler.
- `type: ClusterIP` is right — don't use `NodePort` or `LoadBalancer`. The HTTPRoute (set up by `expose-app`) is how outside traffic gets in.

---

## Full minimal example

App: a Next.js app called `myshop`, image `ghcr.io/me/myshop:v1`, container port 3000, 1 replica.

**namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myshop
  labels:
    app: myshop
```

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myshop
  namespace: myshop
  labels:
    app: myshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myshop
  template:
    metadata:
      labels:
        app: myshop
    spec:
      containers:
        - name: myshop
          image: ghcr.io/me/myshop:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myshop
  namespace: myshop
  labels:
    app: myshop
spec:
  selector:
    app: myshop
  ports:
    - name: http
      port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply:

```bash
oc apply -f k8s/myshop/namespace.yaml
oc apply -f k8s/myshop/deployment.yaml
oc apply -f k8s/myshop/service.yaml
```

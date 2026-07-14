---
name: otel-operator
description: Sets up Collector and/or PodLogCollector for the pre-installed otel-operator on the Intility Developer Platform. Use when the user asks to "set up otel collector", "install otel collector", "configure otel operator", "collect pod logs", or "set up OpenTelemetry".
allowed-tools:
  - AskUserQuestion
  - Bash(oc whoami*)
  - Bash(oc get*)
  - Bash(oc apply*)
  - Bash(oc describe*)
  - Bash(oc rollout*)
  - Bash(oc logs*)
  - Bash(oc patch*)
  - Bash(mkdir *)
  - Read
  - Write
  - Edit
---

# Otel Operator

## Goal

Set up one or both of:
- **Collector** — receives telemetry (traces and metrics) that your apps send it over the network, and forwards it to wherever you store and view that data.
- **PodLogCollector** — reads the log output of pods on the cluster and ships it to the same kind of destination.

When "Both" is selected, complete Collector fully (through to `Running` and operator `Ready`) before starting PodLogCollector. Never interleave the two.

End state: resource(s) are `Running` and the operator has marked them `Ready`.

---

## Explaining things to the user

Assume the user is new to Kubernetes and OpenTelemetry. The first time a term comes up, define it in one plain sentence, then move on. Keep your questions jargon-free, and never paste a wall of troubleshooting steps — diagnose it yourself and tell them the one thing that matters.

Plain-language definitions to reuse:

- **OpenTelemetry (OTel)** — an open standard for collecting telemetry from your apps.
- **Telemetry** — data about how your app is behaving. It comes in three kinds:
  - **Traces** — the path a single request takes through your app, and how long each step took.
  - **Metrics** — numbers measured over time, like CPU usage or requests per second.
  - **Logs** — the text lines your app prints as it runs.
- **Collector** — a small service that receives telemetry from your apps and forwards it on.
- **OTLP** — the standard way apps send telemetry to a collector (gRPC on port 4317, or HTTP on 4318).
- **Backend** — where telemetry ends up so you can search and chart it (e.g. Tempo for traces, Loki for logs, Prometheus for metrics). This skill assumes you already have one.
- **PodLogCollector** — runs on every node in the cluster and collects logs, but only from the pods you opt in.

---

## Step 0 — Confirm access

```bash
oc whoami
```

If it errors, run the `cluster-login` skill first. Don't proceed without auth.

## If `oc` returns "Unauthorized" mid-flow

The `oc` token has expired. Stop where you are, route to the `cluster-login` skill, then resume from the failed step.

---

## Step 0.5 — Check available CRDs

Before asking the user anything, check which CRDs the operator has installed:

```bash
oc get crd collectors.otel.intility.io 2>/dev/null
oc get crd podlogcollectors.otel.intility.io 2>/dev/null
```

Determine availability from the results:

| Collector CRD | PodLogCollector CRD | Action |
|---|---|---|
| Present | Present | Ask Step 1 setup question with all three options (Collector / PodLogCollector / Both) |
| Present | Missing | Skip the Step 1 setup question entirely — proceed directly as if "Collector" was selected |
| Missing | Present | Ask Step 1 setup question with only the PodLogCollector option |
| Missing | Missing | Stop and tell the user: "The otel-operator does not appear to be installed on this cluster — neither the Collector nor PodLogCollector CRD was found." |

---

## Step 1 — Shared setup questions

**Skip this step if only one CRD is available** (selection is already determined — see Step 0.5).

Call `AskUserQuestion` with these three questions in one shot:

- header: "Setup"
  question: "What do you want to set up?"
  options:
    - label: "Collector"
      description: "Receives traces & metrics from your apps and forwards them to a backend"
    - label: "PodLogCollector"
      description: "Collects log output from pods you opt in, runs as a DaemonSet on every node"
    - label: "Both"
      description: "Set up a Collector and a PodLogCollector"

  Only include options for CRDs that are available (from Step 0.5).

- header: "Namespace"
  question: "What namespace should the collector(s) run in?"
  options:
    - label: "otel-operator-system"
      description: "Recommended default — where the operator itself lives"
    - label: "Other"
      description: "I'll type a different namespace"

- header: "Names"
  question: "What names do you want to use?"
  options:
    - label: "Use defaults"
      description: "platform-collector / platform-podlogcollector"
    - label: "Custom"
      description: "I'll provide my own name(s)"

If the user picks "Custom" for names, follow up with a free-text question for the specific name(s) relevant to what they selected in Q1.

---

## Step 2 — Collector

**Skip this step entirely if only PodLogCollector was selected.**

### 2a — Configure the export backend

Call `AskUserQuestion`:

- header: "Backend"
  question: "Is the backend that will receive traces and metrics running on the cluster, or is it external?"
  options:
    - label: "On the cluster"
      description: "I'll help you find the right service"
    - label: "External"
      description: "Hosted outside the cluster — I'll enter the hostname/IP and port"

**If on-cluster:**

1. Ask: "What namespace is the backend running in? (e.g. `monitoring`, `observability`)"
2. Run:
   ```bash
   oc get svc -n <backend-namespace>
   ```
3. Show the user the service list and ask which service to export to.
4. If the chosen service exposes multiple ports, use this table to pick the right one — do not ask the user to choose blindly:

   | Port | Protocol | Use |
   |------|----------|-----|
   | 4317 | gRPC | OTLP gRPC — use `otlp` exporter |
   | 4318 | HTTP | OTLP HTTP — use `otlphttp` exporter |
   | 3100 | HTTP | Loki HTTP API — use `otlphttp` exporter |
   | 9095 | gRPC | Loki internal clustering — **not for OTLP, skip this** |

   If only one suitable port exists, pick it without asking. If genuinely ambiguous, ask the user.

5. Construct the endpoint based on the port chosen:
   - gRPC (`otlp`): `<service>.<namespace>.svc.cluster.local:<port>`
   - HTTP (`otlphttp`): `http://<service>.<namespace>.svc.cluster.local:<port>`

   Note the exporter type (`otlp` or `otlphttp`) — it drives the manifest template in 2b. No auth question for on-cluster endpoints.

**If external:**

Ask: "What is the OTLP endpoint for traces and metrics? (e.g. `tempo.example.com:4317`)"

Then call `AskUserQuestion`:

- header: "Auth"
  question: "Does this endpoint require authentication?"
  options:
    - label: "Yes"
      description: "I have a Kubernetes Secret with a write token or API key"
    - label: "No"
      description: "The endpoint is unauthenticated"

**If auth is required:**

Ask: "What is the name of the Kubernetes Secret that holds the credentials? (must exist in the collector namespace)"

Ask: "What key inside that secret contains the token? (e.g. `token`, `write-token`, `api-key`)"

Verify the secret exists — but do not read its contents:

```bash
oc get secret <secret-name> -n <namespace>
```

If not found, tell the user and ask them to create the secret first. Do not proceed without a confirmed secret.

Note the secret name, key, and use mount path `/secrets/backend` — these feed into the manifest in 2b.

### 2b — Generate manifest

Use the Collector template in [references/manifest-templates.md](references/manifest-templates.md). If auth was requested, use the secret-mount variant. Write to:

```
otel/<collector-name>/collector.yaml
```

If the directory doesn't exist, create it. If the file already exists, ask before overwriting.

### 2c — Apply manifest

```bash
oc apply -f otel/<collector-name>/collector.yaml
```

### 2d — Watch rollout

```bash
oc rollout status deployment/<collector-name> -n <namespace> --timeout=120s
```

### 2e — Verify operator status

```bash
oc get col <collector-name> -n <namespace>
```

The `Ready` column should show `True`. If it shows `False` or `Unknown`, describe the resource to read the conditions:

```bash
oc describe col <collector-name> -n <namespace>
```

A `ConfigurationError` condition means the OTel config YAML is invalid — the message will say what's wrong.

### 2f — If pods aren't Running

Don't dump troubleshooting trees on the user. Look at the pod yourself, then summarise in one sentence what's wrong.

```bash
oc get pods -n <namespace>
oc describe pod <pod-name> -n <namespace>
oc logs <pod-name> -n <namespace>
```

| What you see | Tell the user |
|---|---|
| `CrashLoopBackOff` + OTel config parse error in logs | "The collector config is invalid — here's the error: …" |
| `CrashLoopBackOff` + connection refused on export | "The collector can't reach the export endpoint. Verify the address is correct and reachable from the cluster." |
| Pending forever | "The cluster is out of capacity. Check `oc get nodes` and consider resizing." |

---

## Step 3 — PodLogCollector

**Skip this step entirely if only Collector was selected.**

**If Both was selected: only start this step after Step 2 is fully complete.**

### 3a — Configure the log export backend

Call `AskUserQuestion`:

- header: "Log backend"
  question: "Where should pod logs be shipped?"
  options:
    - label: "Debug"
      description: "Print to stdout — good for verifying collection works before wiring a real backend"
    - label: "On the cluster"
      description: "I'll help you find the right service"
    - label: "External"
      description: "Hosted outside the cluster — I'll enter the hostname/IP and port"

**If debug:** no endpoint needed — use the debug exporter template. Continue to 3b.

**If on-cluster:**

1. Ask: "What namespace is the backend running in?"
2. Run:
   ```bash
   oc get svc -n <backend-namespace>
   ```
3. Show the user the service list and ask which service to ship logs to.
4. If the chosen service exposes multiple ports, use this table to pick the right one — do not ask the user to choose blindly:

   | Port | Protocol | Use |
   |------|----------|-----|
   | 4317 | gRPC | OTLP gRPC — use `otlp` exporter |
   | 4318 | HTTP | OTLP HTTP — use `otlphttp` exporter |
   | 3100 | HTTP | Loki HTTP API — use `otlphttp` exporter |
   | 9095 | gRPC | Loki internal clustering — **not for OTLP, skip this** |

   If only one suitable port exists, pick it without asking. If genuinely ambiguous, ask the user.

5. Construct the endpoint based on the port chosen:
   - gRPC (`otlp`): `<service>.<namespace>.svc.cluster.local:<port>`
   - HTTP (`otlphttp`): `http://<service>.<namespace>.svc.cluster.local:<port>`

   Note the exporter type (`otlp` or `otlphttp`) — it drives the manifest template in 3b. No auth question for on-cluster endpoints.

**If external:**

Ask: "What is the log export endpoint? (e.g. `loki.example.com:4317`)"

Then call `AskUserQuestion`:

- header: "Auth"
  question: "Does this endpoint require authentication?"
  options:
    - label: "Yes"
      description: "I have a Kubernetes Secret with a write token or API key"
    - label: "No"
      description: "The endpoint is unauthenticated"

**If auth is required:**

Ask: "What is the name of the Kubernetes Secret that holds the credentials? (must exist in the collector namespace)"

Ask: "What key inside that secret contains the token? (e.g. `token`, `write-token`, `api-key`)"

Verify the secret exists — but do not read its contents:

```bash
oc get secret <secret-name> -n <namespace>
```

If not found, tell the user and ask them to create the secret first. Do not proceed without a confirmed secret.

Note the secret name, key, and use mount path `/secrets/backend` — these feed into the manifest in 3b.

### 3b — Generate manifest

Use the PodLogCollector template in [references/manifest-templates.md](references/manifest-templates.md). If auth was requested, use the secret-mount variant. Write to:

```
otel/<podlogcollector-name>/podlogcollector.yaml
```

If debug was selected, use the debug exporter variant from the templates. If the file already exists, ask before overwriting.

### 3c — Apply manifest

```bash
oc apply -f otel/<podlogcollector-name>/podlogcollector.yaml
```

### 3d — Watch rollout

PodLogCollector is a DaemonSet — use `ds/`, not `deployment/`:

```bash
oc rollout status ds/<podlogcollector-name> -n <namespace> --timeout=120s
```

### 3e — Verify operator status

```bash
oc get podlogcol <podlogcollector-name> -n <namespace>
```

The `Ready` column should show `True`. If it shows `False` or `Unknown`:

```bash
oc describe podlogcol <podlogcollector-name> -n <namespace>
```

A `ConfigurationError` condition means the OTel config YAML is invalid — the message will say what's wrong.

### 3f — If pods aren't Running

```bash
oc get pods -n <namespace>
oc describe pod <pod-name> -n <namespace>
oc logs <pod-name> -n <namespace>
```

| What you see | Tell the user |
|---|---|
| `CrashLoopBackOff` + OTel config parse error in logs | "The collector config is invalid — here's the error: …" |
| `CrashLoopBackOff` + connection refused on export | "The log collector can't reach the export endpoint. Verify the address is correct and reachable from the cluster." |
| Pending forever | "The cluster is out of capacity. Check `oc get nodes` and consider resizing." |
| `permission denied` on `/var/log/pods` | "The operator should auto-grant privileged SCC to the SA. Check: `oc get rolebinding -n <namespace> \| grep podlogcol`" |

### 3g — Label workloads

The PodLogCollector runs on every node and reads all log files under `/var/log/pods/`, but it only keeps logs from pods that carry the selector label. This opt-in design means you get exactly the logs you asked for and nothing else — no accidental collection from unrelated workloads sharing the same node.

Call `AskUserQuestion`:

- header: "Workloads"
  question: "Which workloads do you want to collect logs from? For each one I'll need the name and namespace."
  options:
    - label: "List them now"
      description: "e.g. deployment/my-app in namespace my-team"
    - label: "Skip for now"
      description: "I'll add the label to my workloads manually later"

If the user lists workloads, run `oc get <kind> <name> -n <namespace>` for each one to confirm it exists before patching. If one isn't found, tell the user and continue with the rest.

For each confirmed workload, apply the selector label (`collect-podlogs: "true"` by default):

**Deployment:**
```bash
oc patch deployment <name> -n <namespace> --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"collect-podlogs":"true"}}}}}'
```

**StatefulSet:**
```bash
oc patch statefulset <name> -n <namespace> --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"collect-podlogs":"true"}}}}}'
```

**DaemonSet:**
```bash
oc patch daemonset <name> -n <namespace> --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"collect-podlogs":"true"}}}}}'
```

**DeploymentConfig** (OpenShift-specific):
```bash
oc patch deploymentconfig <name> -n <namespace> --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"collect-podlogs":"true"}}}}}'
```

Verify logs are flowing after the pods restart:
```bash
oc logs -n <namespace> -l app.kubernetes.io/component=otel-podlog-collector --tail=20
```

---

## Step 4 — Hand off

Once everything is healthy, tell the user:

- **Collector**: "Your collector is running. Configure your app to send its telemetry to `<name>.<namespace>.svc.cluster.local:4317` (gRPC) or `:4318` (HTTP) — that's the address your app points its OpenTelemetry exporter at."
- **PodLogCollector**: "Your log collector is running. Add the label `collect-podlogs: true` to any pod (or its Deployment) to start collecting that pod's logs — that's how you opt a workload in."
- **Both**: Combine the above.

Also mention: "If you want to customise processor settings later (batch size, add attributes, filter logs) without editing this manifest, you can create a `CollectorProcessor` — ask me when you're ready."

---

## References

- [references/manifest-templates.md](references/manifest-templates.md) — YAML templates for Collector and PodLogCollector

---

## What not to do

- Don't set `allowProcessorOverride: false` unless the user explicitly asks — it blocks `CollectorProcessor` from working later.
- Don't use `oc rollout status deployment/` for PodLogCollector — it's a DaemonSet, use `ds/`.
- Don't pre-create the ServiceAccount for PodLogCollector — the operator creates it and grants it the required SCC automatically.
- Don't add resource requests/limits unless the user asks — the operator sets sensible defaults.
- Don't use `matchingLabels` in PodLogCollector manifests — the correct field is `podSelector.matchLabels`. The sample in the otel-operator repo uses the wrong field name.
- Don't interleave Collector and PodLogCollector setup when "Both" is selected — finish one before starting the other.
- Never read secret values — no `oc get secret -o yaml`, no `oc describe secret`, no `oc extract`. Only `oc get secret <name> -n <namespace>` to confirm existence.

# Wealth Demo App — Setup Runbook

Deploy the multi-service wealth-management app to your k3s cluster via GitHub +
Argo CD, then break it on demand for the Problems-app demo.

---

## Step 1 — Put the app in your repo

Add the whole **`wealth-demo/`** folder to `github.com/jib94/selfhealing-demo`
on the `main` branch. Do **not** put it in `manifests/` — that folder belongs to
the self-healing demo, and keeping them separate means the two demos can't
interfere with each other.

Your repo should end up looking like:

```
selfhealing-demo/
├── manifests/            <- Demo 3 (self-healing OOMKill)
│   └── oom-demo.yaml
└── wealth-demo/          <- Demo 2 (this app)
    ├── 00-namespace-and-config.yaml
    ├── 01-app-code.yaml
    ├── 02-data-services.yaml
    ├── 03-domain-services.yaml
    ├── 04-edge-services.yaml
    ├── 05-traffic-generator.yaml
    └── README.md
```

**Easiest upload (browser):** open the repo → **Add file ▸ Upload files** →
drag the entire `wealth-demo` folder in → **Commit changes**.

**Or with git:**

```
git clone https://github.com/jib94/selfhealing-demo.git
```
```
cd selfhealing-demo
```
(copy the `wealth-demo` folder in, then)
```
git add wealth-demo && git commit -m "Add wealth management demo app" && git push
```

---

## Step 2 — Point Argo CD at it

Save `argocd-wealth-demo-application.yaml` locally (e.g. `~/dynatrace-demo/`) and apply it:

```
kubectl apply -f argocd-wealth-demo-application.yaml
```

Then watch it come up:

```
kubectl -n wealth-demo get pods -w
```

All seven pods should reach `Running` / `1/1`. First start pulls `node:20-slim`,
so give it a minute.

---

## Step 3 — Confirm it's healthy and generating traffic

```
kubectl -n wealth-demo logs -l app=traffic-generator --tail=5
```

You want lines like `requests=180 ok=180 failed=0 failRate=0.0%`.

Quick manual check of the full call chain:

```
kubectl -n wealth-demo exec deploy/traffic-generator -- node -e "require('http').get({host:'web-portal',port:8080,path:'/dashboard/C-1001'},r=>r.pipe(process.stdout))"
```

You should see a JSON portfolio with a market value.

---

## Step 4 — Confirm Dynatrace is monitoring it

Your Dynatrace operator injects OneAgent into pods. Verify it reached this new
namespace:

```
kubectl -n wealth-demo get pod -l app=web-portal -o jsonpath='{.items[0].spec.initContainers[*].name}'; echo
```

If that prints `dynatrace-operator`, injection worked and you'll get full
service-level tracing. **If it prints nothing**, your DynaKube uses a namespace
selector — check it with:

```
kubectl get dynakube -A -o yaml | grep -A5 namespaceSelector
```

and add a matching label to the namespace, e.g.:

```
kubectl label namespace wealth-demo monitoring=dynatrace
```

then restart the pods:

```
kubectl -n wealth-demo rollout restart deploy
```

Give Dynatrace 5–10 minutes, then look for the services in the **Services** list
and check the **Service flow** for `web-portal` — you should see the full chain
down to `positions-store`.

---

## Step 5 — Let it bake (important)

Leave the app running **healthy for at least a few hours** before the demo — ideally
overnight. Dynatrace needs a clean baseline so that when you flip problem mode, Davis
raises a proper problem with a clear root cause rather than treating the errors as
normal.

---

## Demo day — breaking it

In GitHub, edit `wealth-demo/00-namespace-and-config.yaml`:

```yaml
  problem-mode: "on"      # was "off"
```

Commit to `main`. Argo CD syncs within ~1 minute, and Kubernetes refreshes the
mounted config within another 1–2 minutes. No pods restart.

**Do this 10–15 minutes before you present** so Dynatrace has time to detect the
degradation and open a problem.

Watch it happen:

```
kubectl -n wealth-demo logs -l app=traffic-generator --tail=5
```

You're looking for `failRate` climbing to roughly 30%.

```
kubectl -n wealth-demo logs -l app=positions-store --tail=10
```

You should see `connection pool exhausted` errors — that's the root cause the AI
will identify.

---

## Resetting after the demo

Set `problem-mode` back to `"off"` and commit. Recovery takes 1–2 minutes.

To remove the app entirely:

```
kubectl delete -f argocd-wealth-demo-application.yaml
```
```
kubectl delete namespace wealth-demo
```

---

## Troubleshooting

**Pods stuck `ContainerCreating`** — usually still pulling `node:20-slim`. Check with
`kubectl -n wealth-demo describe pod <name>`.

**Argo shows `Unknown` / `ComparisonError`** — the repo path is wrong or the repo is
private. Confirm `path: wealth-demo` matches the folder name exactly.

**No errors after flipping the toggle** — the mounted file can take up to ~2 minutes to
refresh. If it still looks healthy, confirm the ConfigMap actually changed:
`kubectl -n wealth-demo get configmap wealth-demo-config -o yaml`

**Too few / too many errors** — tune `db-pool-size` (lower = worse) and `db-query-ms`
(higher = worse) in the same ConfigMap, or raise `VUS` in the traffic generator
(that one needs `kubectl -n wealth-demo rollout restart deploy/traffic-generator`).

# Meridian Wealth Platform — Dynatrace demo app

A small, multi-service wealth-management application that runs on Kubernetes,
generates its own realistic traffic, and can be broken **on demand** from a single
YAML value — producing a failure that spreads across several services with one
genuinely interesting root cause.

No container build and no registry: every service runs on the stock `node:20-slim`
image, using only the Node standard library, with the source mounted from a ConfigMap.

---

## The application

```
              traffic-generator          (simulated clients + advisors)
                      |
          +-----------+-----------+
          |                       |
     web-portal              advisor-api
     (client portal)         (advisor workstation)
          |                    |        |
          |                    |        +--> client-profile-service --+
          |                    |                                      |
          +--> portfolio-service <-----------------------------------+|
                    |        |                                        |
                    |        +--> pricing-service   (stays healthy)   |
                    |                                                 |
                    +--> positions-store <----------------------------+
                              ^
                              |
                    THE SHARED DEPENDENCY — and the root cause
```

| Service | Role |
|---|---|
| `web-portal` | Client-facing portal: login, dashboard, holdings, retirement projection |
| `advisor-api` | Advisor workstation: book of business, client reviews |
| `portfolio-service` | Values a portfolio (holdings × prices) |
| `client-profile-service` | Advisor-facing client record |
| `pricing-service` | Market data and quotes — **deliberately stays healthy** |
| `positions-store` | System of record for holdings — **the shared dependency** |
| `traffic-generator` | Simulates client and advisor journeys continuously |

---

## Deploy

Argo CD syncs this folder. Once the Application exists, everything below is
driven by Git.

```bash
kubectl apply -f argocd-wealth-demo-application.yaml   # one time
kubectl -n wealth-demo get pods
```

All seven pods should reach `Running`, and the traffic generator starts driving
load immediately.

---

## Turning on the problem

Edit **`00-namespace-and-config.yaml`** and change one value:

```yaml
  problem-mode: "on"      # was "off"
```

Commit it. Argo CD syncs within about a minute, and the services pick up the new
value **without restarting** (they re-read the mounted config on every request).
Allow 1–2 minutes for Kubernetes to refresh the mounted file, then errors begin.

To restore the healthy state, set it back to `"off"` and commit.

---

## What actually breaks, and why

`positions-store` models a database connection pool. In problem mode the pool is
starved — only `db-pool-size` connections, each held for `db-query-ms` — which is
far below what the live traffic needs. Requests queue, then time out waiting for a
connection:

```
connection pool exhausted: could not acquire a connection to the positions
database within 1500ms (pool size 2, 3 requests queued)
```

Because **two different applications** read from that one datastore, the failure
fans out:

| Entity | Effect |
|---|---|
| `positions-store` | **Root cause** — 503s, connection pool exhausted |
| `portfolio-service` | Impacted — 502s, depends on positions-store |
| `client-profile-service` | Impacted — 502s, depends on positions-store |
| `web-portal` | Impacted — client dashboards and holdings fail |
| `advisor-api` | Impacted — client reviews fail |
| `pricing-service` | **Healthy** — proves the failure is not "everything is broken" |

Measured locally at the shipped settings: **~32% failure rate**, with slow
responses on everything that touches the datastore.

---

## The fix

The root cause is a configuration value, and so is the remedy — raise the pool
size in `00-namespace-and-config.yaml`:

```yaml
  db-pool-size: "10"     # was "2"
```

Commit, and the platform recovers as Argo CD syncs. (Or simply set
`problem-mode: "off"`.)

---

## Verifying

```bash
# watch the failures happen
kubectl -n wealth-demo logs -l app=positions-store --tail=20 -f

# the impacted upstream service
kubectl -n wealth-demo logs -l app=portfolio-service --tail=20

# overall traffic health, printed every 30s
kubectl -n wealth-demo logs -l app=traffic-generator --tail=10
```

### Tuning severity

In `00-namespace-and-config.yaml`: lower `db-pool-size` or raise `db-query-ms`
for a worse outage. For more load, raise `VUS` in `05-traffic-generator.yaml`
(this one does require a pod restart, since it is an environment variable).

---

## Notes

* Changing the **application code** in `01-app-code.yaml` needs a restart:
  `kubectl -n wealth-demo rollout restart deploy`.
* Changing **`problem-mode`, `db-pool-size`, or `db-query-ms`** does not.
* Everything is namespaced to `wealth-demo`, so it is isolated from the rest of
  the cluster and easy to remove: `kubectl delete namespace wealth-demo`.

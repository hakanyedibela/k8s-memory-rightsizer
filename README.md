# k8s-memory-rightsizer

A minimal CronJob that periodically queries Prometheus for the **peak memory usage** of a target workload over a window, applies a headroom margin, clamps to safe bounds, and patches the workload's `requests`/`limits` accordingly.

Think of it as a small, opinionated alternative to VPA's recommender — useful when you want **policy you control** (clamps, cooldowns, opt-in per workload) instead of running the full VPA stack.

Works on **vanilla Kubernetes** and **OpenShift** (an example for the platform Prometheus is included).

---

## When to use this

- You have a workload with a known memory leak or growth pattern.
- Grafana alerts fire repeatedly and someone has to manually bump memory.
- You want automated, bounded resizing without installing VPA / Goldilocks / KRR.

## When *not* to use this

- The leak is unbounded — this only delays OOM, it doesn't fix the bug.
- You also have an HPA scaling on memory for the same workload (they will fight).
- You need recommendations across the whole cluster — use Goldilocks / KRR / VPA recommender instead.

---

## How it works

1. CronJob runs on a schedule (default: every 6h).
2. Container queries Prometheus:
   ```
   max(max_over_time(container_memory_working_set_bytes{namespace="...",pod=~"...",container="..."}[24h]))
   ```
3. Computes `new = peak * (1 + HEADROOM_PCT/100)`, clamped to `[MIN_MI, MAX_MI]`.
4. Compares to current request — skips if change is below `CHANGE_THRESHOLD_PCT` (avoids restart churn).
5. Runs `kubectl set resources` to patch the workload, which triggers a rolling restart.

The rolling restart is intentional: it both resizes the workload **and** resets the leak.

---

## Layout

```
.
├── manifests/
│   ├── 01-rbac.yaml         # ServiceAccount + Role + RoleBinding
│   ├── 02-configmap.yaml    # the rightsize.sh script
│   └── 03-cronjob.yaml      # CronJob with env-var configuration
├── examples/
│   ├── openshift-cronjob.yaml    # OpenShift platform Prometheus variant
│   └── statefulset-cronjob.yaml  # StatefulSet target variant
└── docs/
    └── queries.md           # Prometheus / Thanos verification queries
```

Before deploying, run through [`docs/queries.md`](docs/queries.md) to confirm the
metrics this project relies on are reachable and labelled the way you expect.

---

## Install

### Vanilla Kubernetes

```sh
kubectl apply -f manifests/
```

Then edit `manifests/03-cronjob.yaml` to point at your workload and Prometheus, and re-apply.

### OpenShift

```sh
# RBAC + script ConfigMap
kubectl apply -f manifests/01-rbac.yaml
kubectl apply -f manifests/02-configmap.yaml

# Allow the SA to query the platform Prometheus
oc adm policy add-cluster-role-to-user cluster-monitoring-view \
  -z rightsizer -n default

# CronJob that uses the SA token + thanos-querier
kubectl apply -f examples/openshift-cronjob.yaml
```

---

## Configuration (env vars on the CronJob)

| Var | Required | Default | Purpose |
|---|---|---|---|
| `NAMESPACE` | yes | — | Namespace of the target workload |
| `WORKLOAD_KIND` | yes | — | `deployment` or `statefulset` |
| `WORKLOAD_NAME` | yes | — | Name of the target workload |
| `CONTAINER` | yes | — | Container name inside the pod template |
| `PROM_URL` | yes | — | Prometheus base URL (no trailing slash) |
| `WINDOW` | no | `24h` | PromQL lookback window for the peak |
| `HEADROOM_PCT` | no | `30` | Safety margin above observed peak |
| `MIN_MI` | no | `256` | Floor for the new memory value (MiB) |
| `MAX_MI` | no | `4096` | **Ceiling — your circuit breaker against runaway leaks** |
| `CHANGE_THRESHOLD_PCT` | no | `10` | Skip patch if change is smaller than this |
| `POD_REGEX` | no | `${WORKLOAD_NAME}-.*` | Override if name prefix collides with other workloads |
| `PROM_TOKEN` | no | — | Bearer token for protected Prometheus |
| `DRY_RUN` | no | `false` | Log only, don't patch |

---

## Recommended rollout

1. **Start with `DRY_RUN=true`** for a few cycles. Check `kubectl logs` on the Job pod and verify:
   - The Prometheus query returns data.
   - The computed `new` value is sane.
2. Lower the schedule to `*/15 * * * *` temporarily for faster feedback during tuning.
3. Once happy, set `DRY_RUN=false` and restore the long schedule.
4. Set `MAX_MI` to the largest value you'd ever willingly give this workload. **When the script repeatedly hits `MAX_MI`, that's the signal that you have a real leak that needs an application fix** — not just more memory.

---

## Gotchas

- **Patching `resources` triggers a rolling restart.** That's the point, but don't run the cron more often than your rollout completes.
- **Don't combine with HPA on memory.** The two will fight. HPA on CPU or custom metrics is fine.
- **`POD_REGEX` collisions.** If `WORKLOAD_NAME=app` and you also have `app-cron`, both will match. Override `POD_REGEX` with something stricter.
- **OpenShift `thanos-querier` uses self-signed certs.** The script uses `curl -k`. If you want strict TLS, mount the cluster CA bundle and remove `-k`.
- **Bare pods can't be targeted.** Only Deployments / StatefulSets (anything `kubectl set resources` understands).

---

## Verifying it ran

```sh
# Most recent Job pod
kubectl -n default get pods -l job-name --sort-by=.metadata.creationTimestamp | tail -1

# Logs
kubectl -n default logs -l job-name=<job-pod-name>

# Confirm new requests/limits on the target
kubectl -n default get deployment leaky-app \
  -o jsonpath='{.spec.template.spec.containers[*].resources}{"\n"}'
```

---

## Uninstall

```sh
kubectl delete -f manifests/
```

---

## License

MIT

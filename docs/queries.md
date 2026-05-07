# Verifying Prometheus / Thanos before deploying the rightsizer

Run these queries to confirm the metrics the CronJob depends on are scraped, labelled the way you expect, and reachable from inside the cluster. Every query uses the standard Prometheus HTTP API (`/api/v1/...`) — so the same commands work against Prometheus, Thanos, Cortex, Mimir, and VictoriaMetrics.

---

## 0. Setup: how to reach the API

Pick one of these and use it as `$PROM` in the rest of the doc.

### Vanilla Kubernetes (kube-prometheus-stack, prometheus-operator)

```sh
# Port-forward Prometheus to localhost
kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090 &
PROM=http://localhost:9090
AUTH=()                              # no auth header needed
```

### OpenShift platform Prometheus (Thanos Querier)

```sh
# From your laptop
oc -n openshift-monitoring port-forward svc/thanos-querier 9091:9091 &
PROM=https://localhost:9091
TOKEN=$(oc whoami -t)
AUTH=(-k -H "Authorization: Bearer $TOKEN")
```

### From inside the cluster (e.g. a debug pod)

```sh
kubectl run curl-test --rm -it --image=alpine/k8s:1.29.4 -- sh
# inside the pod:
PROM=http://prometheus.monitoring.svc:9090            # vanilla
# or
PROM=https://thanos-querier.openshift-monitoring.svc:9091   # OpenShift
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
AUTH=(-k -H "Authorization: Bearer $TOKEN")
```

> All examples below use `curl -sG "${AUTH[@]}"`. If your `$AUTH` is empty, the array just expands to nothing — works either way.

---

## 1. Health & reachability

### Is the API up?

```sh
curl -sG "${AUTH[@]}" "$PROM/-/healthy"
# Expect: HTTP 200, body "Prometheus Server is Healthy."
```

### Is it ready to serve queries?

```sh
curl -sG "${AUTH[@]}" "$PROM/-/ready"
# Expect: HTTP 200, body "Prometheus Server is Ready."
```

### Build / version info

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/status/buildinfo" | jq
```

Useful response keys: `.data.version`, `.data.revision`. Confirms the API is the Prometheus dialect (Thanos returns its own version here).

---

## 2. Does the metric we need exist?

The rightsizer queries `container_memory_working_set_bytes`. Confirm it's scraped:

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode 'query=container_memory_working_set_bytes' \
  | jq '.data.result | length'
```

| Result | Meaning |
|---|---|
| `0` | Metric isn't being scraped — check that cAdvisor / kubelet targets are up. |
| `> 0` | Metric exists. |

### List every metric name (sanity check)

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/label/__name__/values" \
  | jq '.data | map(select(startswith("container_"))) | .[]'
```

Should include `container_memory_working_set_bytes`, `container_memory_usage_bytes`, `container_memory_rss`, `container_cpu_usage_seconds_total`, etc.

---

## 3. Are the labels we filter on populated?

The rightsizer filters on `namespace`, `pod`, `container`. Verify each:

```sh
# Namespaces that have container memory data
curl -sG "${AUTH[@]}" "$PROM/api/v1/series" \
  --data-urlencode 'match[]=container_memory_working_set_bytes' \
  | jq -r '.data[].namespace' | sort -u

# Containers inside one namespace
NS=default
curl -sG "${AUTH[@]}" "$PROM/api/v1/series" \
  --data-urlencode "match[]=container_memory_working_set_bytes{namespace=\"$NS\"}" \
  | jq -r '.data[].container' | sort -u

# Pods matching your workload prefix
WORKLOAD=leaky-app
curl -sG "${AUTH[@]}" "$PROM/api/v1/series" \
  --data-urlencode "match[]=container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"${WORKLOAD}-.*\"}" \
  | jq -r '.data[].pod' | sort -u
```

If any of these come back empty, fix that **before** deploying the CronJob — the script will exit with `no data returned — skipping`.

> **OpenShift gotcha:** the platform Prometheus filters series by namespace based on the requesting token's RBAC. If you query as a SA without `cluster-monitoring-view`, you'll silently get an empty result — not an error.

---

## 4. The exact query the rightsizer runs

Run it manually first. Replace the variables with your values:

```sh
NS=default
POD_REGEX='leaky-app-.*'
CONTAINER=app
WINDOW=24h

curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode "query=max(max_over_time(container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"}[$WINDOW]))" \
  | jq
```

### Interpreting the response

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      { "metric": {}, "value": [1715000000.123, "1342177280"] }
    ]
  }
}
```

| Field | Meaning |
|---|---|
| `status` | `"success"` or `"error"` — anything else means transport/proxy failure. |
| `data.resultType` | `"vector"` for instant queries (what we use). |
| `data.result` | Empty array `[]` = no matching series. **This is the silent-failure case.** |
| `data.result[0].value[1]` | The value as a string. Bytes for memory metrics. |

Convert that bytes value to MiB the same way the script does:

```sh
echo "1342177280" | awk '{printf "%.0f Mi\n", $1/1024/1024}'
# 1280 Mi
```

### Common error responses

```json
{ "status": "error", "errorType": "bad_data", "error": "..." }   # malformed PromQL
```

```json
{ "status": "error", "errorType": "execution", "error": "..." }  # query timed out / too expensive
```

If you get HTML back instead of JSON, you're hitting an auth proxy (very common with OpenShift `thanos-querier` if the token is missing/expired).

---

## 5. Useful queries for understanding the leak

These don't drive the rightsizer, but they're worth running to understand the workload's behavior before you let the CronJob start patching.

### Current memory per pod (instant)

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode "query=container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"}/1024/1024" \
  | jq -r '.data.result[] | "\(.metric.pod): \(.value[1] | tonumber | floor) Mi"'
```

### Growth rate over the last hour (MiB/min)

A non-zero, sustained positive value is leak-shaped.

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode "query=deriv(container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"}[1h])*60/1024/1024" \
  | jq -r '.data.result[] | "\(.metric.pod): \(.value[1] | tonumber) Mi/min"'
```

### Predicted memory in 4 hours

Linear extrapolation. If this exceeds your `MAX_MI`, you have a real leak that the CronJob will only paper over.

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode "query=predict_linear(container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"}[1h], 4*3600)/1024/1024" \
  | jq -r '.data.result[] | "\(.metric.pod): \(.value[1] | tonumber | floor) Mi predicted"'
```

### Memory pressure vs. limit (0–1, where 1.0 = OOM imminent)

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode "query=container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"} / on(pod,container) kube_pod_container_resource_limits{resource=\"memory\"}" \
  | jq -r '.data.result[] | "\(.metric.pod): \(.value[1] | tonumber | .*100 | floor)%"'
```

> Requires `kube-state-metrics`. Anything sustained above ~85% is the early warning your Grafana alert is probably firing on.

### OOMKill events in the last 24h

```sh
curl -sG "${AUTH[@]}" "$PROM/api/v1/query" \
  --data-urlencode "query=increase(kube_pod_container_status_terminated_reason{reason=\"OOMKilled\",namespace=\"$NS\"}[24h])" \
  | jq -r '.data.result[] | "\(.metric.pod)/\(.metric.container): \(.value[1])"'
```

### Range query (raw time series, useful for piping into a chart)

```sh
END=$(date +%s)
START=$(( END - 24*3600 ))
curl -sG "${AUTH[@]}" "$PROM/api/v1/query_range" \
  --data-urlencode "query=container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"}" \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "step=300" \
  | jq '.data.result[0].values | map([.[0], (.[1] | tonumber / 1024 / 1024)])'
```

---

## 6. Quick troubleshooting reference

| Symptom | Likely cause | Where to look |
|---|---|---|
| `status: success`, `result: []` | Label mismatch — wrong namespace/pod/container, or metric not scraped for that namespace | Section 3 — list actual label values |
| HTTP 401 / 403 | Missing or expired bearer token | Section 0 — re-issue token |
| HTML response, not JSON | Hitting an oauth-proxy without auth | Add the `Authorization` header |
| `errorType: bad_data` | Malformed PromQL — usually quoting | Run the query in Grafana's Explore tab to validate |
| `errorType: execution`, `query timed out` | Window too long over too many series | Narrow `pod=~` regex, shorten `WINDOW` |
| Connection refused | Port-forward died, or wrong service name | Section 0 — check `kubectl get svc -A \| grep -i prom` |
| Rightsizer logs `no data returned — skipping` after deploy | The in-cluster query returns empty even though it works locally | Most often: SA lacks `cluster-monitoring-view` on OpenShift |

---

## 7. One-liner: end-to-end sanity check

Drop this into a script and run before flipping `DRY_RUN=false`:

```sh
#!/bin/sh
set -eu
: "${PROM:?set PROM to your Prometheus URL}"
NS="${NS:-default}"
POD_REGEX="${POD_REGEX:-leaky-app-.*}"
CONTAINER="${CONTAINER:-app}"
WINDOW="${WINDOW:-24h}"

QUERY="max(max_over_time(container_memory_working_set_bytes{namespace=\"$NS\",pod=~\"$POD_REGEX\",container=\"$CONTAINER\"}[$WINDOW]))"

RESP=$(curl -sSfk ${TOKEN:+-H "Authorization: Bearer $TOKEN"} -G \
  --data-urlencode "query=$QUERY" "$PROM/api/v1/query")

VAL=$(echo "$RESP" | jq -r '.data.result[0].value[1] // empty')
if [ -z "$VAL" ]; then
  echo "FAIL: no data. Response was:"
  echo "$RESP" | jq
  exit 1
fi

MI=$(echo "$VAL" | awk '{printf "%d", $1/1024/1024}')
echo "OK: peak over $WINDOW = ${MI} Mi"
```

If that prints `OK:` you're good to deploy.

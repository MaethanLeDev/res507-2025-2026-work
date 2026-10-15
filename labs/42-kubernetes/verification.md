# Lab 42 — Verification

Environment: Ubuntu 26.04.1 LTS VM, k3s `v1.36.5+k3s1`, runtime `containerd://2.3.4-k3s1.36`, namespace `quote-lab`.

## Image import

```
$ sudo k3s ctr images list | grep quote-app:local
docker.io/library/quote-app:local
```

## Health requests

Port-forward (`kubectl port-forward svc/quote-app 8080:80`):

```
$ curl --fail --show-error --max-time 5 http://127.0.0.1:8080/health
{"ok":true}
```

In-cluster, through Service DNS (`service-check` busybox Pod):

```
$ kubectl exec service-check -- wget -qO- http://quote-app/health
{"ok":true}
```

## Three Ready replicas

```
$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS    RESTARTS   AGE
quote-app-69fbbcff49-44ds6   1/1     Running   0          14s
quote-app-69fbbcff49-5646w   1/1     Running   0          44s
quote-app-69fbbcff49-ghtwv   1/1     Running   0          14s
```

## Pod replacement (reconciliation)

```
$ kubectl delete pod quote-app-69fbbcff49-44ds6
$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS    RESTARTS   AGE
quote-app-69fbbcff49-5646w   1/1     Running   0          50s
quote-app-69fbbcff49-9xqrh   1/1     Running   0          6s     <- replacement, new name
quote-app-69fbbcff49-ghtwv   1/1     Running   0          20s
```

Deleted: `quote-app-69fbbcff49-44ds6` → replacement: `quote-app-69fbbcff49-9xqrh`, created by the ReplicaSet `quote-app-69fbbcff49`.

## Deliberate readiness failure

Readiness probe path changed from `/health` to `/does-not-exist`:

```
$ kubectl rollout status deployment/quote-app --timeout=60s
error: timed out waiting for the condition

$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS    RESTARTS   AGE
quote-app-57b88c776b-qcv5n   0/1     Running   0          61s
quote-app-69fbbcff49-5646w   1/1     Running   0          2m1s
quote-app-69fbbcff49-9xqrh   1/1     Running   0          77s
quote-app-69fbbcff49-ghtwv   1/1     Running   0          91s

$ kubectl describe pod quote-app-57b88c776b-qcv5n
    Readiness:  http-get http://:3000/does-not-exist delay=3s timeout=1s period=5s #success=1 #failure=3
  Warning  Unhealthy  1s (x12 over 56s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404

$ kubectl get endpointslices -l kubernetes.io/service-name=quote-app   (targetRef.name / conditions.ready)
quote-app-69fbbcff49-5646w  ready=true
quote-app-69fbbcff49-ghtwv  ready=true
quote-app-69fbbcff49-9xqrh  ready=true
quote-app-57b88c776b-qcv5n  ready=false
```

RESTARTS stayed at `0`: a failing readiness probe does not restart the container. The old ReplicaSet kept 3 Ready Pods, so the Service still answered `{"ok":true}`.

## Recovery

```
$ kubectl apply -f deployment.yaml   # path restored to /health
$ kubectl rollout status deployment/quote-app --timeout=120s
deployment "quote-app" successfully rolled out
$ kubectl get pods -l app=quote-app
quote-app-57b88c776b-qcv5n   0/1     Terminating   0          72s
quote-app-69fbbcff49-5646w   1/1     Running       0          2m12s
quote-app-69fbbcff49-9xqrh   1/1     Running       0          88s
quote-app-69fbbcff49-ghtwv   1/1     Running       0          102s
$ kubectl exec service-check -- wget -qO- http://quote-app/health
{"ok":true}
```

## Symptom / cause / fix

- **Symptom:** rollout timed out; the new Pod stayed `Running` but `0/1` Ready and was marked `ready=false` in the Service EndpointSlice.
- **Cause:** the readiness probe path was changed to `/does-not-exist`; the app has no such route, so the kubelet got HTTP 404 (`Readiness probe failed: HTTP probe failed with statuscode: 404`).
- **Fix:** restored the readiness probe path to `/health`, which returns `200 {"ok":true}`. The probe succeeds again, so the Pods become Ready and the rollout completes with 3/3 Ready.

## Extension — Lane B: liveness probe

Added a `livenessProbe` on `/health` (port 3000, `initialDelaySeconds: 5`, `periodSeconds: 10`). Then changed **only** the liveness path to `/does-not-exist` (readiness kept on `/health`):

```
$ kubectl get pods -l app=quote-app      # ~90 s after the change
NAME                       READY   STATUS    RESTARTS      AGE
quote-app-c85d74c4-mfc75   1/1     Running   2 (26s ago)   87s
quote-app-c85d74c4-mvhd7   1/1     Running   2 (20s ago)   81s
quote-app-c85d74c4-sw672   0/1     Running   3 (2s ago)    93s

$ kubectl describe pod quote-app-c85d74c4-mfc75
    Last State:     Terminated
      Reason:       Error
    Restart Count:  2
    Liveness:       http-get http://:3000/does-not-exist delay=5s timeout=1s period=10s #success=1 #failure=3

$ kubectl get events --field-selector involvedObject.name=quote-app-c85d74c4-mfc75
Normal    Killing     Container quote-app failed liveness probe, will be restarted
Warning   Unhealthy   Liveness probe failed: HTTP probe failed with statuscode: 404
```

Restored the liveness path to `/health`:

```
$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS    RESTARTS   AGE
quote-app-58897d6459-9d9wd   1/1     Running   0          52s
quote-app-58897d6459-hkwjh   1/1     Running   0          46s
quote-app-58897d6459-k54t2   1/1     Running   0          60s
```

**What it demonstrates:** after 3 consecutive liveness failures (~30 s), the kubelet kills and restarts the container, so RESTARTS keeps increasing. In the core exercise, only readiness failed: the kubelet just removed the Pod from Service endpoints and never restarted it. Readiness controls *traffic*; liveness controls *restarts*.

## What this evidence leaves untested

`/health` returns `{"ok": true}` hard-coded; it does not check PostgreSQL. Quote listing and insertion are not validated here (no database deployed yet, see Lab 52). Three replicas on a single node do not protect against losing the VM.

# ActiveMQ Classic — Active/Passive HA Test (Kubernetes Lease, no persistence)

Same active/passive goal as [../README-kahadb-ha.md](../README-kahadb-ha.md),
but built for a use case that doesn't need message durability: no shared
storage, no KahaDB, no NAS dependency. Each broker pod is a fully
independent, persistence-free ActiveMQ instance (`persistent="false"` — no
store loaded at all, not just non-persistent messages). "Which pod is
active" is decided by a sidecar doing leader election against a Kubernetes
`Lease` object, not by a shared-storage lock.

## How the HA works

Each pod in the `amq` StatefulSet runs two containers:

- **`activemq`** — a fully independent broker. It always starts completely
  and binds `61616`/`8161` immediately; nothing blocks it like KahaDB's
  lock did in the other setup.
- **`leader-elector`** — a small `sh` loop (`alpine/k8s:1.30.14` image, so
  it has a version-matched `kubectl` plus `jq`) that repeatedly tries to
  claim/renew a single `Lease` object (`amq-active`) shared by both pods.
  It writes `true`/`false` to a file on an `emptyDir` shared with the
  `activemq` container.

The `activemq` container's `readinessProbe` just checks that shared file
(`test "$(cat /shared/leader)" = true`). So:

- Both pods are always fully running and listening.
- Only the pod whose elector currently holds the Lease is marked **Ready**.
- The `amq-broker` Service's Endpoints therefore always point at exactly
  one pod — the current leader — same externally-visible behavior as the
  KahaDB setup, just arbitrated differently.

**Claiming the lease safely:** the elector reads the Lease's
`resourceVersion`, decides whether to claim it (empty holder, holder is
itself, or the holder's last renewal is older than `LEASE_TTL=15s`), then
submits a JSON Patch with a `test` op on that `resourceVersion` before the
actual update. If another pod's patch lands first, the `resourceVersion`
no longer matches and the patch is atomically rejected — this is what
prevents two pods from ever both believing they're leader.

**Identity matters:** a StatefulSet reuses the same pod name after a
restart, so identity is `hostname-<random-uuid-suffix>` generated fresh
each container start, not just `hostname`. Using bare `hostname` was tried
first and was a real bug: a just-restarted pod would see the Lease still
said its own old hostname and instantly re-claim it, completely skipping
the staleness wait and defeating any kill test.

## RBAC

Each pod's ServiceAccount (`amq-broker`) is bound to a Role scoped to
`get`/`patch` on exactly one named Lease (`resourceNames: [amq-active]`) —
no broader cluster access.

## Files

| File | Purpose |
|---|---|
| `00-namespace.yaml` | `amq-lease-ha` namespace |
| `01-rbac.yaml` | ServiceAccount + Role + RoleBinding, scoped to one Lease |
| `02-lease.yaml` | The `amq-active` Lease, pre-initialized so `replace` patches always have a target |
| `03-configmap-activemq.yaml` | `activemq.xml` with `persistent="false"` — no store at all |
| `04-configmap-elector.yaml` | `elector.sh` — the leader-election loop |
| `05-headless-service.yaml` | Required by the StatefulSet |
| `06-service.yaml` | Client-facing Service; Endpoints = current lease holder |
| `07-statefulset.yaml` | 2 replicas, 2 containers each, readiness gated on the shared leader-status file |

## Deploy

```
kubectl apply -f manifests-lease-ha/
kubectl -n amq-lease-ha get pods -o wide
```

Check who's leader:
```
kubectl -n amq-lease-ha get lease amq-active -o jsonpath='{.spec.holderIdentity}'
kubectl -n amq-lease-ha get endpoints amq-broker
```

## Testing failover

```
kubectl -n amq-lease-ha get lease amq-active -o jsonpath='{.spec.holderIdentity}'
kubectl -n amq-lease-ha delete pod <leader-pod> --grace-period=0 --force
watch kubectl -n amq-lease-ha get endpoints amq-broker
```

### Verified 2026-09-04

- Killed the leader repeatedly; the Lease was reclaimed and the Service
  Endpoint updated within **~15-24s** each time — bounded by
  `LEASE_TTL=15s` plus `POLL_INTERVAL=3s` plus readiness-probe timing.
  Noticeably slower than the KahaDB setup's ~6-9s, because a Lease has no
  "release on death" signal — nobody is told the holder died, so every
  contender has to wait out the full TTL before assuming it's gone.
  Tunable: shorten `LEASE_TTL`/`POLL_INTERVAL` in `04-configmap-elector.yaml`
  for faster failover at the cost of more frequent Kubernetes API calls.
- On several runs, the Lease was reclaimed by the **restarted same-named
  pod**, not the other standby — both are equally valid contenders once
  the Lease goes stale; who wins is a genuine race (safely arbitrated by
  the `resourceVersion` compare-and-swap, so never both at once).
- Load test (5 producer threads + 3 consumer threads, `failover://` broker
  URL) survived a leader kill mid-flight: producer logged `Successfully
  reconnected`, consumer resumed draining, no manual restart needed.
  In-flight non-persistent messages at the moment of the kill were lost
  (expected — there's no store to recover them from), same tradeoff as the
  non-persistent test against the KahaDB broker.

### Performance, 2026-09-04

Single-thread except where noted, no sleep, against a broker with no
persistence adapter loaded at all:

| Test | Result | Rate |
|---|---|---|
| `--persistent false`, 1 thread | 20,000 msgs / 1,359 ms | ~14,700 msg/sec |
| `--persistent false`, 5 threads | 100,000 msgs / ~4,200 ms | ~23,800 msg/sec aggregate |
| `--persistent true` (flag accepted, nothing stored) | 5,000 msgs / 1,374 ms | ~3,639 msg/sec |

Faster than the KahaDB non-persistent number (~10,000 msg/sec) since there's
no store anywhere in the path at all, and it scales further with more
producer threads. The `--persistent true` result is the interesting one:
the broker never errors and never stores anything, but is still ~4x slower
than `--persistent false` on the *exact same* broker — proof that part of
what people call "the persistence tax" is actually the persistent
delivery mode's synchronous-send/wait-for-ack client behavior, not disk
I/O itself.

## Known caveats / things to watch

- **No durability at all, by design.** There is no store, so *any* message
  not yet consumed at the moment a pod dies is gone — not just in-flight
  ones. Only pick this setup if that's genuinely acceptable.
- **Failover latency is bounded by `LEASE_TTL`, not instant.** Unlike a
  lock that's released the moment its holder's process dies, nobody
  "tells" the standby the leader is gone — it has to notice the lease
  wasn't renewed in time. Don't expect KahaDB-level failover speed without
  tuning the TTL down.
- **Cluster memory is tight on this homelab.** The `amq-test` (KahaDB)
  namespace was deleted specifically to free ~1Gi of memory requests so
  these two pods (160Mi request / 320Mi limit each, `ACTIVEMQ_OPTS_MEMORY`
  capped to `-Xmx192M`) could schedule. If you bring both HA setups back
  at once, check `kubectl describe nodes` for headroom first.
- Default `admin`/`admin` credentials from the stock image — fine for an
  isolated test namespace only.
- Not wired into Flux — `kubectl apply`-only, isolated in `amq-lease-ha`.

## Teardown

```
kubectl delete namespace amq-lease-ha
```
No PV/PVC involved, so nothing else to clean up afterward.

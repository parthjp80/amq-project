# ActiveMQ Classic — Active/Passive HA Test (KahaDB / shared storage)

> **Status (2026-09-04): torn down.** `kubectl delete namespace amq-test`
> was run to free cluster memory for the second HA experiment
> ([lease-based](manifests-lease-ha/README.md), see the top-level
> [README.md](README.md)). Everything below is fully reproducible with
> `kubectl apply -f manifests/` — nothing here depended on manual/undocumented
> state.

Apache ActiveMQ 5.19.2 ("classic"), shared-filesystem master/slave, deployed
as plain Kubernetes manifests (no operator, no Helm) against the homelab
cluster for HA testing. Standalone `kubectl apply` for now — not wired into
Flux yet.

This approach needs message **persistence** — the master/slave election
itself is implemented as a lock inside the shared KahaDB store. See
[README.md](README.md) for how this compares to the persistence-free,
Lease-based approach in `manifests-lease-ha/`.

## How the HA works

Two pods (`amq-0`, `amq-1`) run the *exact same* broker config and both
mount the *same* KahaDB data directory from a single RWX PVC (`nfs-csi`
StorageClass, backed by the NAS at 192.168.1.219). ActiveMQ's KahaDB
persistence adapter uses a file lock inside that shared directory to decide
who's master:

- Whichever pod acquires the lock starts fully (binds `61616` openwire,
  `8161` web console) and becomes **master**.
- The other pod blocks at startup ("`... in slave mode waiting a lock to be
  acquired`") and does **not** bind any ports — it's a hot standby.
- If the master dies, its lock is released; the standby acquires it within
  a few seconds (`lockAcquireSleepInterval=2000` in the config) and starts
  serving.

Kubernetes routes to whichever pod is currently master via a
`readinessProbe` (TCP check on 61616): only the master passes it, so the
`amq-broker` Service's Endpoints always point at the live master. There is
**no livenessProbe on the broker port** — the standby not listening is
correct/expected behavior, and a liveness probe there would kill the
healthy standby in a loop.

## Files

| File | Purpose |
|---|---|
| `manifests/00-namespace.yaml` | `amq-test` namespace |
| `manifests/01-pvc.yaml` | Shared RWX PVC for KahaDB (`nfs-csi`) |
| `manifests/02-configmap.yaml` | `activemq.xml` — shared-file-locker config |
| `manifests/03-headless-service.yaml` | Required by the StatefulSet; per-pod DNS for debugging |
| `manifests/04-service.yaml` | Client-facing Service; Endpoints always = current master |
| `manifests/05-statefulset.yaml` | 2 replicas, anti-affinity (preferred), readiness-gated |

## Deploy

```
kubectl apply -f manifests/
kubectl -n amq-test get pods -o wide
```

Check who's master:
```
kubectl -n amq-test logs amq-0 | grep -E "started|slave mode"
kubectl -n amq-test get endpoints amq-broker   # IP shown = current master
```

## Testing failover

```
# note current master
kubectl -n amq-test get endpoints amq-broker

# kill it
kubectl -n amq-test delete pod <master-pod> --grace-period=0 --force

# watch the standby take over (usually ~5-10s)
watch kubectl -n amq-test get endpoints amq-broker
```

### Verified 2026-09-04

- Killed the master repeatedly; standby was promoted and passed readiness
  within ~6-9 seconds each time. Service endpoints updated automatically.
- Durability check: produced 1000 persistent messages to `queue://TEST.HA`
  via the master, killed that pod, then consumed all 1000 back out from the
  newly-promoted master. Zero message loss.
- CLI test tools live inside the image, e.g.:
  ```
  kubectl -n amq-test exec <pod> -- /opt/apache-activemq/bin/activemq producer \
    --destination queue://TEST.HA --persistent true --user admin --password admin
  kubectl -n amq-test exec <pod> -- /opt/apache-activemq/bin/activemq consumer \
    --destination queue://TEST.HA --user admin --password admin
  ```

### Performance (persistent vs non-persistent), 2026-09-04

Single-thread, no sleep, `activemq producer --messageCount 5000`, against
the NFS-backed KahaDB store:

| Mode | Time | Rate |
|---|---|---|
| Persistent | 9,816 ms | ~509 msg/sec |
| Non-persistent | 500 ms | ~10,000 msg/sec |

~19.6x faster non-persistent. The gap is unusually large here specifically
because every persistent-message fsync is a network round-trip to the NAS
(192.168.1.219) rather than local disk. Non-persistent messages skip the
store entirely (they only ever live in the broker's memory), so a broker
crash loses anything that was only queued, not yet consumed — confirmed by
killing the master mid-load with 3,212 non-persistent messages queued: the
promoted broker showed `QueueSize: 0` afterward. Contrast with the
persistent durability test above, where 1000 persistent messages produced
just before a kill were fully recovered.

## Known caveats / things to watch

- **Both pods currently land on the same node (`k8s-worker-2`).** The
  StatefulSet has `preferredDuringSchedulingIgnoredDuringExecution`
  pod anti-affinity (soft), not `required` (hard) — soft was chosen
  deliberately so pods don't go `Pending` if a node lacks room. Right now
  `k8s-worker-1` is already at ~89% of its allocatable memory *requests*
  from other homelab workloads, so the scheduler packs both AMQ pods onto
  worker-2. This means today's failover test only proves *process-level*
  failover, not *node-level* failure. To force a true node-loss test, free
  up memory on worker-1 or temporarily cordon worker-2.
- **NFS locking caveat**: shared-filesystem master/slave depends on the
  underlying filesystem's locking being trustworthy. Plain NFS advisory
  locks have a history of being flaky, which is why the config explicitly
  uses `<shared-file-locker/>` (polling-based, not relying on raw fcntl
  semantics) rather than the default locker. This has worked cleanly in
  testing so far, but keep an eye out for split-brain if the NFS server
  itself has issues.
- **The NFS server (192.168.1.219) is a new single point of failure** for
  this setup — if it goes down, both master and slave lose their store.
  That's an accepted tradeoff for a HA *broker process* test; it's not
  storage-layer HA.
- Default `admin`/`admin` credentials from the stock image — fine for an
  isolated test namespace, don't reuse if this ever gets promoted beyond
  `amq-test`.
- Not wired into Flux — this is `kubectl apply`-only in the `amq-test`
  namespace, isolated from the rest of the GitOps-managed homelab stack.

## Teardown

```
kubectl delete namespace amq-test
```
(The PVC's `nfs-csi` StorageClass has `reclaimPolicy: Retain`, so the
underlying NFS-backed volume will need manual cleanup on the NAS/PV if you
want the space back.)

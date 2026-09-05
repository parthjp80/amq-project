# ActiveMQ Active/Passive HA — Homelab Test

Two different ways to run Apache ActiveMQ 5.19.2 ("classic") active/passive
on the homelab Kubernetes cluster, tested live under load and failure. Pick
based on whether you need message durability:

| | Needs durability? | Doc |
|---|---|---|
| **KahaDB / shared storage** | Yes | [README-kahadb-ha.md](README-kahadb-ha.md) |
| **Kubernetes Lease election** | No | [manifests-lease-ha/README.md](manifests-lease-ha/README.md) |

## Architecture 1: KahaDB shared-storage master/slave

Election is implemented *as* a lock inside the persistence store itself —
there's no separate arbitration mechanism. Requires RWX shared storage
(NFS in this cluster), which also becomes a new single point of failure.

```mermaid
flowchart LR
    subgraph clients [" "]
        P["Producer / Consumer"]
    end

    P -->|"tcp://61616<br/>failover-aware"| SVC

    subgraph k8s ["Kubernetes: amq-test namespace"]
        SVC["Service: amq-broker<br/>(readinessProbe-gated)"]

        subgraph pod0 ["Pod: amq-0"]
            AMQ0["activemq<br/>MASTER<br/>listening 61616/8161"]
        end
        subgraph pod1 ["Pod: amq-1"]
            AMQ1["activemq<br/>SLAVE<br/>blocked, not listening"]
        end

        SVC -->|"routed (Ready)"| AMQ0
        SVC -.->|"excluded (Not Ready)"| AMQ1
    end

    AMQ0 <-->|"read/write +<br/>file lock"| PVC
    AMQ1 <-.->|"polls for lock<br/>(lockAcquireSleepInterval)"| PVC

    subgraph storage ["Shared storage"]
        PVC[("PVC: amq-kahadb-shared<br/>RWX, nfs-csi")]
        NFS[("NAS<br/>192.168.1.219")]
        PVC --- NFS
    end

    style AMQ0 fill:#2d6a4f,color:#fff
    style AMQ1 fill:#6c757d,color:#fff
    style NFS fill:#7f1d1d,color:#fff
```

**Failure mode tested:** kill `amq-0` → `amq-1`'s KahaDB lock-poll acquires
the now-released lock (~6-9s) → it starts fully → Service Endpoints flip to
it. Persistent messages produced right before the kill survive (recovered
from the shared store); non-persistent ones don't.

## Architecture 2: Kubernetes Lease leader election

No shared storage, no persistence adapter at all (`persistent="false"`).
Each broker pod is fully independent; a sidecar in the same pod does
leader election against a `Lease` object (an etcd-backed lock — the same
primitive Kubernetes controllers use internally).

```mermaid
flowchart LR
    subgraph clients [" "]
        P["Producer / Consumer"]
    end

    P -->|"tcp://61616<br/>failover-aware"| SVC

    subgraph k8s ["Kubernetes: amq-lease-ha namespace"]
        SVC["Service: amq-broker<br/>(readinessProbe-gated)"]

        subgraph pod0 ["Pod: amq-0"]
            direction TB
            AMQ0["activemq<br/>always fully running<br/>listening 61616/8161"]
            EL0["leader-elector sidecar<br/>(kubectl + jq loop)"]
            FILE0["/shared/leader = true"]
            EL0 -->|writes| FILE0
            FILE0 -->|readinessProbe reads| AMQ0
        end
        subgraph pod1 ["Pod: amq-1"]
            direction TB
            AMQ1["activemq<br/>always fully running<br/>listening 61616/8161"]
            EL1["leader-elector sidecar<br/>(kubectl + jq loop)"]
            FILE1["/shared/leader = false"]
            EL1 -->|writes| FILE1
            FILE1 -->|readinessProbe reads| AMQ1
        end

        SVC -->|"routed (Ready)"| AMQ0
        SVC -.->|"excluded (Not Ready)"| AMQ1

        LEASE["Lease: amq-active<br/>holderIdentity + renew-epoch<br/>(resourceVersion = CAS lock)"]
        EL0 <-->|"get / patch<br/>(claims + renews)"| LEASE
        EL1 <-->|"get / patch<br/>(polls, contends when stale)"| LEASE
    end

    style AMQ0 fill:#2d6a4f,color:#fff
    style AMQ1 fill:#6c757d,color:#fff
    style LEASE fill:#1d4e89,color:#fff
```

**Failure mode tested:** kill `amq-0` → its Lease entry goes stale after
`LEASE_TTL=15s` with no active renewal → whichever pod's poll loop hits
next claims it via a `resourceVersion`-guarded patch (~15-24s total,
observed both `amq-1` and a restarted `amq-0` win this race) → Service
Endpoints flip. Nothing survives a kill — there's no store, so any
unconsumed message is gone regardless of when it was produced.

## Summary of what was verified

| | KahaDB master/slave | Lease election |
|---|---|---|
| Shared storage required | Yes (RWX NFS) | No |
| Failover time (observed) | ~6-9s | ~15-24s |
| Persistent messages survive failover | Yes | N/A (no persistence) |
| Non-persistent messages survive failover | No | No |
| Peak throughput (non-persistent) | ~10,000 msg/sec | ~23,800 msg/sec (5 threads) |
| Client reconnect after failover | Automatic (`failover://` URL) | Automatic (`failover://` URL) |

Full details, exact commands, and caveats for each are in their respective
READMEs linked above. Neither is wired into Flux — both are
`kubectl apply -f <dir>/`-only, kept isolated from the rest of the
GitOps-managed homelab stack.

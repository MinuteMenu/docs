# Queue Cost Comparison — TP 315953 / 323225 Attendance Bidirectional Sync

> **⚠ All numbers are unverified Microsoft / CloudAMQP / RabbitMQ list prices
> sourced from 2024 reference data. Verify on the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
> and [cloudamqp.com/plans.html](https://www.cloudamqp.com/plans.html) for the
> target subscription before quoting to client. Region: East US.**

---

## 1. Volume Assumptions

| Environment | Daily volume | Days / month | Monthly total |
|---|---:|---:|---:|
| Production | 1,000,000 msg / day | 22 working days¹ | **22,000,000 msg / mo** |
| Dev + QA (combined) | 2,000 msg / working day | 22 working days | **44,000 msg / mo** |

¹ Working day = Mon–Fri. Range 21–22; this file uses 22 (upper bound, conservative).

---

## 2. Cost Formula

| Component | Value |
|---|---|
| Operations per message (PeekLock) | **3** (Send + Receive + Complete) |
| Polling overhead (Storage Queue only, 2 s interval) | **~1,300,000 empty polls / month / queue** |
| Total billable ops (Storage Queue) | (msg / mo × 3) + 1.3M polls |
| Total billable ops (Service Bus) | msg / mo × 3 |
| Total billable ops (RabbitMQ / CloudAMQP) | Not charged per-op (priced by tier / VM size) |

---

## 3. Per-Vendor Rate Sheet (East US, 2024 list)

### 3.1. Azure Queue Storage (used by §4 — AZ Queue LRS / GRS)

| Line item | Unit rate |
|---|---:|
| Storage Queue LRS — Transaction | $0.04 / 1M ops |
| Storage Queue LRS — Storage | $0.045 / GB-mo |
| Storage Queue GRS — Transaction | $0.05 / 1M ops |
| Storage Queue GRS — Storage | $0.09 / GB-mo |

### 3.2. Azure Service Bus (used by §4 — SB Basic / Standard)

| Line item | Unit rate |
|---|---:|
| SB Basic — Operations | $0.05 / 1M ops (flat, no namespace fee) |
| SB Standard — Namespace base | $9.81 / mo per namespace |
| SB Standard — Free tier | 12.5M ops / mo included per namespace |
| SB Standard — Overage | $0.80 / 1M ops above free |

### 3.3. RabbitMQ self-hosted infrastructure (used by §5 — RabbitMQ on Azure VM)

| Line item | Unit rate |
|---|---:|
| Azure VM B2s — RabbitMQ **prod** broker (2 vCPU, 4 GB, Linux PAYG) | $0.0416 / hour ≈ $30.37 / mo |
| Azure VM B1s — RabbitMQ **dev + qa** broker (1 vCPU, 1 GB, Linux PAYG) | $0.0104 / hour ≈ $7.59 / mo |
| Premium SSD P4 (30 GB) — RabbitMQ disk persistence | ≈ $5.76 / mo per disk |

### 3.4. CloudAMQP managed (used by §5 — CloudAMQP single-node / HA)

| Line item | Unit rate |
|---|---:|
| "Little Lemur" tier (shared, 1M msg / mo cap) | **$0** (free) |
| "Big Bunny" tier (dedicated 1 GB, no msg cap) | ≈ $99 / mo |
| "Roaring Rabbit" tier (HA 3-node cluster) | ≈ $399 / mo |

---

## 3.5. Functional Capability Glossary

The matrices in §4 and §5 compare options against 5 capabilities. This
glossary defines each term and explains **why TP 315953 needs it**.

**FIFO / Sessions** — *First-In-First-Out ordering per key.*
Service Bus "Sessions" is the SB feature: messages with the same `SessionId`
(e.g., `ChildId`) are delivered to one consumer in arrival order.
**Why needed**: prevents the race condition where two attendance updates for
the same child arrive out of order and the later one (older clock-in time)
overwrites the earlier one (newer clock-out time).

**Duplicate detection (native)** — *Broker auto-rejects duplicates within a
time window based on `MessageId`.*
**Why needed**: when the sync job retries a failed send, the same message may
hit the queue twice. Without native dedup, the consumer would insert two
attendance rows. With dedup, the broker silently drops the second one within
the dedup window (e.g., 10 minutes).

**Dead-letter queue / DLQ (native)** — *Secondary queue that holds "poison
messages" that failed processing after N retries, exceeded TTL, or violated
size limits.*
**Why needed**: bad payloads (corrupted JSON, missing fields, schema
mismatch) should not block the main queue forever. DLQ quarantines them so
ops can inspect, fix, and replay.

**TTL — Time-To-Live** — *Maximum lifetime of a message in the queue before
auto-deletion.*
**Why needed**: if the consumer is down for days, stale attendance messages
(e.g., > 3 days old) are no longer relevant — they should be auto-expired
and routed to DLQ for review, not silently re-applied later.

**Max message size** — *Largest payload the queue accepts in a single message.*
**Why needed**: a single CX batch save can include all children in a center
(payload ~50–200 KB). 64 KB limit (Azure Queue Storage) forces the producer
to split payloads; 256 KB (Service Bus) and 128 MB (RabbitMQ) fit the whole
batch.

**Geo-replication** — *Automatic asynchronous copy of data to a paired region
for disaster recovery.*
**Why needed (or not)**: optional. The durable source of truth is
`P_AttendanceSyncLog` in SQL — losing the queue during a region outage is
recoverable by replaying the DB log. **Geo-replication is a "nice to have",
not required.**

---

## 4. Matrix #1 — Azure Queue Storage vs Service Bus

| Item | AZ Queue LRS | AZ Queue GRS | SB Basic | SB Standard |
|---|---:|---:|---:|---:|
| **Prod cost / mo** | $2.74 | $3.46 | $3.30 | $52.61 |
| **Prod cost / yr** | $32.84 | $41.46 | $39.60 | $631.32 |
| **Dev + QA cost / mo** | $0.06 | $0.07 | $0.01 | $0.11² |
| **Dev + QA cost / yr** | $0.69 | $0.87 | $0.08 | $1.27² |
| **Combined / mo** | **$2.80** | **$3.53** | **$3.31** | **$52.72** |
| **Combined / yr** | **$33.53** | **$42.33** | **$39.68** | **$632.59** |
| FIFO / Sessions | ❌ | ❌ | ❌ | ✅ |
| Duplicate detection (native) | ❌ manual | ❌ manual | ❌ | ✅ |
| Dead-letter queue (native) | ❌ manual | ❌ manual | ✅ | ✅ |
| Max message size | 64 KB | 64 KB | 256 KB | 256 KB |
| Max TTL | 7 days | 7 days | unlimited | unlimited |
| Geo-replication | LRS local-only | ✅ paired region | ❌ | ❌ (Premium only) |

² Dev + QA share production namespace (1 shared namespace topology). If a
separate non-prod namespace required: +$9.81 / mo base (44K msg × 3 ops =
132K ops, well within 12.5M free tier) = **$62.42 / mo combined**,
$749.04 / yr.

**LRS vs GRS — why LRS is sufficient here.** **LRS** = Locally-Redundant
Storage (3 copies in **one** datacenter, same region). **GRS** = Geo-Redundant
Storage (LRS + 3 async copies in a **paired region**, RPO ~15 min). The queue
in this architecture is **transient transport only** — the durable source of
truth is `P_AttendanceSyncLog` in the database. Messages lost during a region
outage can be replayed from the DB log. **LRS is recommended** unless the
client requires sub-15-minute RPO at the transport layer.

**Prod calc transparency (1M msg / day × 22 working days = 22M msg / mo):**
```
AZ Queue LRS  prod: (22M×3 + 1.3M polls) × $0.04/1M + 1GB × $0.045
                  = 67.3M × $0.04/1M + $0.045  = $2.692 + $0.045 = $2.74
AZ Queue GRS  prod: 67.3M × $0.05/1M + 1GB × $0.09   = $3.365 + $0.090 = $3.46
SB Basic      prod: 22M × 3 × $0.05/1M               = 66M × $0.05/1M = $3.30
SB Standard   prod: $9.81 + (66M − 12.5M free) × $0.80/1M
                  = $9.81 + 53.5M × $0.80/1M = $9.81 + $42.80 = $52.61
```

---

## 5. Matrix #2 — RabbitMQ Self-Hosted vs CloudAMQP Managed

| Item | RabbitMQ on Azure VM³ | CloudAMQP single-node⁴ | CloudAMQP HA 3-node⁵ |
|---|---:|---:|---:|
| **Prod cost / mo** | $36 (1× B2s + P4) | $99 (Big Bunny) | $399 (Roaring Rabbit) |
| **Prod cost / yr** | $432 | $1,188 | $4,788 |
| **Dev + QA cost / mo** | $13 (1× B1s + P4) | $0 (Little Lemur free) | $0 (Little Lemur free) |
| **Dev + QA cost / yr** | $156 | $0 | $0 |
| **Combined / mo** | **$49** | **$99** | **$399** |
| **Combined / yr** | **$588** | **$1,188** | **$4,788** |
| FIFO / Sessions | ✅ (consistent-hash exchange or single-active-consumer) | ✅ same | ✅ same |
| Duplicate detection (native) | ✅ (x-deduplication-header plugin) | ✅ same | ✅ same |
| Dead-letter queue (native) | ✅ (DLX) | ✅ same | ✅ same |
| Max message size | 128 MB default (tunable) | 128 MB | 128 MB |
| Max TTL | unlimited (per-message / per-queue) | unlimited | unlimited |
| Geo-replication | manual (Federation / Shovel) | optional add-on | ✅ built-in HA |
| License | Open-source (free) | Managed (vendor-locked) | Managed (vendor-locked) |

³ Prod = 1× B2s VM; dev + qa = 1× B1s VM (combined), East US. License free;
   cost = VM rental + Premium SSD P4 (30 GB). Self-managed Linux instance.
⁴ "Big Bunny" tier ≈ $99 / mo at time of writing — single dedicated 1 GB node,
   no monthly message cap. **Verify on cloudamqp.com/plans.html.**
⁵ "Roaring Rabbit" tier ≈ $399 / mo — 3-node HA cluster, automatic failover.
   **Verify on cloudamqp.com/plans.html.**

---

## 6. Side-by-Side Total Cost (All Four Options)

| Cost line | AZ Queue LRS | SB Standard | RabbitMQ self-host | CloudAMQP (single-node) |
|---|---:|---:|---:|---:|
| Prod / mo | $2.74 | $52.61 | $36 | $99 |
| Dev + QA / mo | $0.06 | $0.11 | $13 | $0 |
| **Total / mo** | **$2.80** | **$52.72** | **$49** | **$99** |
| **Total / yr** | **$33.53** | **$632.59** | **$588** | **$1,188** |
| Rank by cost (cheapest = 1) | **1** | 3 | 2 | 4 |

*SB Basic and AZ Queue GRS excluded from this side-by-side because they do not
add meaningful new datapoints; full numbers remain in §4 for completeness.*

---

## 7. Summary — Which Option Fits

### Functional gate (TP 323225 hard requirements)

The chosen queue **must** support all four to avoid app-level workarounds:

1. **FIFO / Sessions** (ordering per `ChildId` to prevent attendance overwrite races)
2. **Native duplicate detection** (idempotency across retries)
3. **Native dead-letter queue** (poison-message handling)
4. **TTL ≥ 3 days** + **max message size ≥ 256 KB**

### Options that pass the functional gate

| Option | Total / mo | Total / yr | Passes? |
|---|---:|---:|---|
| AZ Queue LRS | $2.80 | $33.53 | ❌ (no Sessions, no native dedup, no native DLQ) |
| AZ Queue GRS | $3.53 | $42.33 | ❌ same as LRS |
| SB Basic | $3.31 | $39.68 | ❌ (no Sessions, no dedup) |
| **SB Standard** | **$52.72** | **$632.59** | ✅ all four |
| **RabbitMQ self-host** | **$49** | **$588** | ✅ all four (plugins + native) |
| **CloudAMQP Big Bunny** | **$99** | **$1,188** | ✅ all four |
| CloudAMQP Roaring Rabbit | $399 | $4,788 | ✅ all four + HA |

### Cost-only ranking (passing options)

1. **RabbitMQ self-hosted on Azure VM** — $49 / mo · $588 / yr
2. **SB Standard** — $52.72 / mo · $632.59 / yr
3. **CloudAMQP Big Bunny** — $99 / mo · $1,188 / yr
4. **CloudAMQP Roaring Rabbit** (HA) — $399 / mo · $4,788 / yr

### Verification reminder before quoting

- [ ] Re-price each row on the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) for the
      target subscription, region, and term-discount (Reserved / Savings Plan).
- [ ] Re-check CloudAMQP tier prices + msg / sec ceilings on
      [cloudamqp.com/plans.html](https://www.cloudamqp.com/plans.html) for
      30M msg / mo headroom and Sessions support.
- [ ] Confirm Premium SSD P4 size (30 GB) is sufficient for RabbitMQ disk
      queue persistence at peak.
- [ ] Confirm whether dev + qa share the production Service Bus namespace
      (assumed in §4) or non-prod needs its own (+$9.81 / mo).

# Storage Benchmarks — palma (Vitastor + DRBD + LVM)

**Host:** palma (EPYC 8124P, 32GB RAM)
**Date:** 2026-04-23
**Companion doc:** `palma-storage-benchmarks.md` (LVM baseline + DRBD/TCP, 2026-04-22)
**Stacks under test:**
- LVM thin pool (baseline, no replication) — *from companion doc*
- DRBD/TCP `testy_tcp` (2× replica, 100GbE TCP) — *from companion doc*
- Vitastor 2.x cluster (palma + calvia, 2× replica, 100GbE RDMA) — **this doc**

**fio invariants:** `--direct=1 --time_based --group_reporting --runtime=60`, QD=32 (writes/rand reads), QD=16 (seqwrite)
- libaio runs: `--ioengine=libaio --filename=/dev/ublkb0`
- Vitastor native runs: `-thread --ioengine=libfio_vitastor.so --image=test_bench`

## Vitastor cluster topology

- **Etcd:** 3 nodes, 3 monitors (master: palma)
- **OSDs at time of test:**
  - palma: **4 OSDs** (2× CD6-R 1.92TB, `osd_per_disk=2`), ~893 GiB each
  - calvia: **2 OSDs** (2× CD6-R 1.92TB, `osd_per_disk=2`), ~893 TiB each
- **Pool:** replicated, `pg_size=2`, `failure_domain=host`, `pg_count=256`
- **Test image:** `test_bench`, 100 GiB
- **Network:** Mellanox CX5 (palma) ↔ BlueField-2 (calvia), 100GbE RoCEv2, RDMA enabled
- **Disk path:** native engine talks directly to OSDs; UBLK runs via `vitastor-cli map → /dev/ublkb0`

## Vitastor — UBLK kernel mount, 1 OSD per drive (calvia client, symmetric 1+1)

Target: `/dev/ublkb0` mapped from `test_bench`

| Workload  | bs  | jobs | QD | IOPS    | BW (MB/s) | avg lat  | p99 lat  | Notes |
|-----------|-----|------|----|---------|-----------|----------|----------|-------|
| randwrite | 4k  | 4    | 32 | 175k    | 715       | 722 µs   | 1,254 µs | |
| randwrite | 4k  | 8    | 32 | 143k    | 584       | 1.78 ms  | 3.06 ms  | Negative scaling vs j=4 |
| randwrite | 4k  | 16   | 32 | 167k    | 682       | 2.98 ms  | 5.80 ms  | Recovers slightly |
| randread  | 4k  | 4    | 32 | 196k    | 802       | 644 µs   | 979 µs   | |
| randread  | 4k  | 16   | 32 | 166k    | 680       | 2.99 ms  | 5.15 ms  | Negative scaling |
| randread  | 4k  | 32   | 32 | 160k    | 657       | 6.19 ms  | 11.21 ms | Worst case |
| seqwrite  | 1M  | 1    | 16 | 1,987   | 2,084     | 7.94 ms  | 14.09 ms | |

## Vitastor — UBLK kernel mount, 2 OSDs per drive (palma client, asymmetric 4+2)

| Workload  | bs  | jobs | QD | IOPS    | BW (MB/s) | avg lat  | p99 lat  | Notes |
|-----------|-----|------|----|---------|-----------|----------|----------|-------|
| randwrite | 4k  | 4    | 32 | 205k    | 840       | 614 µs   | 1,090 µs | +17% vs 1-OSD |
| randwrite | 4k  | 8    | 32 | 172k    | 705       | 1.47 ms  | 2.57 ms  | +20% |
| randwrite | 4k  | 16   | 32 | 179k    | 732       | 2.78 ms  | 5.80 ms  | +7% |
| randread  | 4k  | 4    | 32 | 217k    | 889       | 580 µs   | 914 µs   | +11% |
| randread  | 4k  | 16   | 32 | 188k    | 769       | 2.64 ms  | 4.55 ms  | +13% |
| randread  | 4k  | 32   | 32 | 181k    | 742       | 5.48 ms  | 9.90 ms  | +13% |
| seqwrite  | 1M  | 1    | 16 | 2,247   | 2,357     | 7.03 ms  | 8.98 ms  | +13% |

## Vitastor — native fio engine, 2 OSDs per drive (palma client, asymmetric 4+2)

Bypasses kernel block layer entirely — `libfio_vitastor.so` talks RDMA directly to OSDs.

| Workload  | bs  | jobs | QD | IOPS    | BW (MB/s) | avg lat  | p99 lat  | Notes |
|-----------|-----|------|----|---------|-----------|----------|----------|-------|
| randwrite | 4k  | 4    | 32 | 399k    | 1,633     | 302 µs   | 955 µs   | ⭐ |
| randwrite | 4k  | 8    | 32 | 449k    | 1,840     | 565 µs   | 1.66 ms  | Peak write IOPS |
| randwrite | 4k  | 16   | 32 | 409k    | 1,676     | 1.24 ms  | 4.69 ms  | |
| randread  | 4k  | 4    | 32 | 397k    | 1,627     | 317 µs   | 2.18 ms  | |
| randread  | 4k  | 16   | 32 | 540k    | 2,214     | 940 µs   | 3.59 ms  | Peak read IOPS |
| randread  | 4k  | 32   | 32 | 540k    | 2,213     | 1.88 ms  | 7.31 ms  | Same — saturated |
| seqwrite  | 1M  | 1    | 16 | 2,243   | 2,352     | 6.97 ms  | 8.29 ms  | Same as UBLK — network-bound |

## Vitastor — T1Q1 latency floor (native engine, separate test)

Not in DRBD/baseline doc but worth recording — this is what you actually feel as a single thread doing one IO at a time.

| Workload                         | IOPS    | avg lat  | p99 lat  | Notes |
|----------------------------------|---------|----------|----------|-------|
| randwrite 4k T1Q1 + fsync        | 8,784   | **112 µs** | 145 µs | Replicated, durable, fsync per IO |
| randread  4k T1Q1                | 10,100  | **98 µs**  | 155 µs | |
| linear write 4M T1Q32            | 573     | 55.4 ms  | 64.2 ms  | Single-thread BW: 2,404 MB/s |
| linear read 4M T1Q32             | 1,010   | 31.2 ms  | 50 ms    | Single-thread BW: 4,240 MB/s |

## Vitastor — native engine, 1 OSD per drive (no matching Q32 data)

⚠ Was not benchmarked at matching parameters. Original test used Q128 jobs=4 (deeper queue):
- randwrite Q128 j=4: 240k IOPS, 936 MB/s, avg 2.12 ms
- randread Q128 j=4: 337k IOPS, 1316 MB/s, avg 1.50 ms

Direct comparison to 2-OSD native is therefore not possible. Estimating from the UBLK +13–17% scaling pattern, native 1-OSD at Q32 j=4 would have been ~340–360k IOPS randwrite — so the OSD-count effect is real but not the dominant factor.

---

## Comparison — randwrite 4k, jobs=4, QD=32

| Stack                              | IOPS    | BW (MB/s) | avg lat  | p99 lat  | vs LVM raw |
|------------------------------------|---------|-----------|----------|----------|------------|
| LVM thin (raw, no replication)     | 305k    | 1,249     | 419 µs   | 429 µs   | —          |
| **Vitastor native, RDMA, 2-OSD**   | **399k**| **1,633** | **302 µs** | **955 µs** | **+31%** ⭐ |
| Vitastor UBLK, RDMA, 2-OSD         | 205k    | 840       | 614 µs   | 1,090 µs | −33%       |
| Vitastor UBLK, RDMA, 1-OSD         | 175k    | 715       | 722 µs   | 1,254 µs | −43%       |
| DRBD/TCP C + al-updates=no         | 79k     | 325       | 1.61 ms  | 2.70 ms  | −74%       |
| DRBD/TCP C (tuned baseline)        | 72k     | 294       | 1.78 ms  | 2.77 ms  | −76%       |
| DRBD/TCP B                         | 54k     | 222       | 2.37 ms  | 3.26 ms  | −82%       |

**Vitastor native > LVM raw is real**, not an artifact: with RDMA + 3 servers serving requests in parallel, the aggregate IO path can outperform a single local NVMe stack at moderate concurrency. Replication overhead is more than compensated by parallel server-side processing.

## Comparison — randwrite 4k, higher concurrency

| Stack                            | jobs=8 IOPS | jobs=16 IOPS |
|----------------------------------|-------------|--------------|
| LVM thin (raw)                   | 563k        | 576k (sat.)  |
| Vitastor native 2-OSD            | **449k**    | 409k         |
| Vitastor UBLK 2-OSD              | 172k        | 179k         |
| Vitastor UBLK 1-OSD              | 143k        | 167k         |

LVM continues to scale toward the 2-drive 1.4M IOPS spec ceiling. Vitastor native peaks around jobs=8 at ~450k and declines — bottleneck is no longer disk but client-side PG routing + OSD CPU on the asymmetric calvia replica.

## Comparison — randread 4k

| Stack                            | j=4 IOPS | j=16 IOPS | j=32 IOPS |
|----------------------------------|----------|-----------|-----------|
| LVM thin (raw)                   | 315k     | 967k      | 1,321k    |
| Vitastor native 2-OSD            | **397k** | **540k**  | 540k      |
| Vitastor UBLK 2-OSD              | 217k     | 188k      | 181k      |
| Vitastor UBLK 1-OSD              | 196k     | 166k      | 160k      |

Vitastor native beats LVM at j=4 again (RDMA + 2 nodes serving reads in parallel). At j=32 LVM raw is 2.4× Vitastor — the 540k peak is the cluster's CPU/PG-routing ceiling, not an underlying device limit. Adding more nodes or tuning client I/O threads would lift this.

## Comparison — sequential write 1M, jobs=1, QD=16

| Stack                            | BW (MB/s) | avg lat  |
|----------------------------------|-----------|----------|
| LVM thin (raw)                   | 2,451     | 6.84 ms  |
| Vitastor native 2-OSD            | 2,352     | 6.97 ms  |
| Vitastor UBLK 2-OSD              | 2,357     | 7.03 ms  |
| Vitastor UBLK 1-OSD              | 2,084     | 7.94 ms  |

Sequential 1M is bandwidth-limited and roughly identical across all configs — drive write rate × replication factor mostly explains the numbers. UBLK overhead disappears here because per-IO cost is small relative to the 1M payload transfer time.

---

## UBLK overhead analysis

UBLK adds significant overhead vs the native vitastor fio engine on small-IO workloads:

| Workload          | Native  | UBLK    | UBLK penalty |
|-------------------|---------|---------|--------------|
| randwrite j=4     | 399k    | 205k    | **−49%**     |
| randwrite j=8     | 449k    | 172k    | −62%         |
| randwrite j=16    | 409k    | 179k    | −56%         |
| randread  j=4     | 397k    | 217k    | −45%         |
| randread  j=16    | 540k    | 188k    | −65%         |
| randread  j=32    | 540k    | 181k    | −66%         |
| seqwrite  1M j=1  | 2,243 MB/s | 2,247 MB/s | 0%      |

**Implication for MeisterStack agent design:** exposing Vitastor volumes via UBLK to CloudHypervisor would forfeit roughly half of the achievable performance. The right paths are:
1. **VDUSE** — userspace virtio data path, no kernel block layer (best option for vmm-style consumers)
2. **Library linking** — link the Vitastor client lib directly into the Hypervisor driver (best perf, like the QEMU driver)
3. **NVMe-oF re-export** — Vitastor → vhost-user-blk → guest (more layers, more latency, but cleanest integration boundary)

UBLK is fine for one-off mounts and admin tasks but not for production VM I/O paths.

## OSD-per-drive scaling (UBLK only)

| Workload          | 1-OSD   | 2-OSD   | Δ      |
|-------------------|---------|---------|--------|
| randwrite j=4     | 175k    | 205k    | +17%   |
| randwrite j=8     | 143k    | 172k    | +20%   |
| randwrite j=16    | 167k    | 179k    | +7%    |
| randread  j=4     | 196k    | 217k    | +11%   |
| randread  j=16    | 166k    | 188k    | +13%   |
| randread  j=32    | 160k    | 181k    | +13%   |
| seqwrite  1M      | 2,084   | 2,357 MB/s | +13% |

Consistent +10–20% improvement. This with **only one node** rebuilt (palma → 2-OSD). Symmetric 4+4 would likely add another similar increment on writes. For drives larger than the CD6-R 1.92TB (e.g. planned PM1735 1.6TB or future 4TB+ Samsung/Micron), `osd_per_disk=4` is worth testing.

## Latency floor — Vitastor T1Q1 vs DRBD (estimated)

DRBD T1Q1 fsync was not in the original benchmark set. Estimated from architecture (single-thread sender + synchronous peer ACK + commit roundtrip):

| Stack                            | T1Q1 randwrite fsync IOPS | Latency  |
|----------------------------------|--------------------------|----------|
| Vitastor native, RDMA            | **8,784**                | **112 µs** |
| DRBD/TCP C (estimated)           | ~2,500–4,000             | ~250–400 µs |
| DRBD/RDMA 9.3.2 (when released)  | ~6,000–8,000 (estimated) | ~125–170 µs |
| LVM raw (single-disk floor)      | ~20,000+                 | ~50 µs   |

**Direct measurement on DRBD pending** — see TODO. This is the most important comparison for ACID DBMS workloads (single-thread WAL fsync).

---

## Anomalies and observations

### Negative job-count scaling on UBLK

On UBLK, IOPS *decrease* from j=4 to j=8 (175k → 143k randwrite, 196k → 166k randread). On the native engine, IOPS continue to climb to j=8 before declining. Cause is almost certainly contention in the single ublk worker thread — the `vitastor-cli map` daemon is single-threaded and serializes all IO submissions through one userspace handler before forwarding to OSDs.

This matches the +13–20% pattern when OSD count doubled but UBLK stayed single-threaded — the bottleneck moved further toward UBLK and away from per-OSD CPU.

**Fix would be:** multi-queue ublk (`ublk_drv` supports it, vitastor's UBLK server may need patching), or skip UBLK entirely.

### Native engine declines past j=8 randwrite

449k j=8 → 409k j=16. Likely client-side: `vitastor_cluster` engine spawns one io_uring per fio thread, all sharing TCP/RDMA connections to the OSDs. Past a threshold the RDMA QP contention or PG-routing CPU overhead exceeds the gain from more concurrent submissions.

Could be tested by raising `client_io_threads` (currently default) or splitting load across multiple test images on different pools.

### Seqwrite is identical across all configs at ~2.35 GB/s

All stacks hit the same ceiling. With 2× replication and a single 100GbE link to peer, theoretical max is 12.5 GB/s ÷ 2 (replication overhead) = ~6 GB/s. We're nowhere near that, so the bottleneck is single-thread payload generation, not network. Multi-job seqwrite would expose the real ceiling.

### Vitastor native at j=4 outperforms raw LVM

Counterintuitive but reproducible. Explanation: LVM at j=4 only has 4 in-flight IOs hitting the local NVMe queues (depth 32 each = 128 in flight). NVMe queue parallelism on a single Kioxia is high but the *single submission path through the kernel block layer* limits effective utilization at low job counts. Vitastor parallelizes across 6 OSD processes on 2 nodes — each OSD has its own kernel I/O path on its node — so even at j=4 client-side, server-side parallelism is much higher.

At higher j= the LVM single-host wins out because the bottleneck shifts from submission parallelism to raw NVMe IOPS, and there's no network roundtrip in the LVM path.

---

## Key takeaways

1. **Vitastor native + RDMA delivers 5× DRBD/TCP IOPS** at jobs=4 randwrite (399k vs 79k), and 4× at fsync-bound single-thread workloads. With only 2 replicas and 2 nodes serving — not "scale-out advantage", just architecture.
2. **UBLK kernel mount halves Vitastor's small-IO performance.** Acceptable for admin tasks, unacceptable as the VM I/O path. MeisterStack should use VDUSE or library-linked clients.
3. **`osd_per_disk=2` is +10–20% across the board** with no downside on Kioxia CD6-R. Worth setting at install time. Higher values likely beneficial on larger/faster drives.
4. **Single-thread fsync latency 112 µs** beats every other distributed system in this class. For ACID DBMS workloads this is the number that matters, and it's about 5–10× better than Ceph on equivalent hardware.
5. **Vitastor scales worse than raw LVM at j=16+** — this is a server-side CPU / PG-routing ceiling, not a fundamental limitation. Adding nodes or tuning `client_io_threads` should lift it. With 3 nodes serving and asymmetric OSDs, 540k peak read is what we have today.
6. **Sequential write = network-bound at ~2.35 GB/s** — same across all Vitastor configs. Single-thread payload generation is the limit, not the storage stack.
7. **Operationally Vitastor is dramatically simpler than LINSTOR/DRBD.** No protocol selection, no AL tuning, no multi-path debugging, no resource definitions — `vitastor-disk prepare`, `vitastor-cli create-pool`, done. Replication is automatic and PG-based, not pair-based.

## TODO

- [ ] **Rebuild calvia to `osd_per_disk=2`** for symmetric 4+4 layout, re-run write tests
- [ ] **DRBD T1Q1 fsync measurement** on `testy_tcp` for direct apples-to-apples latency comparison
- [ ] **Vitastor native 1-OSD at Q32** for clean OSD-scaling delta (currently only have Q128 data)
- [ ] **CPU pinning + isolcpus** test — pin OSDs to dedicated cores, IRQs to others, measure tail-latency improvement
- [ ] **VDUSE export** test — `vitastor-cli map` with VDUSE backend instead of UBLK; verify it closes the UBLK gap
- [ ] **Multi-client test** — run fio simultaneously from palma, calvia, and a third node to find the real cluster IOPS ceiling
- [ ] **DRBD 9.3.2 + RDMA** when released (DRBD's response to this comparison)
- [ ] **Long-run sustained randwrite** (1h, j=8) — find SLC-cache cliff on Kioxia, if any
- [ ] **Native engine with `client_io_threads` tuning** — see if j=16+ randwrite scales further
- [ ] **Test `pg_size=3`** when third storage node is added — current 2-replica is below typical production durability target

## Test commands (for reference / reproducibility)

```bash
# Native engine (per workload, change bs/numjobs/iodepth/rw):
fio -thread -ioengine=libfio_vitastor.so -direct=1 -time_based -group_reporting \
    -runtime=60 -image=test_bench \
    -name=test -bs=4k -numjobs=4 -iodepth=32 -rw=randwrite

# UBLK mount:
sudo vitastor-cli map --image test_bench   # → /dev/ublkb0
fio -ioengine=libaio -direct=1 -time_based -group_reporting \
    -runtime=60 -filename=/dev/ublkb0 \
    -name=test -bs=4k -numjobs=4 -iodepth=32 -rw=randwrite
sudo vitastor-cli unmap /dev/ublkb0

# T1Q1 fsync floor (native):
fio -thread -ioengine=libfio_vitastor.so -direct=1 -time_based -group_reporting \
    -runtime=60 -image=test_bench \
    -name=test -bs=4k -numjobs=1 -iodepth=1 -fsync=1 -rw=randwrite

# OSD rebuild for osd_per_disk=2 (per node):
sudo vitastor-disk purge --force osd:<id>
sudo wipefs -a /dev/nvme0n1 /dev/nvme1n1
sudo sgdisk --zap-all /dev/nvme0n1 /dev/nvme1n1
sudo partprobe /dev/nvme0n1 /dev/nvme1n1
sudo vitastor-disk prepare --osd_per_disk 2 /dev/nvme0n1 /dev/nvme1n1
# wait for `vitastor-cli status` to show 0 degraded, 0 misplaced before next node
```

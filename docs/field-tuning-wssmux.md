# Field notes: a CPU-saturated WSS Mux pair

Notes from tuning one production reverse WSS Mux tunnel on 2026-11-22. They
describe one deployment, not a benchmark. The method carries over; the numbers
do not.

## The deployment

| | Entry server (`[server]`) | Exit server (`[client]`) |
|---|---|---|
| vCPUs | 8, generic "QEMU Virtual CPU" (sse/sse2/sse3 only) | 8, AMD EPYC with AES/AVX2 |
| NIC | virtio, 8 queues | virtio, 1 queue |
| Kernel | 6.8 | 7.0 |
| Role | accepts users on 443, forwards over the tunnel | delivers to a local Xray on 127.0.0.1:443 |

Preset `throughput`, 99 ms RTT, about 1% TCP retransmits, 30-40k concurrent
user connections, about 850 Mbit/s at peak. The entry server ran at 93% CPU.

## What did not move the load

| Change | Result |
|---|---|
| v1.26.0 to v1.27.0 | Entry CPU unchanged. RSS fell about 45%, but only because the process was fresh. |
| `mux_streambuffer` 32 MiB to 16 MiB | No change in CPU or throughput. |
| Physical sessions 1,873 to about 1,060 (side effect of the restart) | No change in entry CPU. |

The tunnel was never window-limited. Before the change, the fastest single user
flow was about 245 Mbit/s, which needs about 3 MB in flight at this RTT, and
kernel socket statistics showed essentially no receive-window-limited time. The
smaller stream window is still worth having on WS Mux and WSS Mux, because a
stream window equal to the session window lets one stalled stream hold the whole
session's receive credit. It is protection, not a speed-up.

Each restart cost about two minutes of dropped connections: with
`channel_size = 8192` the admission queue overflowed while every user
reconnected at once (about 28,000 `the queue is full` drops after one restart).

## Where the entry server's CPU went

`perf record -a -g` for 10 seconds: 66% kernel, 33% backpack.

| Cost | Share of all CPU | Cause |
|---|---|---|
| Receive softirq path | 29.5% | 100k packets/s in each direction |
| `write` syscalls | 13.3% | relay writes |
| TCP pacing timers (`tcp_pace_kick`) | 7.3% | `eth0` used `mq` + `pfifo_fast`, so BBR paced with one kernel timer per socket |
| ChaCha20-Poly1305, generic Go code | about 8% | the VM exposes no AES-NI, SSSE3 or AVX, so Go's TLS has no assembly path |
| virtio TX notifications (`iowrite16`) | 6.5% | legacy virtio on an i440FX machine type |
| Netfilter hooks | about 4% | six nft tables with no rules still run their hooks per packet |

The Go binary is stripped, but `go tool addr2line` resolves `perf` addresses
through the pclntab of the same release binary (add the ELF load address,
`0x400000`).

## What helped, with no tunnel restart

**Entry server: use `fq` under `mq`.** `net.core.default_qdisc = fq` was set,
but the interface had come up before the sysctl applied and kept `pfifo_fast`.

```sh
tc qdisc replace dev eth0 root handle 1: mq
for i in 1 2 3 4 5 6 7 8; do tc qdisc replace dev eth0 parent 1:$i fq; done
```

Timer work fell from 8.7% to 0.8% of CPU; pacing inside `fq` costs about 2%.

**Entry server: remove empty nft tables.** Check that they really hold no rules
first, and keep a copy (`nft list ruleset > backup.nft`).

```sh
for t in "ip filter" "ip raw" "ip6 filter" "ip6 raw" "ip mangle" "ip6 mangle"; do nft delete table $t; done
```

`nf_hook_slow` fell from 2.4% to 0.17%.

Together, the kernel's share fell from 66.4% to 62.9%. The box stayed at 93%
busy because it was saturated, so the gain appeared as throughput per core:
8.76 to 8.34 host cores per Gbit/s (single 60-second samples; treat as
approximate).

**Exit server: RPS/RFS for the single-queue NIC.** One core did all receive
work (62% softirq, 14% idle), a one-core ceiling for the whole path.

```sh
sysctl -w net.core.rps_sock_flow_entries=65536
echo 65536 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
echo fb > /sys/class/net/eth0/queues/rx-0/rps_cpus   # all cores except the IRQ core
```

The hot core went from 14% to 21% idle. Total CPU on that host rose from about
60% to 67%: steering is not free. It is the right trade only when the host has
spare total CPU and one core is the risk.

None of these survive a reboot on their own. A oneshot systemd unit ordered
after `network-online.target` reapplies the qdisc and RPS settings.

## What is left

1. **Hosting provider:** a host-passthrough CPU model on the entry server would
   give TLS its AES path and remove most of the 8% encryption cost. A modern
   virtio machine type addresses the notification cost.
2. **Next planned restart:** a larger `channel_size`, so a full reconnect does
   not overflow the admission queue.
3. **Capacity:** the entry server is CPU-bound near 850 Mbit/s. More cores or a
   second entry server moves that ceiling; presets do not.

## Sampling scripts

`tools/fieldtuning/bp-sample` prints one line per measurement window: CPU split,
backpack cores, RSS, throughput, connections, sessions, retransmit rate and
cores per Gbit/s. `tools/fieldtuning/bp-sample-cron` appends that line every
five minutes together with queue-full drops and errors from the journal.

Compare cores per Gbit/s rather than CPU percentage. A saturated host stays at
the same percentage after an improvement and carries more traffic instead.

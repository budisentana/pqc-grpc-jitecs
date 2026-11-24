# Communication cost of post-quantum bearer tokens in gRPC metadata

Experiments, data and analysis scripts for the JITECS paper on the communication
cost of post-quantum (ML-DSA) bearer tokens carried in gRPC metadata.

- **E2**: bytes on the wire and latency of bearer tokens in gRPC metadata, on plain loopback
- **E2b**: E2 repeated over a bandwidth-limited link (100 Mbit/s, `tc netem`)

All results in `experiments/e2_metadata/results/` were produced on 7 Nov 2026 on the
machine described below. Every command here is run from the repository root.

## Repository layout

| Path | Contents |
| --- | --- |
| `experiments/e2_metadata/` | E2 and E2b scripts |
| `experiments/e2_metadata/results/` | Raw data, summaries, reports, netem setup and packet capture |
| `experiments/common/cpu_setup.sh` | Fixes the CPU frequency for benchmarking, or restores the defaults (needs sudo) |
| `experiments/common/environment.txt` | Test machine record |
| `protobufs/` | `security_service.proto` and the generated `_pb2` / `_pb2_grpc` modules used by the E2 client and server |

## Test machine

From `experiments/common/environment.txt`:

| | |
| --- | --- |
| CPU | Intel Core i3-12100 (12th Gen), x86_64, 4 physical cores, 8 logical CPUs |
| RAM | 7.0 GiB (4.0 GiB swap) |
| OS | Ubuntu 26.04.1 LTS, kernel 7.0.0-38-generic |
| Python | 3.14.4 |
| gRPC | grpcio 1.84.1, grpcio-tools 1.84.1 |
| PQC library | liboqs-python 0.16.0.1 |

All runs used a fixed CPU frequency (governor `performance`, turbo off; recorded per run in
`*_run_environment.csv`), with the client pinned to core 2 and the server to core 3.

## E2: bearer tokens in gRPC metadata (loopback)

E2 measures what a bearer token in the standard `authorization` header costs a gRPC call,
both in bytes on the wire and in latency. The server (`e2_server.py`) only answers: it has
no interceptor and does not verify signatures, so the cost measured is that of carrying
the token.

- **Part A: real tokens.** `none`, ES256, EdDSA, ML-DSA-44, ML-DSA-65 and ML-DSA-87, each
  signing the same 200-byte claim set.
- **Part B: synthetic token sweep.** Base64url values of 100 to 6,000 bytes (100, 250,
  500, 750, then 1,000 to 6,000 in steps of 250, plus 4,051 and 4,052), to locate where the
  header no longer fits in the default 4,096-byte HPACK dynamic table (entry size =
  name + value + 32 bytes).

Each scenario uses a fresh TCP connection and is measured in two passes:

1. **Byte pass:** 10 calls through a local relay (port 50061) that parses the HTTP/2
   frames and records the HEADERS bytes of every call: the first call of a connection
   and the later calls, which show whether HPACK indexed the header.
2. **Latency pass:** 50 warm-up + 500 measured calls, made directly to the server
   (port 50060).

Scenario order is shuffled in every run. One launch of `e2_benchmark.py` is one run; the
results come from 5 runs (33 scenarios, 82,500 measured calls plus 1,650 byte-pass calls,
0 failures).

Main results (median latency): ML-DSA-65 322.1 us vs ES256 311.3 us (+3.5%). ML-DSA-65
and ML-DSA-87 tokens do not fit in the HPACK table, so they are re-sent in full on every
call (4,766 and 6,524 HEADERS bytes per call vs 434 for ES256). The full tables are in
`results/e2_report.txt`.

## E2b: E2 over a 100 Mbit/s link (tc netem)

E2b repeats E2 unchanged, but gRPC traffic (TCP port 50060, both directions, IPv4 and
IPv6) on loopback is limited to 100 Mbit/s with `tc netem`. No delay is added, so the
extra cost measured is serialization. All other loopback traffic is not affected.
5 runs, fixed CPU frequency, 84,150 calls, 0 failures, 0 netem drops.

Main results (median latency): ML-DSA-65 791.0 us vs ES256 368.7 us (+114.5%), against
+3.5% on plain loopback (E2). The sweep cost is 80-81 ns/byte below 3 KB and above 4 KB,
matching serialization at 100 Mbit/s (8 bits / 100 Mbit/s = 80 ns/byte), with a
transition of about 60 us between 3 and 4 KB. A packet capture of S3000, S3500 and S4000
shows one request segment per call and no extra packets, so the cause of the transition
is left open.

- Network condition: `e2b_netem.sh` (`on [rate] [port]` / `off` / `status`; defaults
  100mbit and port 50060), recorded with its counters in `results/e2b_netem_setup.txt`
- Comparison with E2: `e2b_compare.py` -> `results/e2b_comparison.csv`
- Packet capture: `e2b_capture.sh` -> `results/e2b_segments.pcap`, analysed by
  `e2b_capture_gaps.py` -> `results/e2b_capture_gaps.csv`

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

`liboqs-python` needs the liboqs C library (it builds it on first import if it is
not installed). `e2b_capture_gaps.py` needs `tcpdump`, and the netem setup needs `tc`
(iproute2).

## Rerunning

Two scripts need root: `experiments/common/cpu_setup.sh` (writes to
`/sys/devices/system/cpu`) and `experiments/e2_metadata/e2b_netem.sh` (changes the `lo`
qdisc). `e2b_capture.sh` also calls `sudo` itself for `tcpdump`. These are run by hand
with `sudo`; the benchmark and analysis scripts run as a normal user with the venv active.

Rerunning a benchmark appends to the existing CSV files in `results/`, so move the
published files aside first if you want a clean set.

### E2

```bash
sudo bash experiments/common/cpu_setup.sh            # governor=performance, turbo off

python experiments/e2_metadata/e2_benchmark.py        # one run; repeat 5 times
python experiments/e2_metadata/e2_analyze.py          # -> e2_bytes_summary.csv, e2_latency_summary.csv, e2_report.txt
python experiments/e2_metadata/e2_check_bytes.py      # -> e2_byte_check.txt (byte counts that differ between runs)

sudo bash experiments/common/cpu_setup.sh restore    # governor=powersave, turbo on
```

The E2 server is started and stopped by `e2_benchmark.py`.

### E2b

```bash
sudo bash experiments/common/cpu_setup.sh
sudo bash experiments/e2_metadata/e2b_netem.sh on     # 100 Mbit/s on port 50060
bash experiments/e2_metadata/e2b_run.sh               # 5 runs, analysis and comparison with E2
sudo bash experiments/e2_metadata/e2b_netem.sh off
sudo bash experiments/common/cpu_setup.sh restore
```

`e2b_run.sh` stops if the netem rule is not active. It writes only `results/e2b_*` files
and never touches the E2 files. It runs `e2_benchmark.py` with the output redirected to
`e2b_*.csv`, analyses those with `e2_analyze.py` (outputs saved as `e2b_*_summary.csv`
and `e2b_report.txt`), then runs `e2b_compare.py`.

Packet capture (with the netem rule still on):

```bash
bash experiments/e2_metadata/e2b_capture.sh           # short S3000/S3500/S4000 trial -> /tmp/e2b_seg.pcap
cp /tmp/e2b_seg.pcap experiments/e2_metadata/results/e2b_segments.pcap
python experiments/e2_metadata/e2b_capture_gaps.py    # -> results/e2b_capture_gaps.csv
```

### Analysis only (published data)

These scripts need no root and do not run the benchmark:

```bash
python experiments/e2_metadata/e2_analyze.py
python experiments/e2_metadata/e2_check_bytes.py
python experiments/e2_metadata/e2b_compare.py
python experiments/e2_metadata/e2b_capture_gaps.py
```

## Result files

| File | Contents |
| --- | --- |
| `e2_bytes.csv`, `e2b_bytes.csv` | One row per byte-pass call: HEADERS bytes and frames, DATA bytes, client-to-server bytes |
| `e2_latency.csv`, `e2b_latency.csv` | One row per measured call (latency in ns, gRPC status) |
| `e2_run_environment.csv`, `e2b_run_environment.csv` | Per run: scenario order, CPU governor, turbo, cores, Python/grpcio/liboqs versions, token sizes |
| `e2_bytes_summary.csv`, `e2b_bytes_summary.csv` | Bytes per scenario, first call and later calls |
| `e2_latency_summary.csv`, `e2b_latency_summary.csv` | Median, mean, p95, p99, mean of run means with 95% CI, call rate |
| `e2_report.txt`, `e2b_report.txt` | Text summary of the above |
| `e2_byte_check.txt` | Byte counts that are not identical across the 5 E2 runs |
| `e2b_comparison.csv` | E2b vs E2 median latency per scenario |
| `e2b_netem_setup.txt` | qdisc and filters during E2b, with counters after the 5 runs |
| `e2b_segments.pcap`, `e2b_capture_gaps.csv` | Packet capture of S3000/S3500/S4000 and its per-call client/server gaps |

`experiments/e2_metadata/config.txt` is the original E2 planning note. The parameters
actually used are the ones in `e2_benchmark.py` and described above.

## License

Apache License 2.0, see `LICENSE`.

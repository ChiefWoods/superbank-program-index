# Superbank Program Index

Assignment requirements:
- stand up a local topology
- backfill a bounded devnet slice
- load-test `getSignaturesForAddress` and `getSignatureStatuses` using k6 suites
- improve access pattern for viewing program activity

## Assignment Report

Source repo: `[solana-rpc/superbank](https://github.com/solana-rpc/superbank)`, inspected from the local checkout at `/Users/chiiyuen/solana-rpc/superbank` on `main`.

### Devnet Ingestion

Devnet ingestion requires a recent bounded range. Old devnet slots can be unavailable because devnet has been restarted several times and public RPC history is limited.

For the assignment, the devnet slot 116114407 is closen, with the starting slot being 1000 slots behind this slot (116113407.

```bash
SUPERBANK_SOURCE=rpc \
RPC_URL=https://api.devnet.solana.com \
RPC_FROM_SLOT=116113407 \
RPC_SLOT_COUNT=1000 \
RPC_MAX_SUPPORTED_TX_VERSION=0 \
RPC_FLUSH_EVERY_SLOTS=25 \
CLICKHOUSE_URL=http://localhost:8123 \
CLICKHOUSE_DATABASE=default \
cargo run -p superbank --
```

### RPC And k6 Results

After ingestion, `superbank-rpc` was run locally on `http://localhost:8899`, and k6 pools were generated from the ingested data instead of using stale checked-in samples:

```bash
docker exec -i clickhouse clickhouse-client --query "
SELECT DISTINCT base58Encode(address)
FROM default.gsfa
LIMIT 1000" > tests/k6/data/pools/devnet-addresses.txt

docker exec -i clickhouse clickhouse-client --query "
SELECT DISTINCT base58Encode(signature)
FROM default.signatures
LIMIT 1000" > tests/k6/data/pools/devnet-signatures.txt
```

Saved result artifacts are under `results/`: six JSON run files plus `summary.txt`.

| Method                    | Scenario | Requests | Failures | Avg latency | p95 latency | Max latency | Artifact |
| ------------------------- | -------- | -------- | -------- | ----------- | ----------- | ----------- | -------- |
| `getSignaturesForAddress` | basic    | 21,414   | 0        | 6.80 ms     | 11.55 ms    | 243.73 ms   | `gsfa-basic-2026-07-02T05-55-11Z.json` |
| `getSignaturesForAddress` | soak     | 419,543  | 0        | 28.32 ms    | 62.34 ms    | 1549.18 ms  | `gsfa-soak-2026-07-02T05-54-09Z.json` |
| `getSignaturesForAddress` | stress   | 78,253   | 0        | 191.47 ms   | 467.35 ms   | 1183.60 ms  | `gsfa-stress-2026-07-02T06-08-57-100Z.json` |
| `getSignaturesForAddress` | spike    | 26,027   | 0        | 233.18 ms   | 675.27 ms   | 1407.32 ms  | `gsfa-spike-2026-07-02T06-10-30-595Z.json` |
| `getSignatureStatuses`    | basic    | 14,722   | 0        | 9.91 ms     | 16.34 ms    | 152.47 ms   | `sigstatus-basic-2026-07-02T06-11-11-456Z.json` |
| `getSignatureStatuses`    | stress   | 68,564   | 0        | 218.54 ms   | 534.44 ms   | 1653.35 ms  | `sigstatus-stress-2026-07-02T06-14-16-208Z.json` |

The GSFA basic run returned an average of `8.59` signatures per request with a configured limit of `25`. The status basic run returned an average of `9.96` rows per request with a batch size of `10`. All saved k6 summaries reported `0` HTTP errors, `0` RPC errors, and `0` timeouts.

### k6 Findings And Analysis

The k6 results show that both tested RPC paths were functionally stable against the local Superbank RPC server and ClickHouse backend. Across the six saved JSON summaries, all `628,523` requests completed successfully with no recorded HTTP errors, RPC errors, or timeouts.

GSFA showed strong steady-state behavior in the basic run. It completed `21,414` requests with `6.80 ms` average latency, `11.55 ms` p95 latency, and a `243.73 ms` max latency.

The GSFA soak result is the strongest stability signal. Over `10m` with `20` VUs, it handled `419,543` successful requests with `28.32 ms` average latency and `62.34 ms` p95 latency. That is consistent with the `default.gsfa` materialized view doing the intended job: address-keyed lookups stay fast when requests hit known ingested addresses from the generated devnet pool.

Under GSFA stress and spike traffic, tail latency increased but failures stayed at zero. Stress reached `191.47 ms` average latency and `467.35 ms` p95. Spike reached `233.18 ms` average latency and `675.27 ms` p95. The max latency stayed around `1.2s` to `1.4s`. This looks like queueing under burst/concurrency pressure rather than correctness failure, because the error counters stayed at zero and the average rows returned stayed stable at roughly `8.6` signatures per request.

`getSignatureStatuses` also behaved correctly for known-present signatures. The basic run completed `14,722` requests with `9.91 ms` average latency and `16.34 ms` p95 latency. The stress run completed `68,564` requests with `218.54 ms` average latency and `534.44 ms` p95 latency. The average returned rows stayed at roughly `9.95` per request with a batch size of `10`, which means the generated signature pool was well matched to the ingested data.

Comparing methods, the basic runs are close: GSFA averaged `6.80 ms`, while `getSignatureStatuses` averaged `9.91 ms`. Under stress, both methods moved into the same latency band: GSFA averaged `191.47 ms`, and `getSignatureStatuses` averaged `218.54 ms`. That is expected because both are backed by purpose-built views, but `getSignatureStatuses` performs batched signature lookups and returns status data for up to `10` signatures per request.

The saved summaries have two measurement limitations. First, the generated JSON reports `p99: 0`, so p99 should not be used for analysis from these artifacts. The report should rely on average, p95, and max latency instead. Second, CPU, memory, and ClickHouse server-side timing were not captured in these JSON summaries, so the results prove local functional stability and request latency, but not full system capacity. See `results/summary.txt` for a brief rollup of all runs.

### Conclusion

Superbank works locally for a bounded devnet slice, but it requires extra devnet-specific setup: choose a recent finalized slot range instead of assuming old slots are available. The k6 results show that GSFA and signature-status queries are stable for known ingested devnet data.
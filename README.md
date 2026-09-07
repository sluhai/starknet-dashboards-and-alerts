# Starknet validator dashboards and alerts

Grafana dashboards and alert rules for a Starknet validator, built on the metrics from
[eqlabs/starknet-validator-attestation](https://github.com/eqlabs/starknet-validator-attestation)
and the [pathfinder](https://github.com/eqlabs/pathfinder) node behind it.

| File | What it is |
| --- | --- |
| `starknet-attestation-dashboard.json` | Attestation dashboard. Seven rows, only the top one open. |
| `starknet-node-dashboard.json` | Pathfinder node health. Five rows, only `Status` open. |
| `attestation-monitoring.yaml` | 22 alert rules, Grafana provisioning format. |

## Loading a dashboard

These are Grafana 13 schema v2 files. Neither carries a dashboard identifier, so the same file works
whether you are creating a dashboard or replacing one.

To create one: `+` → `New dashboard` → open the toolbar on the right edge → the `{}` icon → select all the
text in the editor and paste the file over it → `Apply changes` → **`Save`**. Grafana assigns the
identifier itself.

To replace one you already have: open it → `Edit` → the same `{}` icon → select all → paste → `Apply
changes` → **`Save`**. The dashboard keeps its identifier and its address.

The title comes from the file. Grafana refuses to create a second dashboard with a title that already
exists in the same folder, so rename one of them if that happens.

You do not have to repoint anything afterwards. Neither dashboard hardcodes a data source: each has a
`Data source` selector at the top and every panel reads it from there. Grafana picks a Prometheus for you
on load; if you have more than one, switch it in that selector.

## The Network selector

Three choices: `Mainnet`, `Sepolia`, `Both`, and `Both` is the default.

Each value is a regular expression, because the metrics name the network three different ways. The
attestation tool uses the label `exported_network` with `SN_MAIN` and `SN_SEPOLIA`; pathfinder uses
`network` with `mainnet` and `testnet-sepolia`; `up` and `process_start_time_seconds` carry no network
label at all, and there the network is the scrape port inside `instance`, `:9000` for mainnet and `:9001`
for testnet. One value covers all of them: `Sepolia` is `SN_SEPOLIA|testnet-sepolia` in the attestation
dashboard and `testnet-sepolia|.*9001` in the node dashboard.

To add a third network, edit the variable by hand — it is a fixed list, not a query.

## Prometheus jobs

Two scrape jobs. The relabelling in the first one is what creates the `exported_network` label.

```yaml
- job_name: "starknet-attestation"
  static_configs:
    - targets: ['localhost:9095', 'localhost:9096']
  relabel_configs:
    - source_labels: [__address__]
      regex: localhost:9095
      target_label: exported_network
      replacement: SN_MAIN
    - source_labels: [__address__]
      regex: localhost:9096
      target_label: exported_network
      replacement: SN_SEPOLIA

- job_name: "pathfinder"
  static_configs:
    - targets: ['localhost:9000', 'localhost:9001']
```

Run the attestation tool with `--metrics-address 127.0.0.1:9095` for mainnet and `:9096` for testnet, and
pathfinder with `--monitor-address 0.0.0.0:9000`, mapped to host port 9000 for mainnet and 9001 for
testnet. Reload Prometheus afterwards.

## Loading the alert rules

`Alerting → Import alert rules` will refuse this file: that page takes Prometheus rule format, this is
Grafana provisioning format. Two ways that work:

- put the file in `/etc/grafana/provisioning/alerting/` and restart Grafana. The rules load but become
  read-only in the interface;
- or POST it to the provisioning API with the header `X-Disable-Provenance: true`. The rules stay editable.

The export leaves out two things, and without them the rules load but do nothing: the contact point, which
the rules call by name (`grafana-default-email`), and the data source. Unlike the dashboards, the rules
name Prometheus directly, so each one needs repointing if yours has a different identifier.

## Worth knowing

Node dashboard: three panels use `rpc_websocket_connections`, `rpc_websocket_connections_closed_total` and
`rpc_websocket_connections_rejected_total`, which exist only in pathfinder 0.24.0 and newer; on older
versions they read `No data`. One of them, `Websocket closes and rejects, last 24h`, is also left out of
the `Network` filter: those two counters appear only at the first close and the first refusal, so their
labels cannot be counted on, and filtering could hide the rare event the panel is there for.

Attestation dashboard: `Success rate, 30 days`, `Success rate, 90 days` and `Missed epochs, 90 days` need
that much history to mean what they say. Grafana Cloud on the free plan keeps 14 days — the two rates then
stay correct as ratios but cover less time than their titles claim, and the missed-epoch count comes out
short. Delete them there, or read them knowing that. `Success rate, 14 days` is in the row for exactly that
case: on a store that keeps a fortnight it is the longest window that still means what it says.

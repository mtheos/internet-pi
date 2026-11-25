# Internet Monitor on Kubernetes

This Kustomize package owns the Kubernetes service definitions, pinned images, Prometheus scrape defaults, and the adapted Grafana dashboard. HTTPS probes check three external endpoints over IPv4 every 30 seconds. Speedtest Exporter measures WAN download, upload, and latency once per hour. Each speed test consumes bandwidth. The exporter passes Ookla's license and GDPR acceptance flags automatically. Review the [Speedtest CLI terms and privacy links](https://www.speedtest.net/apps/cli) before installing.

## Standalone install

Run from this repository's root with kubectl configured for the intended cluster. The cluster needs working pod DNS, outbound HTTPS and speed-test connectivity, and a default StorageClass with writable volumes for non-root workloads. The package creates the `internet-monitor` namespace and requests 5 GiB for Prometheus and 1 GiB for Grafana. Prometheus retains up to 30 days and targets at most 3 GB of retained blocks.

Store an initial Grafana password in a private file outside the checkout, with mode 0600 and no trailing newline. Set `GRAFANA_PASSWORD_FILE` to that file's path before running the commands.

```sh
kubectl kustomize kubernetes
kubectl apply -f kubernetes/namespace.yaml
kubectl -n internet-monitor create secret generic grafana-admin \
  --from-file=password="${GRAFANA_PASSWORD_FILE:?set the path to the password file}"
kubectl apply -k kubernetes
kubectl -n internet-monitor rollout status deployment/prometheus
kubectl -n internet-monitor rollout status deployment/grafana
kubectl -n internet-monitor port-forward service/grafana 3000:3000
```

Open `http://localhost:3000` and sign in as `admin`. Open **Internet Monitor** and check the three HTTPS series. Allow an hour for the first bandwidth sample. In Grafana Explore, `probe_success` and `speedtest_up` show the probe and speed-test results.

The Secret initializes a fresh Grafana database. Rotate an existing account in Grafana and update the Secret for future restores. Back up both data volumes if you need dashboard changes or metrics history to survive storage loss.

## Use as a base

Reference `kubernetes/` at a full commit SHA and patch scheduling, storage, and other site settings in your overlay. Keep one deployment owner per namespace. To customize the default probe targets or cadence, update `prometheus.yml` in this repo, or explicitly replace its ConfigMap key in a site overlay. The Ansible `config.yml` configures the separate Raspberry Pi Compose installation.

# k8s_crd

A CoreDNS plugin that is very similar to [k8s_external](https://coredns.io/plugins/k8s_external/) but supporting DNSEndpoint external resource.

**This project is a modification of [k8s_gateway](https://github.com/ori-edge/k8s_gateway) plugin, adopted with DNSEndpoint client.**

This plugin relies on it's own connection to the k8s API server and doesn't share any code with the existing [kubernetes](https://coredns.io/plugins/kubernetes/) plugin. The assumption is that this plugin can now be deployed as a separate instance (alongside the internal kube-dns) and act as a single external DNS interface into your Kubernetes cluster(s).

## Description

`k8s_crd` resolves Kubernetes resources with their external IP addresses based on zones specified in the configuration. This plugin will resolve the following type of resources:

| Kind | Matching Against | External IPs are from |
| ---- | ---------------- | -------- |
| DNSEndpoint | all FQDNs from `spec.endpoints.dnszone` matching configured zones | `.spec.endpoints.dnszone.targets` |

Currently only supports A-type queries, all other queries result in NODATA responses.

This plugin is **NOT** supposed to be used for intra-cluster DNS resolution and does not contain the default upstream [kubernetes](https://coredns.io/plugins/kubernetes/) plugin.

## Install

The recommended installation method is using the helm chart provided in the repo:

```shell
helm install exdns ./charts/coredns
```

## Configure

```text
k8s_crd [ZONE...]
```

Optionally, you can specify what kind of resources to watch, default TTL to return in response and a default name to use for zone apex, e.g.

```text
k8s_crd example.com {
    ttl 10
    apex dns1
}
```

## Resolving order

### GeoIP

In case dnsEndpoint object's target has a label of `strategy: geoip` CoreDNS `k8s_crd` plugin will respond in a special way:

* Assuming record has multiple IPs associated with it, and DNS message comes with edns0 `CLIENT-SUBNET` option.
* CoreDNS will compare the specified field tag(s) for IP extracted from `CLIENT-SUBNET` option against available Endpoint.Targets
* Return only IPs where tags match
* If IP has no common tag, all entries are returned.
* CoreDNS must be supplied with a specially crafted GeoIP database in MaxMind DB format and mounted (at `/geoip.mmdb` by default, configured via the `geodatafilepath` plugin option). Refer to [./terratest/geogen](./terratest/geogen) for examples. Using the MaxMind GeoLite2 database is supported using the necessary `geodatafield` to configure the field to use as required.

#### Single Field Matching

For simple GeoIP matching using a single field:

```text
k8s_crd example.com {
    geodatafilepath /geoip.mmdb
    geodatafield country.iso_code
    ...
}
```

#### Hierarchical Multi-Level Matching

For hierarchical matching with multiple fields in priority order, use `geodatafields` (plural). The plugin will try each field in order and return matches from the first field that produces results:

```text
k8s_crd example.com {
    geodatafilepath /geoip.mmdb
    geodatafields country.iso_code continent.code
    ...
}
```

**Note:** If both `geodatafield` and `geodatafields` are specified, `geodatafields` takes precedence. For backward compatibility, `geodatafield` (singular) is still supported.

### Weight Round Robin

To enable the weight round robin you have to set the configuration to weight load-balancer:

```text
k8s_crd example.com {
    loadbalance weight
    ...
}
```

The dnsEndpoint must also contain information about the percentage distribution per region
and their IP addresses. Thanks to this, the weight round-robin module will know in which
order to return IP addresses. Addresses with high probability will often be at the top of
DNS responses, while those with low probability will be at the bottom.

```yaml
labels:
    strategy: roundrobin
    weight-eu-0-50: 10.0.0.1
    weight-eu-1-50: 10.0.0.2
    weight-za-0-0:  10.10.0.1
    weight-us-0-50: 10.20.0.1
```

For more information about balancing, please visit our [go-weight-shuffling](https://github.com/k8gb-io/go-weight-shuffling
) module.

### Reactive cross-cluster resolution

In a k8gb setup where several clusters are authoritative for the **same**
delegated zone (so an app can migrate freely between clusters), a recursive
resolver may send a query to a cluster that does not host the requested app. By
default that cluster answers an authoritative `NXDOMAIN` and resolution fails.

The `reactive` option makes `k8s_crd` resolve such local misses live from the
peer clusters instead of returning `NXDOMAIN`:

```text
k8s_crd example.com {
    reactive on
    reactiveself eu          # geotag identifying THIS cluster, matched as an
                             # exact dash-delimited segment of the peer NS names
                             # (recommended, so the local cluster is not queried)
    reactiveport 53          # port the peer CoreDNS servers listen on (default 53)
    reactivetimeout 2s       # per-peer query timeout (default 2s)
    reactivettl 30           # TTL of the synthesized A records (default 30)
    reactivecachettl 15s     # positive cache TTL (default 15s)
    reactivenegttl 5s        # negative cache TTL (default 5s)
    ...
}
```

How it works:

* It only runs on a **local miss** (the gateway produced no answer) for `A`
  queries, so local hits are never affected.
* Peer CoreDNS servers are discovered from the k8gb `ZoneDelegation` custom
  resource (`status.dnsServers`). The zone a host belongs to is matched by DNS
  suffix, and the local cluster is excluded via the `reactiveself` geotag.
* Peers are queried for the full `<host>` record and the **first non-empty
  answer wins** (early return). Because every authoritative peer already serves
  the fully-aggregated, load-balanced target set, one peer that has the GSLB is
  enough — there is no need to fan out to all of them. Querying `<host>` (rather
  than the cluster-local `localtargets-<host>`) is what makes a single peer's
  answer complete. Peers are shuffled to spread the load across the fleet, so
  this scales to many clusters without querying every one on each miss.
* Cross-cluster loops are prevented with a marker: reactive probes are sent with
  the DNS `CheckingDisabled` (CD) bit set, and the plugin never reactivates a
  query that already carries CD. So if two clusters both miss the host they do
  not query each other forever — the probe short-circuits to `NXDOMAIN`.
* Results are cached (positive and negative) and concurrent lookups for the
  same host are coalesced.
* While the `ZoneDelegation` cache is still warming up the plugin returns
  `SERVFAIL` (not `NXDOMAIN`) so recursive resolvers fail over to a sibling
  cluster's name server instead of caching a false negative.

## Build

### Container image

The `Makefile` compiles the CoreDNS binary and builds the container image
defined in the `Dockerfile`:

```shell
# build binary + image (uses docker by default)
make image REGISTRY=<your-registry> BIN=k8s_crd TAG=<tag>
```

To build manually (e.g. with podman, or to control the target platform):

```shell
# 1. compile the CoreDNS binary for the target platform (linux/amd64)
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o coredns cmd/coredns.go

# 2. build the image (the Dockerfile just copies the ./coredns binary)
podman build --platform linux/amd64 -t <registry>/k8s_crd:<tag> .
#   or: docker build --platform linux/amd64 -t <registry>/k8s_crd:<tag> .

# 3. push it to your registry
podman push <registry>/k8s_crd:<tag>

# verify the plugin is compiled in
podman run --rm --platform linux/amd64 <registry>/k8s_crd:<tag> -plugins | grep k8s_crd
```

### With compile-time configuration file

```shell
git clone https://github.com/coredns/coredns
cd coredns
vim plugin.cfg
# Replace lines with kubernetes and k8s_external with k8s_crd:github.com/k8gb-io/coredns-crd-plugin
go generate
go build
./coredns -plugins | grep k8s_crd
```

### With external golang source code

```shell
git clone https://github.com/k8gb-io/coredns-crd-plugin.git
cd coredns-crd-plugin
go build cmd/coredns.go
./coredns -plugins | grep k8s_crd
```

For more details refer to [this CoreDNS doc](https://coredns.io/2017/07/25/compile-time-enabling-or-disabling-plugins/)

## Notes regarding Zone Apex and NS server resolution

Due to the fact that there is not nice way to discover NS server's own IP to respond to A queries, as a workaround, it's possible to pass the name of the LoadBalancer service used to expose the CoreDNS instance as an environment variable `EXTERNAL_SVC`. If not set, the default fallback value of `external-dns.kube-system` will be used to look up the external IP of the CoreDNS service.

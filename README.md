# ip-masq-agent

The ip-masq-agent configures `iptables` rules to `MASQUERADE` traffic outside link-local (optional, enabled by default) and additional arbitrary IP ranges.

It creates an `iptables` chain called `IP-MASQ-AGENT`, which contains match rules for link local (`169.254.0.0/16`) and each of the user-specified IP ranges. It also creates a rule in `POSTROUTING` that jumps to this chain for any traffic not bound for a `LOCAL` destination.

IPs that match the rules (except for the final rule) in `IP-MASQ-AGENT` are *not* subject to `MASQUERADE` via the `IP-MASQ-AGENT` chain (they `RETURN` early from the chain). The final rule in the `IP-MASQ-AGENT` chain will `MASQUERADE` any non-`LOCAL` traffic.

`RETURN` in `IP-MASQ-AGENT` resumes rule processing at the next rule the calling chain, `POSTROUTING`. Take care to avoid creating additional rules in `POSTROUTING` that cause packets bound for your configured ranges to undergo `MASQUERADE`.

## Launching the agent as a DaemonSet
This repo includes an example yaml file that can be used to launch the ip-masq-agent as a DaemonSet in a Kubernetes cluster.

```
kubectl create -f ip-masq-agent.yaml
```

The spec in `ip-masq-agent.yaml` specifies the `kube-system` namespace for the DaemonSet Pods.

## Configuring the agent

Important: You should not attempt to run this agent in a cluster where the Kubelet is also configuring a non-masquerade CIDR. You can pass `--non-masquerade-cidr=0.0.0.0/0` to the Kubelet to nullify its rule, which will prevent the Kubelet from interfering with this agent.

By default, the agent is configured to treat the three private IP ranges specified by [RFC 1918](https://tools.ietf.org/html/rfc1918) as non-masquerade CIDRs. These ranges are `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`. To change this behavior, see the flags section below. The agent will also treat link-local (`169.254.0.0/16`) as a non-masquerade CIDR by default.

By default, the agent is configured to reload its configuration from the `/etc/config/ip-masq-agent` file in its container every 60 seconds.

The agent configuration file should be written in yaml or json syntax, and may contain four optional keys:
- `nonMasqueradeCIDRs []string`: A list strings in CIDR notation that specify the non-masquerade ranges.
- `cidrLimit int`: Maximum number of nonMasqueradeCIDRs entries allowed. 64 by default.
- `masqLinkLocal bool`: Whether to masquerade traffic to `169.254.0.0/16`. False by default.
- `masqLinkLocalIPv6 bool`: Whether to masquerade traffic to `fe80::/10`. False by default.
- `resyncInterval string`: The interval at which the agent attempts to reload config from disk. The syntax is any format accepted by Go's [time.ParseDuration](https://golang.org/pkg/time/#ParseDuration) function.

The agent will look for a config file in its container at `/etc/config/ip-masq-agent`. This file can be provided via a `ConfigMap`, plumbed into the container via a `ConfigMapVolumeSource`. As a result, the agent can be reconfigured in a live cluster by creating or editing this `ConfigMap`.

This repo includes a directory-representation of a `ConfigMap` that can configure the agent (the `agent-config` directory). To use this directory to create the `ConfigMap` in your cluster:

```
kubectl create configmap ip-masq-agent --from-file=agent-config --namespace=kube-system
```

Note that we created the `ConfigMap` in the same namespace as the DaemonSet Pods, and named the `ConfigMap` to match the spec in `ip-masq-agent.yaml`. This is necessary for the `ConfigMap` to appear in the Pods' filesystems.

### Agent Flags

The agent accepts five flags, which may be specified in the yaml file.

`masq-chain`
:  The name of the `iptables` chain to use. By default set to `IP-MASQ-AGENT`.

`nomasq-all-reserved-ranges`
:  Whether or not to masquerade all RFC reserved ranges when the configmap is empty. The default is `false`. When `false`, the agent will masquerade to every destination except the ranges reserved by RFC 1918 (namely `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`). When `true`, the agent will masquerade to every destination that is not marked reserved by an RFC. The full list of ranges is (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10`, `192.0.0.0/24`, `192.0.2.0/24`, `192.88.99.0/24`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`, and `240.0.0.0/4`). Note however, that this list of ranges is overridden by specifying the nonMasqueradeCIDRs key in the agent configmap.

`enable-ipv6`
: Whether to configurate ip6tables rules. By default `enable-ipv6` is false.

`random-fully`
: Since ip-masq-agent v2.10, `--random-fully` started to be set by default on the MASQUERADE rule generated (by defaulting this flag to `true`) to avoid a Linux kernel racing issue. This can cause the source port used on the node to be always different from the source port used on the pod. Set this flag to `false` to restore the previous behavior.

`to-ports`
: MASQUERADE rules of iptables can select any port between 1024 and 65535 inclusively by default. This flag adds additional MASQUERADE rules for TCP, UDP and SCTP traffic to specify explicit source ports to be used (traffic in other protocols is unchanged). Ranges can be specified using `1024-29999` syntax, or multiple ranges with `1024-29999,32768-65535` where the traffic is balanced among all ports within the ranges.

## Deploying on OKE with the Native Pod Network (NPN)

This branch adds an enhancement for Oracle Kubernetes Engine (OKE) clusters that
use the **Native Pod Network** (NPN, a.k.a. the VCN-native CNI). On NPN each pod
gets a VCN IP and egress is routed by the CNI via per-pod policy routing rules
(`ip rule`) rather than the host's `main` table. The stock `IP-MASQ-AGENT`
iptables chain alone is therefore not sufficient to control which destinations
get masqueraded on NPN nodes.

In addition to the iptables rules, the agent maintains a **policy routing rule**
per configured IPv4 `nonMasqueradeCIDR`:

```
20: not from all to <cidr> lookup main
```

A packet *not* destined for the configured CIDR is sent to the `main` routing
table (where it is subject to the `IP-MASQ-AGENT` MASQUERADE rule), while a
packet destined for the configured CIDR falls through to the NPN-managed rules
and is delivered without masquerade. Priority `20` is intentionally lower than
the NPN per-pod rules (which live at priority `512` and `1536`), so it is
consulted first and never collides with the CNI.

**Limitations of this mode:**
- Only a **single** IPv4 CIDR is effective in `nonMasqueradeCIDRs`. This is
  inherent to the `not to <cidr> lookup main` semantics (the complement match
  covers "everything else"), so configure exactly one CIDR.
- IPv6 CIDRs are skipped by the ip-rule path (IPv6 still uses the iptables
  `IP-MASQ-AGENT` chain only).

### Quick start on OKE NPN

The `oke/` directory holds ready-to-apply manifests. The image is
`docker.io/robocap/ip-masq-agent:v2` (an Alpine-based image that bundles
`iproute2` and `iptables`, required because `ip rule` is used in addition to
iptables). The DaemonSet runs with `hostNetwork: true` and `NET_ADMIN`/`NET_RAW`
capabilities, and tolerates all taints so it lands on every node.

1. Create the ConfigMap. Set `nonMasqueradeCIDRs` to the **single** CIDR you do
   not want masqueraded (typically your in-cluster pod/service range):

   ```yaml
   # oke/config.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: ip-masq-agent
     namespace: kube-system
   data:
     config: |
       nonMasqueradeCIDRs:
         - 10.30.0.0/16
       resyncInterval: 10s
       masqLinkLocal: false
   ```

   ```
   kubectl apply -f oke/config.yaml
   ```

2. Deploy the DaemonSet:

   ```
   kubectl apply -f oke/ip-masq-agent.yaml
   ```

3. Verify the policy rule and the iptables chain are present on a node:

   ```
   $ kubectl exec -n kube-system ds/ip-masq-agent -- ip rule show prio 20
   20:     not from all to 10.30.0.0/16 lookup main

   $ kubectl exec -n kube-system ds/ip-masq-agent -- iptables -t nat -S IP-MASQ-AGENT
   -N IP-MASQ-AGENT
   -A IP-MASQ-AGENT -d 169.254.0.0/16 ... -j RETURN
   -A IP-MASQ-AGENT -d 10.30.0.0/16 ... -j RETURN
   -A IP-MASQ-AGENT ... -j MASQUERADE --random-fully
   ```

The agent reconciles both the policy rules and the iptables chain every
`resyncInterval`. If you edit the ConfigMap and change the CIDR, the previous
`ip rule` is removed and the new one is added on the next sync; the stale rule
is deleted with properly tokenized argv (`ip rule del prio 20 not to <cidr>
lookup main`).

## Rationale
(from the [incubator proposal](https://gist.github.com/mtaufen/253309166e7d5aa9e9b560600a438447))

This agent solves the problem of configuring the CIDR ranges for non-masquerade in a cluster (via iptables rules). Today, this is accomplished by passing a `--non-masquerade-cidr` flag to the Kubelet, which only allows one CIDR to be configured as non-masquerade. [RFC 1918](https://tools.ietf.org/html/rfc1918), however, defines three ranges (`10/8`, `172.16/12`, `192.168/16`) for the private IP address space.

Some users will want to communicate between these ranges without masquerade - for instance, if an organization's existing network uses the `10/8` range, they may wish to run their cluster and `Pod`s in `192.168/16` to avoid IP conflicts. They will also want these `Pod`s to be able to communicate efficiently (no masquerade) with each-other *and* with their existing network resources in `10/8`. This requires that every node in their cluster skips masquerade for both ranges.

We are trying to eliminate networking code from the Kubelet, so rather than extend the Kubelet to accept multiple CIDRs, ip-masq-agent allows you to run a DaemonSet that configures a list of CIDRs as non-masquerade.

## Releasing

See [RELEASE](RELEASE.md).

## Developing

Clone the repo to `$GOPATH/src/k8s.io/ip-masq-agent`.

The build tooling is based on [thockin/go-build-template](https://github.com/thockin/go-build-template).

Run `make` or `make build` to compile the ip-masq-agent.  This will use a Docker image
to build the agent, with the current directory volume-mounted into place.  This
will store incremental state for the fastest possible build.  Run `make
all-build` to build for all architectures.

Run `make test` to run the unit tests.

Run `make container` to build the container image.  It will calculate the image
tag based on the most recent git tag, and whether the repo is "dirty" since
that tag (see `make version`).  Run `make all-container` to build containers
for all architectures.

Run `make push` to push the container image to `REGISTRY`.  Run `make all-push`
to push the container images for all architectures.

Run `make clean` to clean up.

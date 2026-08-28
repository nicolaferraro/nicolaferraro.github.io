---
title: 'kube-proxy, Love It or Hate It'
published: 2026-08-29
description: 'nftables was supposed to be the mode that finally made kube-proxy scale. Then I watched nodes start dying exactly one hour after they joined the cluster.'
tags: ['Kubernetes', 'Networking', 'nftables', 'Linux', 'Cloud Native']
draft: true
---

A node joined the cluster. Fifty-nine minutes and thirty-six seconds later it
went `NotReady`. A second died at 59m47s. A third at 59m57s.

I used to like kube-proxy, in the way you like anything that never asks for
your attention. That is over now.

## The setup

EKS on AWS, Kubernetes v1.35, Ubuntu Noble, kube-proxy in `nftables` mode,
IPv6-only, and a lot of Services.

I picked nftables because by 1.35 it is the only mode with a future. iptables
is O(n) and rewrites the whole table on every update, and IPVS is deprecated
and gets deleted in 1.43. nftables went GA in 1.33, and the [announcement
post](https://kubernetes.io/blog/2025/02/28/nftables-kube-proxy/) promises
constant-time packets and far faster updates.

All of that is true. The nodes died anyway.

## What actually happens

nftables commits are atomic, and atomic means serialized. Every commit takes
one global lock, `commit_mutex`, and holds it until the transaction finishes.

```text
kubelet     waits on RTNL, listing interfaces for its node status
  aws-cni   holds RTNL, waits on the nftables commit mutex
    nft     holds the commit mutex, deep inside nf_tables_bind_set
```

RTNL is the one big lock the kernel puts around its network interface and
routing tables. The CNI takes it because moving a veth between namespaces is
an RTNL operation, and that same call fires `nf_tables_netdev_event`, which
needs `commit_mutex`. So the CNI sits on RTNL waiting for a lock kube-proxy
is holding, and kubelet queues behind it just to list interfaces.

kubelet never touches netfilter. It shares one lock with something that does,
and that is enough to kill it.

## Why exactly one hour

```go title="kubernetes/pkg/proxy/util/utils.go"
// FullSyncPeriod is iptables and nftables proxier full sync period
FullSyncPeriod = 1 * time.Hour
```

A compile-time constant. Nothing changes it, and the clock starts at node
join, which is why the deaths line up with node age.

On one node that full sync took **16 minutes**, all of it holding
`commit_mutex`.

More cores do not help, because the sync loop is single-threaded: 32x the
cores bought 1.29x. `mode: iptables` does not help either, because on Noble
the `iptables` binary is `iptables-nft` and takes the same lock.

## Where I disagree with everyone

Every comparison of kube-proxy modes ranks them by how fast a packet moves.
Ask instead what kube-proxy holds, and for how long, while it applies a
change, and the order flips.

IPVS adds one row to a hash table per Service, with no transaction and no
global lock. Its known problem is burning 65% of a core in userspace, and
that is CPU you can simply buy. nftables holds a global lock for a whole
transaction, and kube-proxy fills that transaction with the entire ruleset
once an hour.

KEP-5495 deprecates IPVS for lack of maintainers. It never mentions that IPVS
is the only mode that updates incrementally.

nftables scales fine. The problem is rewriting a whole cluster's ruleset
inside one globally locked transaction, on a timer nobody can change.

## Love it or hate it

One binary, one DaemonSet, almost nothing to configure, which is exactly why
nobody looks at it until nodes start dying on a schedule.

I still cannot decide whether I like it. But I have stopped assuming it is
boring.

---
title: 'kube-proxy, Love It or Hate It'
published: 2026-08-29
description: 'nftables was supposed to be the mode that finally made kube-proxy scale. Then I watched nodes start dying exactly one hour after they joined the cluster.'
tags: ['Kubernetes', 'Networking', 'nftables', 'Linux', 'Cloud Native']
---

A node joined the cluster. Fifty-nine minutes and thirty-six seconds later it
went `NotReady`. A second died at 59m47s. A third at 59m57s.

I used to like kube-proxy, in the way you like anything that never asks for
your attention. I have learned a lot more about it since.

## The setup

EKS on AWS, Kubernetes v1.35, Ubuntu Noble, kube-proxy in `nftables` mode,
IPv6-only, and a lot of Services.

I picked nftables because by now it is the only mode with a future. iptables
is O(n) and rewrites the whole table on every update, and IPVS is deprecated
and gets deleted in 1.43. nftables went GA in 1.33, and the [announcement
post](https://kubernetes.io/blog/2025/02/28/nftables-kube-proxy/) promises
constant-time packets and far faster updates.

All of that is true. The nodes died anyway.

## What actually happens

nftables commits are atomic, and atomic means serialized. Every commit takes
one global lock, `commit_mutex`, and holds it until the transaction finishes.

```text
kubelet     blocked on RTNL, doing its own network bookkeeping
  aws-cni   holds RTNL, waits on the nftables commit mutex
    nft     holds the commit mutex, deep inside nf_tables_bind_set
```

RTNL is the one big lock the kernel puts around its network interface and
routing tables. The CNI takes it because moving a veth between namespaces is
an RTNL operation, and that same call fires `nf_tables_netdev_event`, which
needs `commit_mutex`. That is what a stalled node showed: the CNI sitting on
RTNL waiting for a lock kube-proxy held, and kubelet stuck behind it.

kubelet never touches netfilter. It shares one lock with something that does,
and that is enough to kill it.

## Why exactly one hour

```go title="pkg/proxy/util/utils.go"
// FullSyncPeriod is iptables and nftables proxier full sync period
FullSyncPeriod = 1 * time.Hour
```

That is
[`pkg/proxy/util/utils.go:41`](https://github.com/kubernetes/kubernetes/blob/v1.35.0/pkg/proxy/util/utils.go#L40-L41)
in v1.35.0, and it is read once per sync in
[`proxier.go:1122`](https://github.com/kubernetes/kubernetes/blob/v1.35.0/pkg/proxy/nftables/proxier.go#L1122).

A compile-time constant, so no configuration flag changes it. The clock
starts at node join, which is why the deaths line up with node age.

That cluster ran about 6,000 Services with six ports each, so roughly 36,000
service ports. On one node the full sync took **16 minutes**, all of it
holding `commit_mutex`.

More cores do not help, because the sync loop is single-threaded: 32x the
cores bought 1.29x. `mode: iptables` does not help either, because on Noble
the `iptables` binary is `iptables-nft` and takes the same lock.

## But 1.36 on new kernels should be faster

Most of that cost is in the kernel, and
[#135639](https://github.com/kubernetes/kubernetes/issues/135639) tracks real
improvements to it: on a newer kernel the loop check that dominates a big
transaction gets much cheaper. I run the Ubuntu EKS AMIs, and Canonical stops
building them on Noble at 1.35, so my 1.36 nodes came up on Resolute: kernel
7.0 and `nft` 1.1.6 instead of 6.x and 1.0.9.

That is a newer kernel, a newer `nft` and a newer kube-proxy at once. But the
full sync still runs every hour, **it still takes 12–20 minutes, the same as
on 1.35**, and it still grows non-linearly with Service count. Those numbers
come from a cluster built to measure sync speed, not a rebuild of the one that
died, so I have not watched a node go `NotReady` on 1.36. If you have a busy
1.36 cluster and the nerve to leave a node up for an hour, I would love to
know what you see.

## Where I disagree with everyone

Every comparison of kube-proxy modes ranks them by how fast a packet moves.
Ask instead what kube-proxy holds, and for how long, while it applies a
change, and the ranking stops looking so obvious.

I have never run IPVS at this size, so I am not going to tell you it is
better. But it updates one row of a hash table per Service, with no
transaction and no global lock in the path, and its known problem is
[userspace CPU](https://github.com/kubernetes/kubernetes/issues/135230),
which is CPU you can buy. A global lock held for an hourly rewrite of the
whole ruleset is not, and I could not find anyone who has measured the trade.

nftables scales fine. The problem is rewriting a whole cluster's ruleset
inside one globally locked transaction, on a fixed timer.

## But just use Cilium

The obvious reply is to stop using kube-proxy. Cilium replaces it with an
eBPF hash map, one entry per backend, updated in place. No transaction, no
global lock, and none of what I described above can happen. That is a real
answer, and on a new cluster I would think hard about it. On an existing one,
swapping your dataplane to dodge a timer is a big price.

The same goes for a service mesh, Istio or Linkerd or Cilium's own. Those
answer L7 questions, mTLS and retries and traffic splitting. Getting a packet
to the right pod sits a layer below all of that, and it is the only part I
needed to work.

Either way it is another product to run and another failure mode to learn.
Stock Kubernetes should be able to send packets to a Service at this size,
and stock is what most people are running.

## Love it or hate it

None of this is news upstream.
[#135639](https://github.com/kubernetes/kubernetes/issues/135639) covered
nftables sync performance and was addressed in v1.36. Whoever filed
[#139513](https://github.com/kubernetes/kubernetes/issues/139513) measured
sync time still growing super-linearly with Service count on v1.36.1, and the
[patch that closed it](https://github.com/kubernetes/kubernetes/pull/140723)
merged for 1.37: a Service with a single endpoint no longer gets its own
verdict map, because adding a rule scans every map in the table. Let's hope
that is enough.

One binary, one DaemonSet, almost nothing to configure. That is why nobody
looks at it, right up until the moment you have to.

I still cannot decide whether I like it. But I have stopped assuming it is
boring.

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

EKS on AWS, Kubernetes v1.36, Ubuntu Noble, kube-proxy in `nftables` mode,
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
[`pkg/proxy/util/utils.go:41`](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/proxy/util/utils.go#L41-L42)
in v1.36.0, and it is read once per sync in
[`proxier.go:1079`](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/proxy/nftables/proxier.go#L1079).

A compile-time constant, so no configuration flag changes it. The clock
starts at node join, which is why the deaths line up with node age.

That cluster ran about 6,000 Services with six ports each, so roughly 36,000
service ports. On one node the full sync took **16 minutes**, all of it
holding `commit_mutex`.

More cores do not help, because the sync loop is single-threaded: 32x the
cores bought 1.29x. `mode: iptables` does not help either, because on Noble
the `iptables` binary is `iptables-nft` and takes the same lock.

## Where I disagree with everyone

Every comparison of kube-proxy modes ranks them by how fast a packet moves.
Ask instead what kube-proxy holds, and for how long, while it applies a
change, and the ranking stops looking so obvious.

I have never run IPVS at this size, so I am not going to tell you it is
better. But it updates one row of a hash table per Service, with no
transaction and no global lock in the path. Its known problem is burning 65%
of a core in userspace, which is CPU you can buy. nftables holds a global
lock for a whole transaction, and kube-proxy fills that transaction with the
entire ruleset once an hour. On a big cluster I honestly do not know which of
those I would rather have, and I could not find anyone who has measured it.

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
nftables sync performance and was addressed in v1.36, which is what this
cluster runs. Whoever filed
[#139513](https://github.com/kubernetes/kubernetes/issues/139513) measured
sync time still growing super-linearly with Service count on v1.36.1, and
[#139328](https://github.com/kubernetes/kubernetes/issues/139328) is still
open on startup cost at scale. The mode is young, it is already better than
what it replaces, and it is being improved in the open.

One binary, one DaemonSet, almost nothing to configure. That is why nobody
looks at it, right up until the moment you have to.

I still cannot decide whether I like it. But I have stopped assuming it is
boring.

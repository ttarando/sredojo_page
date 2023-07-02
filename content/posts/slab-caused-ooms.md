---
title: "Slab-caused EKS nodes' OOMs"
date: 2023-07-01T22:33:54+01:00
tags: ['k8s', 'kubernetes', 'oom', 'slab', 'memory', 'leak']
draft: true
markup:
  tableOfContents:
    endLevel: 3
    ordered: true
    startLevel: 1
---

## TL;DR
I recently had very intersting case where continuesly growing slab would cause EKS nodes to become unresponsive as well as all the workloads running on them. If you're interested in what caused it and how we found the "perpetrator" read on.

## Background
We're running relatively big Kubernetes environment that consists of several clusters in multiple regions. What we noticed is that quite recently we would have few nodes becoming unresponsive every week. We also saw that when that happens our nodes are running very high on memory and that we see errors in our monitoring agent not being able to talk to kubelet on the node. But hey, our instances have 200+ GB or RAM so the question was..

## Where did our memory go?
It turned out that on any given node Kubernetes was only using up to 60% of memory so the natural question was "where is the remeaining part?". We had spent more time looking at our monitoring and found out that it was a slab eating up the memory. To be precise the unreclaimable slab. Please refer to [this documentation](https://www.kernel.org/doc/gorman/pdf/understand.pdf) (chapter 8, "Slab Allocator") on what it actually is, but to keep it super-simple it's a memory used by kernel. We also noticed that it is continuously growing which could indicate memory leak and that the growth is proportinal to memory usage growth on the node. Okay, so we knew it was slab but..

## Can we identify the exact process?
We then used `slabtop` to identify the exact slab and `perf` to further analyze its usage. Long story short it turned out it was a security scanner service from third-party vendor. We then filed a bug report and got the vendor confirming the issue however our nodes were sitting at high memory usage (and still growing) so we needed..

## The Workaround
Luckily, the vendor was able to identify the exact sub-component which we simply disabled as per their guidance. That stopped slab growth but because the nature of unreclaimable slab that memory was not released back to OS. At this point we had to force worker node rollout across all our clusters. As a result slab size went down from average ~50% of total memory to ~5%. Well done.

## Learnings and actions
10 years ago this story would have ended here for me. Today however, with SRE principles in mind we looked closely into what we can do to detect the issue earlier next time. Here are some of the actions we defined as an outcome of our analysis:
- first, rather obvious observation is that if something similar happens we cannot just simply drain and remove nodes; it is probably okay to do it once or twice but if there is pattern there must be reason,
- we also noticed, and again, it's not surprising at all, that when kubelet is unresponsive all our pods running on the node start to have problems; we therefore implemented a monitor for kubelet service so that we know about potential issues with k8s workloads as they start to fail,
- we've built a dashboard that allows for extensive memory analysis at the node level but also shows global patterns,
- we also reviewed historical data and noticed that before the memory leak observed for third-party services slab on average would consume ~5% of the node's memory. We've then setup a monitor for slab size and used 10% as a target value.

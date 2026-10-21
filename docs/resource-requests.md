# Resource requests and limits

## Current usage

```shell
kubectl top pods -A --containers --sort-by=cpu
```

One line per container, busiest first. Requests and limits are set per container, so this is the level to size at.

- `CPU(cores)` is in millicores: `1000m` is one core. It is an average over the last ~15–30 seconds.
- `MEMORY(bytes)` is the working set (RSS plus active page cache), the value that memory limits and kubelet evictions act on.

## Node headroom

```shell
kubectl top nodes
```

Usage per node, including everything outside pods (k3s, etcd, the OS). The percentages are relative to the node's allocatable
capacity, so high `MEMORY(%)` means kubelet evictions are close. On a single-node cluster this is the real limit: the
requests of all pods together must fit into that node, or new pods stay `Pending`. `kubectl describe node <name>` shows the
sum of requests and limits under "Allocated resources".

## Configured requests, limits and QoS

```shell
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,CPU_LIM:.spec.containers[*].resources.limits.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory,MEM_LIM:.spec.containers[*].resources.limits.memory,QOS:.status.qosClass'
```

Values of multi-container pods are comma-separated in container order; `<none>` means not set.

- `QOS` `BestEffort`: no container has any request; these pods are evicted first under memory pressure.
- `Burstable`: some requests set, but not requests equal to limits for every container.
- `Guaranteed`: CPU and memory requests equal limits for every container. OOM killed last

## Using them together

- CPU request: around typical usage while busy. No CPU limit: it would throttle even when the node is idle.
- Memory request: around typical usage. Memory limit: above the observed peak, so only a runaway container is OOM-killed.
- A pod picks up changed values only when it is recreated. Chart values change the pod template, so Argo CD's sync rolls
  the pods out by itself.

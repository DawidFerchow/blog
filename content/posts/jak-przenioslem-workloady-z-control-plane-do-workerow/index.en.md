+++
title = "Moving applications from control plane to workers"
slug = "moving-applications-from-control-plane-to-workers"
date = "2026-08-09"
summary = "When I started building my homelab, I began with a single node. This approach made it easier to start and simplified some things... while making other things more difficult."
+++
When I started building my homelab, I began with a single node. This approach made it easier to start and simplified some things... while making other things more difficult.

The same node was used as the control plane and also ran all applications, monitoring, FluxCD and the other parts of the cluster.

Later, I added two more nodes.

My target setup was:

* first node - control plane,
* two other nodes - workers.

This is a compromise between high availability and using real hardware for my homelab.

In a production environment, when high availability is important, the control plane should have at least three nodes. The number of workers depends on the workloads and required availability.

## Adding more nodes

At the time of writing this post, I use Ubuntu Server as the operating system for my nodes and k3s as my Kubernetes distribution.

In the future, I would like to replace Ubuntu Server with TalosOS.

According to the k3s documentation, adding another node requires installing k3s with two environment variables:

* K3S_URL - the address of the k3s server, for example `https://192.168.1.10:6443`
* K3S_TOKEN - the token stored on the control-plane node here: `/var/lib/rancher/k3s/server/node-token`

The final command looks like this:

```
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

## Moving workloads to workers

Kubernetes does not move running Pods when new workers are added. Why? The scheduler selects a node when a Pod is created and does not change it later. To force Kubernetes to remove workloads from a node, you need to use the `drain` command.

Here is an example with the --ignore-daemonsets flag.

```
kubectl drain homelab-1 --ignore-daemonsets
```

`kubectl drain` marks the node as unschedulable (`cordon`) and then removes Pods managed by the proper controllers. The controllers later recreate them on other available nodes.

Thanks to `--ignore-daemonsets`, `drain` can continue even if there are Pods managed by DaemonSets on the node. The DaemonSet Pods themselves are not removed.

## What can go wrong? (what went wrong)

### `emptyDir` blocks the `drain` operation

Kubernetes refused to remove some Pods:

```
cannot delete Pods with local storage
(use --delete-emptydir-data to override)
```

The list included:

* FluxCD controllers,
* metrics-server,
* Traefik,
* Prometheus,
* Alertmanager,
* Grafana.

At first, the message about "local storage" may sound quite worrying. However, it does not automatically mean that every Pod uses a local PersistentVolume.

In this case, the main reason was `emptyDir` volumes.

An `emptyDir` exists as long as the Pod exists. When the Pod is removed, the data also disappears. It is used, for example, as cache or temporary working space.

For FluxCD or metrics-server, losing this data is not a major problem. The controllers will be recreated, download the required data and start working again.

Monitoring requires more attention. If Prometheus does not have persistent storage configured, removing the Pod means losing the history of metrics. Grafana may also lose changes made manually through the interface if they are not stored in ConfigMaps or persistent storage.

After checking the configuration, I repeated the operation:

```
kubectl drain homelab-1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

This time, most workloads were recreated on the new nodes.

Most, but not all.

### Everything moved... except the applications

The Pods responsible for Cloudflare tunnels appeared on the workers. My applications did not.

At first, it looked like Kubernetes was not trying to recreate them at all. After checking the events, it turned out that the Pods existed, but their status was `Pending`.

```
kubectl describe pod <pod>
kubectl get pv
kubectl describe pv <pv-name>
```

The scheduler returned the following message:

```
0/3 nodes are available:
1 node(s) were unschedulable,
2 node(s) didn't match PersistentVolume's node affinity.
```

#### The Pod is portable. But its data? It depends.

My applications used PersistentVolumeClaims created by the default `local-path` provisioner in k3s.

As the name suggests, the data is stored locally on the disk of a specific node.

The PersistentVolume had `nodeAffinity` pointing to `homelab-1`, because it was my only node.

After running `drain`, the situation looked like this:

* `homelab-1` was unschedulable,
* `homelab-2` did not have access to the volume,
* `homelab-3` also did not have access to it.

The scheduler had nowhere to place the Pod.

The Cloudflare tunnel did not store any data, so it could be started on any available node. The application using a local PVC, however, depended on the place where its data was stored.

Kubernetes can recreate a Pod but it cannot move data between locations.

### Restoring the applications

The diagnosis could wait. Restoring the applications was more important.

To restore the applications, I unblocked the first node with this command:

```
kubectl uncordon homelab-1
```

After allowing the scheduler to run workloads on this node again, the applications started working.

However, this did not solve the cause of the problem.

I could leave the applications on the control plane. In a homelab, I could accept this situation. However, I did not add more nodes just to use them only partially.

## Solution

My plan is to use an old laptop as a separate storage server.

It will not be part of the Kubernetes cluster. It will be connected to the same network and will share data using NFS.

Every node will be able to mount the same volume. After removing a Pod, the controller will be able to recreate it and the scheduler will be able to place it on another worker without moving the data.

This is not a perfect solution either. One server storing data required by the applications does not provide high availability.

For my homelab, it is a reasonable compromise between:

* simplicity,
* resource usage,
* learning opportunities,
* independence of applications from a specific node.

As in life, sometimes you have to make decisions and deal with their consequences :p

 ## Conclusions

Something that made one topic easier at the beginning later created a problem in another area.

If I were building a minimal homelab again, I would start with:

* control plane
* one worker
* separate storage

It requires more work and more hardware, but from the beginning it forces a separation between compute and storage and helps avoid some of the problems I found later.

Also, Kubernetes is only a tool. It will not do everything for us, it will not make key decisions, and it will not make sure that you know what you are doing. It will do exactly what you tell it to do.


+++
title = "The story of an application outage"
slug = "the-story-of-an-application-outage"
date = "2026-07-15"
summary = "On a beautiful June day, I start my day and want to use one of my saved bookmarks. Instead of the Linkding home page, I see a Cloudflare message with Error 1033."
+++

## A few words of introduction

At the time of writing this post, I have two services hosted in my homelab. I use one of them every day, while the second one is my testing environment.

The first application is [Linkding](https://linkding.link/), which I use to save bookmarks. Why do I need it? I like testing different browsers, and thanks to Linkding, I do not have to synchronize bookmarks between my devices.

Both applications are available from the internet through [Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/).

## On a beautiful June day...

...I start my day and want to use one of my saved bookmarks. Instead of the Linkding home page, I see a Cloudflare message with `Error 1033`.

A quick look at the Cloudflare documentation gives me this information:

```
A `1033` error indicates your tunnel is not connected to Cloudflare's network because Cloudflare's network cannot find a healthy `cloudflared` instance to receive the traffic.
```

That is already something. I can investigate further.

## Investigation

### Remote investigation

Welcome to my lab :)

First, I check if there is a problem with the whole node.

`k get nodes`

```
NAME        STATUS   ROLES           AGE   VERSION
homelab-1   Ready    control-plane   29d   v1.35.4+k3s1
```

The node has the `Ready` status, so at first sight everything looks fine. However, this status alone does not show us all possible problems.

In my homelab, I use namespaces to isolate applications, with one small note: this is not full isolation.

Let’s check deeper.

`k get pods --namespace='linkding'`

For this post, I shortened the output.

```
NAME                           READY   STATUS                   RESTARTS         AGE
linkding-bf676bdc8-hghpr       0/1     ContainerStatusUnknown   0                2d8h
linkding-bf676bdc8-jvgjd       0/1     Error                    0                4d9h
linkding-bf676bdc8-kb8gb       0/1     ContainerStatusUnknown   0                2d10h
linkding-bf676bdc8-kxkwj       0/1     Error                    0                2d9h
linkding-bf676bdc8-p9nkq       0/1     Error                    0                4d
linkding-bf676bdc8-qk6dg       0/1     Error                    0                2d9h
linkding-bf676bdc8-rd995       0/1     Completed                0                5d21h
linkding-bf676bdc8-rxc5k       0/1     Error                    0                6d8h
linkding-bf676bdc8-s7tv6       0/1     Error                    0                2d13h
linkding-bf676bdc8-snjcf       0/1     Error                    0                4d9h
linkding-bf676bdc8-td4hk       0/1     Error                    0                2d8h
linkding-bf676bdc8-xqhc4       0/1     ContainerStatusUnknown   0                2d8h
```

Okay, now we can see that something is wrong. Actually, we can see more than enough.

Unfortunately, the output is not sorted, so it is quite difficult to read.

After a quick search online, I found a way to sort the results. The command looks like this:

`k get pods --sort-by='.metadata.creationTimestamp' --namespace='linkding'`

For this post, I shortened the output.

```
NAME                           READY   STATUS                   RESTARTS         AGE
linkding-bf676bdc8-td4hk       0/1     Error                    0                2d8h
cloudflared-6d986b5b96-5d62z   0/1     Completed                2 (2d8h ago)     2d8h
cloudflared-6d986b5b96-gq4h4   0/1     ContainerStatusUnknown   0                2d8h
linkding-bf676bdc8-7594h       0/1     Pending                  0                2d8h
```

Now we can clearly see that Kubernetes has a problem with starting the pods.

The question is: why?

My mentor, Mischa van den Burg, once told me that when Kubernetes has a problem, it has probably already told you what the problem is.

So, let’s check what Kubernetes told us by describing the pod that did not start.

`k describe pod linkding-bf676bdc8-7594h -n linkding`

```
Events:
  Type     Reason            Age                   From               Message
  ----     ------            ----                  ----               -------
  Warning  FailedScheduling  11m (x672 over 2d8h)  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint(s). no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

Okay, we have `0/1 nodes are available` and a warning about `untolerated taint(s)`.

By the way, there were 672 `FailedScheduling` events. The scheduler definitely did not give up :p

Because I have a good but short memory, I check the node status again.

`k get nodes`

```
NAME        STATUS   ROLES           AGE   VERSION
homelab-1   Ready    control-plane   29d   v1.35.4+k3s1
```

The node responds and still has the `Ready` status.

I need to look deeper.

Following my mentor’s advice again, I check the node description. This time, I use `grep` to find information containing the word `Taint`.

`k describe node homelab-1 | grep "Taint"`

```
Taints:             node.kubernetes.io/disk-pressure:NoSchedule
```

`disk-pressure:NoSchedule`

Now we have the reason.

I take another quick look at the [Kubernetes documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#node-conditions) to make sure I understand it correctly.

But at this point, one thing is still bothering me.

The node has 32 GB of disk space. Ubuntu Server, K3s, FluxCD and two small web applications should not use all of it.

### Direct investigation

At this point, I decide to connect directly to the machine that works as the node.

This is where some Linux knowledge becomes useful.

First, I check the K3s service logs.

`journalctl -u k3s -f`

For this post, I shortened the output.

```
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.080925 3814777 eviction_manager.go:381] "Eviction manager: attempting to reclaim" resourceName="ephemeral-storage"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.082373 3814777 container_gc.go:86] "Attempting to delete unused containers"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.091580 3814777 image_gc_manager.go:459] "Attempting to delete unused images"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.212181 3814777 eviction_manager.go:392] "Eviction manager: must evict pod(s) to reclaim" resourceName="ephemeral-storage"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.212966 3814777 eviction_manager.go:410] "Eviction manager: pods ranked for eviction" pods=["flux-system/kustomize-controller-6b7bfd984d-r8m6z","flux-system/source-controller-7c46cc6d8c-62ccl","kube-system/coredns-c4dbffb5f-mdx4c","flux-system/helm-controller-8445df54f6-zsvjd","kube-system/traefik-9bcdbbd9-dq4ds","kube-system/local-path-provisioner-5c4dc5d66d-w9rv5","kube-system/metrics-server-786d997795-t6ghl","kube-system/svclb-traefik-e7142bfa-qrqjb"]
Jun 17 16:18:42 homelab-1 k3s[3814777]: E0617 16:18:42.213132 3814777 eviction_manager.go:616] "Eviction manager: cannot evict a critical pod" pod="flux-system/kustomize-controller-6b7bfd984d-r8m6z"
```

The logs confirm the message from the Kubernetes API, but I still do not understand why there is not enough free disk space.

Instead of guessing, I search the documentation:

https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#hard-eviction-thresholds

I find information about the exact moment when Kubernetes decides that there is not enough available space.

* `memory.available<100Mi` (Linux nodes)
* `memory.available<500Mi` (Windows nodes)
* `nodefs.available<10%`
* `imagefs.available<15%`
* `nodefs.inodesFree<5%` (Linux nodes)
* `imagefs.inodesFree<5%` (Linux nodes)

Here we can see that the default threshold for `nodefs` is below 10%.

It is time to check how the situation looks directly in Ubuntu Server.

Again, Linux knowledge and basic commands are useful here.

`df -h`

```
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              633M  2.9M  631M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   15G   13G  880M  94% /
tmpfs                              1.6G     0  1.6G   0% /dev/shm
efivarfs                           132k   67k   60k  53% /sys/firmware/efi/efivars
tmpfs                              1.6G     0  1.6G   0% /tmp
/dev/sda2                          2.1G  193M  1.8G  11% /boot
/dev/sda1                          1.2G  6.7M  1.2G   1% /boot/efi
none                               1.1M     0  1.1M   0% /run/credentials/getty@tty1.service
none                               1.1M     0  1.1M   0% /run/credentials/systemd-journald.service
none                               1.1M     0  1.1M   0% /run/credentials/systemd-resolved.service
none                               1.1M     0  1.1M   0% /run/credentials/systemd-networkd.service
tmpfs                              317M  8.2k  317M   1% /run/user/1000
```

Okay, now we have something.

The disk usage is 94%.

Kubernetes behaved correctly.

But another question appears: why does my LVM have only 15 GB assigned instead of the full 32 GB available on the physical disk?

I use the `lsblk` command, which returns:

```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 29.8G  0 disk
├─sda1                      8:1    0    1G  0 part /boot/efi
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0 26.8G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0   15G  0 lvm  /
```

The `sda3` partition had 26.8 GB, but the logical volume stored on it was using only 15 GB.

The remaining 11.8 GB was free inside the LVM volume group and was not assigned to the `/` filesystem.

The fix was quite simple.

I used this command:

`lvresize -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`

The command extends the logical volume by using 100% of the available free space in the volume group.

After running it, I got this result:

```
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              633M  2.9M  631M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   27G   13G   15G  48% /
tmpfs                              1.6G     0  1.6G   0% /dev/shm
efivarfs                           132k   67k   60k  53% /sys/firmware/efi/efivars
tmpfs                              1.6G     0  1.6G   0% /tmp
/dev/sda2                          2.1G  193M  1.8G  11% /boot
/dev/sda1                          1.2G  6.7M  1.2G   1% /boot/efi
none                               1.1M     0  1.1M   0% /run/credentials/getty@tty1.service
none                               1.1M     0  1.1M   0% /run/credentials/systemd-journald.service
none                               1.1M     0  1.1M   0% /run/credentials/systemd-resolved.service
none                               1.1M     0  1.1M   0% /run/credentials/systemd-networkd.service
tmpfs                              317M  8.2k  317M   1% /run/user/1000
```

And this is the new output from `lsblk`:

```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 29.8G  0 disk
├─sda1                      8:1    0    1G  0 part /boot/efi
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0 26.8G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 26.8G  0 lvm  /
```

At this point, I could remove the taint manually or wait for Kubernetes to check the node again.

Because of scientific curiosity, I left it to Kubernetes.

After several minutes, everything returned to normal.

The taint was a result of `DiskPressure`, not the main cause of the problem. That is why I did not remove it manually.

I let Kubernetes check the node again, and after several minutes, everything was working correctly.

## So, what now?

Of course, the best solution would be to avoid human errors, but we all know how it is.

From the technical side, this situation could be improved by adding proper Prometheus rules and configuring Alertmanager to send a warning before the amount of free disk space falls below the thresholds used by kubelet.

This would not prevent the problem itself, but it would allow me to react at the right moment — before the service becomes unavailable.


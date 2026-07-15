+++
title = "Historia awarii pewnej aplikacji"
slug = "historia-awarii-pewnej-aplikacji"
date = "2026-07-15"
summary = "Pięknego czerwcowego dnia zaczynam swój dzień i chcę skorzystać z jednej z zapisanych zakładek. Zamiast strony startowej linkding, wita mnie komunikat cloudflare o błędzie Error 1033"
+++
## Słowem wstępu
Na dzień pisania tego postu posiadam 2 usługi, które hostuję na moim homelabie. Z jednej korzystam na co dzień a druga służy mi jako poligon doświadczalny. Pierwszą z nich jest [Linkding](https://linkding.link/), który służy mi do zapisywania zakładek. Po co? Lubię testować różne przeglądarki i dzięki linkding nie muszę synchronizować zakładek między urządzeniami. Zarówno jedna i druga jest udostępniona na zewnątrz przez [Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/).

## Pięknego czerwcowego dnia...
... zaczynam swój dzień i chcę skorzystać z jednej z zapisanych zakładek. Zamiast strony startowej linkding, wita mnie komunikat cloudflare o błędzie  'Error 1033'. Szybki rzut oka na dokumentację cloudflare, i:

```
A `1033` error indicates your tunnel is not connected to Cloudflare's network because Cloudflare's network cannot find a healthy `cloudflared` instance to receive the traffic.
```

To już coś, mogę inwestygować dalej

## Inwestygacja
### Zdalna
Zapraszam w takim razie do mojego laba :) Najpierw sprawdzam czy nie ma problemów z całym node.

`k get nodes`

```
NAME        STATUS   ROLES           AGE   VERSION
homelab-1   Ready    control-plane   29d   v1.35.4+k3s1
```

Mamy status ready więc na pierwszy rzut oka wygląda to dobrze natomiast sam status nie daje nam poglądu na wszystkie obszary. W homelabie wykorzystuje namespace'y do izolacji aplikacji z gwiazdką, że nie jest to pełna izolacja. No to sprawdzam głębiej:

`k get pods --namespace='linkding'`
(Na potrzeby wpisu skróciłem odpowiedź)

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

Oho, tutaj już coś widać. Nawet więcej niż dużo ale niestety nie jest to posortowane przez co ciężko się to czyta.

Szybki research w internetach i znalazłem sposób na sortowanie. Polecenie wygląda mniej więcej tak:

`k get pods --sort-by='.metadata.creationTimestamp' --namespace='linkding'`
(Na potrzeby wpisu skróciłem odpowiedź)

```
NAME                           READY   STATUS                   RESTARTS         AGE
linkding-bf676bdc8-td4hk       0/1     Error                    0                2d8h
cloudflared-6d986b5b96-5d62z   0/1     Completed                2 (2d8h ago)     2d8h
cloudflared-6d986b5b96-gq4h4   0/1     ContainerStatusUnknown   0                2d8h
linkding-bf676bdc8-7594h       0/1     Pending                  0                2d8h
```

No i tutaj już widzimy, że kubernetes ma problem z postawieniem podów. Pytanie, dlaczego? Mój mentor Mischa van den Burg stwierdził, że jak kubernetes ma jakiś problem to na pewno już powiedział Ci jaki. No więc, sprawdźmy co nam powiedział poprzez sprawdzenie opisu poda, który nie wstał.

``k describe pod linkding-bf676bdc8-7594h -n linkding``

```
Events:
  Type     Reason            Age                   From               Message
  ----     ------            ----                  ----               -------
  Warning  FailedScheduling  11m (x672 over 2d8h)  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint(s). no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.

```

Okay, 0/1 nodes are available oraz warning informujący o untolerated taint(s). Btw. 672 wystąpień FailedScheduling. Scheduler zdecydowanie nie odpuszczał :p

Jako, że mam dobrą pamięć ale krótką, sprawdzam jeszcze raz stan node'a.

`k get nodes`

```
NAME        STATUS   ROLES           AGE   VERSION
homelab-1   Ready    control-plane   29d   v1.35.4+k3s1
```

Node odpowiada i ma status ready.  Sprawdzam głębiej. Więc znowu idąc za radą mojego mentora, sprawdzam opis node'a ale tym razem wykorzystuje grep do znalezienia informacji po słowie 'Taint'.

`k describe node homelab-1 | grep "Taint"`

```
Taints:             node.kubernetes.io/disk-pressure:NoSchedule
```

`disk-pressure:NoSchedule`

No to mamy powód. Jeszcze szybki wgląd do [dokumentacji](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#node-conditions) aby upewnić się w temacie.

Ale.. Na tym etapie jedna myśl nie daje mi spokoju. Node ma dostęp do 32GB przestrzeni dyskowej do dyspozycji. Ubuntu server + K3s + FluxCD + 2 małe aplikacje webowe nie powinny zająć takiej ilości przestrzeni.

### Bezpośrednia
Na tym etapie postanowiłem podłączyć się bezpośrednio do maszyny, która jest node'm. Tutaj przydaje się wiedza odnośnie linuxa. Sprawdzam sobie dziennik aplikacji k3s.

`journalctl -u k3s -f`
(Na potrzeby wpisu skróciłem odpowiedź)

```
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.080925 3814777 eviction_manager.go:381] "Eviction manager: attempting to reclaim" resourceName="ephemeral-storage"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.082373 3814777 container_gc.go:86] "Attempting to delete unused containers"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.091580 3814777 image_gc_manager.go:459] "Attempting to delete unused images"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.212181 3814777 eviction_manager.go:392] "Eviction manager: must evict pod(s) to reclaim" resourceName="ephemeral-storage"
Jun 17 16:18:42 homelab-1 k3s[3814777]: I0617 16:18:42.212966 3814777 eviction_manager.go:410] "Eviction manager: pods ranked for eviction" pods=["flux-system/kustomize-controller-6b7bfd984d-r8m6z","flux-system/source-controller-7c46cc6d8c-62ccl","kube-system/coredns-c4dbffb5f-mdx4c","flux-system/helm-controller-8445df54f6-zsvjd","kube-system/traefik-9bcdbbd9-dq4ds","kube-system/local-path-provisioner-5c4dc5d66d-w9rv5","kube-system/metrics-server-786d997795-t6ghl","kube-system/svclb-traefik-e7142bfa-qrqjb"]
Jun 17 16:18:42 homelab-1 k3s[3814777]: E0617 16:18:42.213132 3814777 eviction_manager.go:616] "Eviction manager: cannot evict a critical pod" pod="flux-system/kustomize-controller-6b7bfd984d-r8m6z"
```

Logi potwierdzają mi komunikat z API Kubernetsa ale dalej mi nie pasuje brak wolnego miejsca. Zamiast się zastanawiać, przeszukuję dokumentację:

https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#hard-eviction-thresholds

I znajduję informację o tym, w którym dokładnie momencie kubernetes uznaję, że nie ma wystarczającego miejsca.

- `memory.available<100Mi` (Linux nodes)
- `memory.available<500Mi` (Windows nodes)
- `nodefs.available<10%`
- `imagefs.available<15%`
- `nodefs.inodesFree<5%` (Linux nodes)
- `imagefs.inodesFree<5%` (Linux nodes)

No i tutaj mamy informację, że dla `nodefs` domyślny próg jest poniżej 10%.

Czas więc sprawdzić jak to wygląda w samym Ubuntu Server. Znowu przydaje się znajomość Linuxa i komend.

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

Oho, i mamy użycie dysku w 94%. Kubernetes zachował się prawidłowo. Teraz znowu nasuwa się pytanie, dlaczego do LVM mam przypisane 15G zamiast 32, które mam fizycznie?

Tutaj skorzystałem z polecenia: `lsblk`, które zwróciło mi:
```NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 29.8G  0 disk
├─sda1                      8:1    0    1G  0 part /boot/efi
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0 26.8G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0   15G  0 lvm  /
```

Partycja `sda3` miała 26,8 GB, ale znajdujący się na niej wolumin logiczny wykorzystywał tylko 15 GB. Pozostałe około 11,8 GB było wolne w grupie woluminów LVM i nie zostało przydzielone do systemu plików `/`.

No więc, szybka naprawa poleceniem: `lvresize -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`, które rozszerzy LVM o 100% dostępnego, wolnego miejsca w grupie woluminów i mamy taki o efekt:

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

i widok z ponownego `lsblk`:
```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 29.8G  0 disk
├─sda1                      8:1    0    1G  0 part /boot/efi
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0 26.8G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 26.8G  0 lvm  /
```

Teraz mogłem ręcznie usunąć taint albo dać kubernetesowi ponowne sprawdzenie node'a. Z naukowej ciekawości zostawiłem to kubernetesowi i po kilkunastu minutach wszystko wróciło do normy.

Taint był skutkiem `DiskPressure`, a nie przyczyną problemu, dlatego nie usuwałem go ręcznie. Zostawiłem to do ponownego sprawdzenia kubernetesowi i po kilkunastu minutach wszystko wróciło do normy.


## Co zrobić, jak żyć?
Oczywiście, najlepiej byłoby uniknąć błędu ludzkiego ale wiadomo jak to bywa. Techniczne można rozwiązać to dodając odpowiednie reguły Prometheusa i skonfigurować Alertmanagera tak, aby wysyłał ostrzeżenie, zanim ilość wolnego miejsca spadnie poniżej progów wykorzystywanych przez kubelet.

To co prawda nie zapobiegnie powstania problemu ale pozwoli zareagować w odpowiednim momencie czyli przed brakiem dostępności do usługi.

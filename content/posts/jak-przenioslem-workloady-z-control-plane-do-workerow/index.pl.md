+++
title = "Historia awarii pewnej aplikacji"
slug = "historia-awarii-pewnej-aplikacji"
date = "2026-07-15"
summary = "Pięknego czerwcowego dnia zaczynam swój dzień i chcę skorzystać z jednej z zapisanych zakładek. Zamiast strony startowej linkding, wita mnie komunikat cloudflare o błędzie Error 1033"
+++
# Jak przeniosłem workloady z control-plane do workerów
Budując mojego homelaba zacząłem od jednego node'a. Takie podejście obniżyło próg wejścia i uprościło kilka rzeczy... jednocześnie utrudniając kilka innych.

Ten sam node pełnił rolę control-plane oraz uruchamiał wszystkie aplikacje, monitoring, FluxCD oraz pozostałe elementy klastra.

Z czasem dołożyłem dwa kolejne nody.

Docelowo, miało to wyglądać tak:
- pierwszy node -  control-plane,
- dwa pozostałe - workery.

Jest to kompromis między wysoką dostępnością oraz użyciem faktycznego sprzętu dla homelab'a.

W środowisku produkcyjnym,  kiedy zależy nam na wysokiej dostępności, control-plane powinien mieć co najmniej trzy nody. Liczba workerów zależy już od uruchamianych workloadów i wymaganej dostępności.

## Dodanie kolejnych node'ów
Na dzień pisania tego wpisu jako system dla moich node'ów wykorzystuję ubuntu server oraz k3s jako dystrybucję Kubernetesa.

W przyszłości chciałbym zastąpić ubuntu server systemem TalosOS.

Zgodnie z dokumentacją k3s, dodanie kolejnego node'a polega na instalacji k3s z ustawionymi dwiema zmiennymi środowiskowymi:
- K3S_URL - adres serwera k3s, np. `https://192.168.1.10:6443`
- K3S_TOKEN - jest to token przechowywany na control-plane w tym miejscu: `/var/lib/rancher/k3s/server/node-token`

Końcowo coś takiego:
```
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

## Przeniesienie workload'ów na workery
Kubernetes nie przenosi działających podów w momencie kiedy pojawią się nowe workery. Dlaczego? Scheduler wybiera node podczas tworzenia poda i nie modyfikuje tego w przyszłości. Aby wymusić opróżnienie node'a z workload'ów należy użyć polecenia 'drain'. 

Tutaj przykład, z flagą --ignore-daemonsets. 

```bash
kubectl drain homelab-1 --ignore-daemonsets
```

`kubectl drain` oznacza node jako nieschedulowalny (`cordon`), a następnie usuwa z niego Pody zarządzane przez odpowiednie kontrolery. Kontrolery odtwarzają je później na innych dostępnych nodach.

Dzięki `--ignore-daemonsets` `drain` wykona się pomimo tego, że na Nodzie znajdują się Pody zarządzane przez DaemonSet. Same Pody DaemonSetów nie są usuwane.

## Co może pójść nie tak? (co poszło nie tak)

### `emptyDir` blokuje wykonanie `drain`

Kubernetes odmówił usunięcia części Podów:

```text
cannot delete Pods with local storage
(use --delete-emptydir-data to override)
```

Na liście znajdowały się między innymi:
- kontrolery FluxCD,
- metrics-server,
- Traefik,
- Prometheus,
- Alertmanager,
- Grafana.

Początkowo komunikat o „local storage” może brzmieć dość niepokojąco. Nie oznacza jednak automatycznie, że każdy z tych Podów korzysta z lokalnego PersistentVolume.

W tym przypadku chodziło przede wszystkim o wolumeny `emptyDir`.

`emptyDir` istnieje tak długo, jak istnieje Pod. Po jego usunięciu dane również znikają. Jest wykorzystywany między innymi jako cache lub przestrzeń robocza.

Dla FluxCD czy metrics-servera utrata takich danych nie jest większym problemem. Kontrolery zostaną odtworzone, pobiorą potrzebne dane i wrócą do pracy.

Więcej zastanowienia wymaga monitoring. Jeśli Prometheus nie ma skonfigurowanego trwałego wolumenu, usunięcie Poda oznacza utratę historii metryk. Podobnie Grafana może stracić zmiany wykonane ręcznie przez interfejs, jeśli nie są przechowywane w ConfigMapach lub trwałym storage.

Po sprawdzeniu konfiguracji ponowiłem operację:

```bash
kubectl drain homelab-1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Tym razem większość workloadów została odtworzona na nowych nodach.

Większość, ale nie wszystkie.

### Przeniosło się wszystko... oprócz samych aplikacji
Pody odpowiedzialne za tunele Cloudflare pojawiły się na workerach. Moje aplikacje nie.

Początkowo wyglądało to tak, jakby Kubernetes w ogóle nie próbował ich odtworzyć. Po sprawdzeniu eventów okazało się, że Pody istnieją, ale jako `Pending`.

```
kubectl describe pod <pod>
kubectl get pv
kubectl describe pv <pv-name>
```

Scheduler zwracał następujący komunikat:

```text
0/3 nodes are available:
1 node(s) were unschedulable,
2 node(s) didn't match PersistentVolume's node affinity.
```

#### Pod jest przenośny. A jego dane? Zależy.
Moje aplikacje korzystały z PersistentVolumeClaimów utworzonych przez domyślny w k3s `local-path`.

Jak sama nazwa sugeruje, dane znajdują się lokalnie na dysku konkretnego noda.

PersistentVolume posiadał `nodeAffinity` wskazujące na `homelab-1`, ponieważ to był mój jedyny node.

Po wykonaniu `drain` sytuacja wyglądała następująco:
- `homelab-1` był nieschedulowalny,
- `homelab-2` nie miał dostępu do wolumenu,
- `homelab-3` również nie miał do niego dostępu.

Scheduler nie miał gdzie umieścić Poda.

Tunel Cloudflare nie przechowywał danych, dlatego mógł zostać uruchomiony na dowolnym dostępnym nodzie. Aplikacja korzystająca z lokalnego PVC była natomiast zależna od miejsca, w którym znajdowały się jej dane.

Kubernetes potrafi odtworzyć Poda ale nie może przenieść danych między lokalizacjami.

### Przywrócenie aplikacji
Sama diagnostyka mogła poczekać. Przywrócenie aplikacji jest ważniejsze.

Żeby przywrócić działanie aplikacji, odblokowałem pierwszy node poleceniem:

```bash
kubectl uncordon homelab-1
```

Po umożliwieniu schedulerowi uruchamiania workloadów na tym nodzie aplikacje zaczęły działać ponownie.

Nie rozwiązało to jednak przyczyny problemu.

Mógłbym zostawić aplikacje na control-plane. W homelabie mogłem pozwolić sobie na taki stan rzeczy. Niemniej, nie po to dodawałem kolejne nody, żeby tylko częściowo z nich korzystać.

## Rozwiązanie
Docelowo planuję wykorzystać starego laptopa jako osobny storage.

Nie będzie należał do klastra Kubernetes. Zostanie podłączony do tej samej sieci i będzie udostępniał dane przez NFS.

Każdy node będzie mógł zamontować ten sam wolumen. Po usunięciu Poda kontroler będzie mógł go odtworzyć, a scheduler umieścić na innym workerze bez konieczności przenoszenia danych.

Nie jest to też idealne rozwiązanie. Jeden serwer na którym są dane konieczne do działania aplikacji nie zapewnia high availability.

Na potrzeby mojego homelaba jest to rozsądny kompromis między:
- prostotą,
- zużyciem zasobów,
- możliwością nauki,
- niezależnością aplikacji od konkretnego noda.

Jak to w życiu bywa, trzeba podejmować decyzje i mierzyć się z ich konsekwencjami :p

## Wnioski
Coś, co początkowo ułatwiło mi zrealizowanie jednego tematu, w konsekwencji zrodziło problem później, przy innym temacie.

Gdybym teraz stawiał homelab minimalnie zacząłbym od:
- control plane
- jeden worker
- osobny storage

Co prawda wymaga to większych nakładów pracy i sprzętu, ale od początku wymusza rozdzielenie compute od storage i pozwala uniknąć części problemów, na które trafiłem później.

Dodatkowo, kubernetes jest tylko narzędziem. Nie wykona wszystkiego za nas, nie podejmie kluczowych decyzji, nie upewni się, czy wiesz co robisz. Zrobi dokładnie to, o co go poprosisz.

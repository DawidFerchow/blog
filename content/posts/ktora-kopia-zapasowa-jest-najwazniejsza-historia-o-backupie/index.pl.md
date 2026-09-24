+++
title = "Która kopia zapasowa jest najważniejsza?"
slug = "ktora-kopia-zapasowa-jest-najwaznieszja"
date = "2026-09-21"
summary = "Ludzie dzielą się na tych co robią kopie zapasowe i na tych, którzy te kopie robić będą. Ja jestem w tych, co te kopie robią natomiast to nie oznacza, że robienie kopii chroni Cię przed utratą danych."
+++
Ludzie dzielą się na tych co robią kopie zapasowe i na tych, którzy te kopie robić będą. Ja jestem w tych, co te kopie robią natomiast to nie oznacza, że robienie kopii chroni Cię przed utratą danych.
## Wstęp
Odkopałem jeden ze starych dysków, z komputera, który pamięta jeszcze względnie normalne nazewnictwo procesorów intela. 

Po lekkim oczyszczeniu z kurzu i znalezieniu kabla DisplayPort, zalogowałem się do systemu. Ostatni dobry system od MS przywitał mnie z prośbą o hasło do konta lokalnego. Tak, to Windows 10, który nie wymagał posiadania konta online. 

Zacząłem przeglądać foldery i znalazłem sporo zdjęć z przeszłości, zrzutów ekranu stron, które zaprojektowałem i wdrożyłem razem z ich kodem źródłowym i tym podobne rzeczy a których kopii jeszcze nie miałem zrobionej. 
## Techniczny zarys sprzętu i oprogramowania
Ten komputer nie ma portów USB 3.0, które umożliwiłyby mi szybkie skopiowanie danych. Z szybkich opcji miałem do wyboru porty SATA ale znowu nie mialem pod ręką takich (spoiler - jak przyszło co do czego to znalazłem). 

Pomyślałem więc, że wykorzystam mojego głównego laptopa, dysk z komputera podłącze przez adapter SATA -> USB 3.+ i dysk PCie, który mam w adapterze również do USB 3.+.

Mój przyjaciel, którego w tym miejscu serdecznie pozdrawiam, pokazał mi kiedyś narzędzie Disk2vhd, które potrafi utworzyć dysk wirtualny systemu, który później możemy włączyć jako wirtualną maszynę. Tworzy komputer, który mogę odpalić na innym komputerze (skrót myślowy).

Na moim głównym laptopie korzystam z Dual Boot. Mam zainstalowanego zarówno Windowsa i Linuxa. Najchętniej zostawiłbym tylko Linuxa ale nie wszystko na nim działa, głównie za sprawą sterowników do dosyć nietypowych urządzeń.

Właśnie na Windowsie postanowiłem skorzystać z Disk2vhd i kombinacji podpięcia dysków.

Sprzęt jest. Soft jest. Plan jest. Można odpalić tworzenie kopii i wypić kawkę.
## Podejście do kopii
No więc, jestem na Windows, dyski są podłączone, uruchamiam Disk2vhd, wybieram jakie partycje i gdzie wirtualna maszyna ma się utworzyć, klikam 'Start', proces się rozpoczyna.

Powoli, ale idzie. Estymowane jest 15 minut.

I po 5 minutach pojawił się bład I/O.

Pomyślałem, że w sumie to nic. Zamknąłem okienko z błędem i disk2vhd. Sprawdziłem dysk docelowy, pokazał mi swoją zawartość. Przeszedłem do sprawdzenia dysku źródłowego. No i tutaj zaczęły się schody. Windows wykrywał 2 partycje z tego dysku ale nie pokazywał zużycia przestrzeni na nich. Kliknąłem na partycję z danymi i nic. Eksplorator 'myślał'.

Wyłączyłem Windowsa, włączyłem Linux'a i tutaj kolejna niespodzianka. Przy próbie zamontowania dysku źródłowego pojawiał się błąd mówiący, że nie można zamontować tego dysku.

W tym momencie pojawiła się lekka panika.

Wyjąłem więc dysk z adaptera, podpiąłem do komputera, z którego go wyciągnąłem i włączyłem komputer.

Windows zaczął się uruchamiać, odetchnąłem, a po chwili pojawił się blue screen z komunikatem, że nie może zamontować partycji.

Kolana mi się ugięły. Dane, które tam były uważałem za utracone, więc były dla mnie podwójnie ważne. Znalazłem je po to, żeby je stracić.
## Odzyskiwanie danych
Dawno, dawno temu, razem z dwoma współpracownikami zostaliśmy wyróżnieni za odzyskanie danych z dysku firmowego.

Przypomniałem sobie o tym.

Wtedy użyliśmy ``ddrescue`` oraz ``dmde`` i się udało.

Nie wiedząc co się tak właściwie stało, zrobienie kopii posektorowej na sprawny dysk aby później próbować odzyskać dane wydaje mi się słusznym podejściem.

GNU ``ddrescue`` potrafi kopiować dane bezpośrednio z urządzenia blokowego. Do tego zostało stworzone z myślą o sytuacjach, w których występują problemy z odczytem. Zapisywany plik mapy pozwala wznowić kopiowanie bez zaczynania wszystkiego od początku.

Miałem do dyspozycji drugi dysk HDD o pojemności 500 GB. Był wcześniej używany jako element RAID-u w Linuxie. Wyciągnięcie jednego dysku z używanej przeze mnie macierzy nie powinno spowodować jej wysypania.

Podłączyłem go jako drugi dysk do komputera, z którego pochodził uszkodzony dysk, już bezpośrednio przez SATA.

Wrzuciłem sobie ISO gparted na Ventoy'a i odpaliłem gparted. Tutaj mógłby być praktycznie dowolny linux, który włączy się jako Live. Gparted jest zbudowany na Debianie więc miałem wszystkie potrzebne narzędzia.

Zacząłem od terminala, wiadomo. Później sprawdzenie ``lsblk``, jakie oznaczenia mają dyski żeby przenieść z prawidłowego na prawidłowy.

Sprawdziłem kilka razy :D

Pierwsze przejście poszło z parametrem ``-n``:
```
sudo ddrescue -f -n /dev/sda /dev/sdb rescue.log
```

który sprawia, że pierwsze przejście nie próbuje od razu męczyć ewentualnych, problematycznych obszarów. Najpierw chcemy odzyskać wszystko, co da się normalnie odczytać.
## Najdłuższe 120GB w życiu
Najgorsze co mogło się wydarzyć to duża liczba błędów podczas kopiowania. Z każdą minutą i przerzuconymi MB błędów nie przybywało, co dawało mi dużą nadzieję, że problem jest mniejszy niż myślałem.

Patrzyłem na proces i spodziewałem się, że w końcu pojawią się jakieś błędy.

Nie pojawiły się.

`ddrescue` doszedł do końca i nie zgłosił żadnych błędów odczytu.

Zero.

Nie było `read errors`, nie było `bad areas`, nie było `errsize`.

Dysk, który chwilę wcześniej nie chciał się normalnie zamontować, został bez problemu odczytany sektor po sektorze.
## Kluczowe dane
Po wykonaniu ``ddrescue`` wyłączyłem komputer, odpiąłem dysk źródłowy, zostawiłem docelowy, uruchomiłem komputer i odpaliłem znowu gparted.

Po zamontowaniu partycji z danymi okazało się, że mogę normalnie przeglądać katalogi.

Zdjęcia, dokumenty, projekty, stare pliki, których dawno nie otwierałem i oczywiście kilka gigabajtów rzeczy, których prawdopodobnie nigdy więcej nie użyję.

Ale były.

I to było najważniejsze.

Zacząłem kopiować dane na kolejny zewnętrzny dysk.

Do kopiowania wykorzystałem `rsync`, ponieważ można wznowić kopiowanie i nie trzeba zaczynać wszystkiego od początku, jeżeli jakiś plik sprawi problem.

A było kilka takich plików. Co prawda nie zablokowały przenoszenia ale znalazły się w logu. Były to pliki głównie z `AppData` oraz samej aplikacji OneDrive (nie korzystalem z niego). 

Rollercoaster. Znalazłem dane, straciłem je a na końcu odzyskałem.

Mając już kopię posektorową, nie było sensu ryzykować modyfikowania oryginalnego dysku. Najpierw chciałem odzyskać dane, a dopiero później zastanawiać się, czy Windowsa da się jeszcze uruchomić.
## Co właściwie się stało?
Tego nie wiedzą nawet najstarsi szamani, ale mam pewną teorię.

Proces wyglądał tak:
1. Disk2vhd
2. błąd I/O
3. Windows: UNMOUNTABLE_BOOT_VOLUME
4. Linux: problem z montowaniem
5. ddrescue: 120 GB skopiowane bez błędów
6. dostęp do danych

Przy przenoszeniu danych Disk2vhd mógł doprowadzić do sytuacji, w której partycja została oznaczona jako `dirty`. Taka flaga informuje system, że system plików wymaga sprawdzenia. Nie mam jednak pewności, czy faktycznie to było przyczyną problemu. 

Flaga nie została zdjęta, bo proces został przerwany. Dlaczego został przerwany? Najbardziej podejrzana jest stacja dokująca, której używam do rozszerzenia portów w moim laptopie (mam tylko 4 USB-C - nie pozdrawiam Dell'a).

Nie mniej, jest to tylko moja teoria.
## Lekcja na przyszłość
Najważniejszy backup, to ten pierwszy. I bezpieczeństwo jego wykonania. 

Mój błąd? Skorzystałem z dodatkowych warstw, które miały mi ułatwić i przyspieszyć wykonanie kopii - stacji dokującej i adaptera SATA - USB.

Być może rozsądniejszym byłoby przenieść najważniejsze dane w sposób bardziej klasyczny a dopiero później robić tak dużą operację na plikach.

Sumarycznie, kosztowało mnie to dużo nerwów, natomiast zachowałem zimną głowę i na koniec dnia mam zarówno kopię jak i wpis na bloga i LinkedIn'a :)

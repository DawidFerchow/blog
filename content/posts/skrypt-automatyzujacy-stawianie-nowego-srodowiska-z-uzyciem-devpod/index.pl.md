---
title: "Skrypt automatyzujący stawianie nowego środowiska z użyciem DevPod"
date: "2026-09-04"
slug: "skrypt-automatyzujacy-stawianie-nowego-srodowiska-z-uzyciem-devpod"
summary: "Na codzień korzystam z Devpod. Działa to świetnie ale każdy nowy projekt lub eksperyment zaczyna się od tej samej, nudnej sekwencji w terminalu. Postanowiłem coś z tym zrobić."
---
## Problem
Na co dzień korzystam z DevPod, które to znowu korzysta z Development Containers. Działa to mniej więcej tak, że definiuję plik devcontainer, który zawiera obraz lub Dockerfile i uruchamia powtarzalne środowisko.

Dzięki temu mogę korzystać np. z różnych wersji PHP w obrębie jednego systemu. Działa to świetnie ale jest jeden, mały problem.

Każdy nowy projekt lub eksperyment zaczyna się od tej samej, nudnej sekwencji w terminalu. Mniej więcej coś takiego:

```
# 1. Stworzenie katalogu
mkdir my-app

# 2. Przejście do niego
cd my-app

# 3. Pobranie szablonu projektu (boilerplate)
git clone boilerplate-repo.

# 4. Usunięcie katalogu git powiązanego z boilerplate
rm -rf .git

# 5. Postawienie kontenera deweloperskiego z własnymi dotfiles
devpod up . --dotfiles your-dotfiles-repo
```

Punkt 5 można usprawnić definiując zmienną w samym DevPod:
```bash
devpod context set-options -o DOTFILES_URL=your-dotfiles
```

ale dalej mam do wpisania 5 komend.

Wykonanie tego raz czy dwa razy w tygodniu nie stanowi problemu. Jednak gdy mam to pisać 3 razy dziennie, jest to po prostu strata czasu, która może się zwiększyć przez np. literówki.
## Rozwiązanie - skrypt Bash
Pokusiłem się o napisanie skryptu, który z 5 poleceń zrobiłby po prostu jedno bo dlaczego nie? Wyszło coś takiego:

```bash
#!/usr/bin/env bash
set -euo pipefail

dotfiles=""
project_name=""
path_to_project_dir="${DEVPOD_PROJECTS_DIR:-}"
project_boilerplate="${DEVPOD_REPO_BOILERPLATE:-}"
project_path=""

usage() {
  local exit_code="${1:-1}"

  cat << EOF
  Usage: $0 [-d <repo_personal_dotfiles>] <project_name> [path_to_project_parent_dir] [repo_boilerplate_project]

  Options:
    -d URL   (Optional) Repo with your personal dotfiles
    -h       Show help

  Environment variables:
    DEVPOD_PROJECTS_DIR     - absolute path for your projects directory
    DEVPOD_REPO_BOILERPLATE - URL for projects boilerplate REPO
EOF
  exit "$exit_code"
}

parse_args() {
  while getopts "d:h" o; do
    case "${o}" in
      d)
        dotfiles=$OPTARG ;;
      h)
        usage 0 ;;
      *)
        usage 1 ;;
    esac
  done

  shift "$((OPTIND - 1))"

  project_name="${1:-}"
  [ -n "${2:-}" ] && path_to_project_dir="$2"
  project_path="${path_to_project_dir}/${project_name}"
  [ -n "${3:-}" ] && project_boilerplate="$3"
}

check_dependencies() {
  for cmd in git devpod; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
      echo "Error: Required command '$cmd' is not installed." >&2
      exit 1
    fi
  done
}

validate_data() {
  check_dependencies

  if [ -z "$project_name" ]; then
    echo "Error: missing project name argument." >&2
    usage 1
  fi

  if [ -z "$path_to_project_dir" ]; then
    echo "Error: missing path to projects directory." >&2
    usage 1
  fi

  if [[ ! "$path_to_project_dir" =~ ^/ ]]; then
    echo "Error: Path '$path_to_project_dir' needs to be an absolute path." >&2
    usage 1
  fi

  if [ -d "$project_path" ]; then
    echo "Error: Target directory '$project_path' already exists." >&2
    exit 1
  fi

  if [ -z "$project_boilerplate" ]; then
    echo "Error: missing boilerplate repo argument." >&2
    usage 1
  fi

  local regex_git_repo='^((https?|git|ssh|file)://|git@[[:alnum:]._-]+:|/).+$'

  if [[ ! "$project_boilerplate" =~ $regex_git_repo ]]; then
    echo "Error: $project_boilerplate does not look like a valid git repo." >&2
    usage 1
  fi

  if [ -n "$dotfiles" ]; then
    if [[ ! "$dotfiles" =~ $regex_git_repo ]]; then
      echo "Error: $dotfiles does not look like a valid git repo." >&2
      usage 1
    fi
  fi
}

cleanup() {
  local exit_code=$?
  if [ $exit_code -ne 0 ] && [ -n "${project_path:-}" ] && [ -d "$project_path" ]; then
    echo -e "\n==> Cleaning up incomplete project directory: $project_path" >&2
    rm -rf "$project_path"
  fi
}

main () {
  trap cleanup EXIT INT TERM

  echo "==> Creating project directory: $project_path"
  mkdir -p "$project_path"

  echo "==> Cloning repo..."
  git clone "$project_boilerplate" "$project_path"

  echo "==> Removing .git directory from repo directory..."
  rm -rf "${project_path}/.git"

  echo "==> Starting DevPod..."
  local devpod_args=("$project_path")
  if [ -n "$dotfiles" ]; then
    devpod_args+=(--dotfiles "$dotfiles")
  fi

  devpod up "${devpod_args[@]}"

  echo "==> Done. Now you can run: devpod ssh ${project_name}"
}

[ $# -eq 0 ] && usage 1

parse_args "$@"
validate_data
main
```

## Działanie oraz optymalizacja
Najmniej optymalnie:
```bash
./devpod-run.sh -d git@github.com:DawidFerchow/dotfiles.git new-project /home/dawid/Projects/ git@github.com:DawidFerchow/boilerplate.git
```

ale idąc po nitce do kłębka.

Zamiast dodawać za każdym razem moje dotfiles, możemy je zdefiniować bezpośrednio w DevPod:
```bash
devpod context set-options -o DOTFILES_URL=git@github.com:DawidFerchow/dotfiles.git
```

Więc na chwilę obecną mamy:
```bash
./devpod-run.sh new-project /home/dawid/Projects/ git@github.com:DawidFerchow/boilerplate.git
```

Nadal za dużo :D

No więc, w skrypcie używam dwóch globalnych zmiennych. Folder, w którym projekt ma się utworzyć oraz link do repo, który jest boilerplate'm projektu.

Jeżeli je eksportuje np. w `~/.bashrc`, w ten sposób:

```bash
export DEVPOD_PROJECTS_DIR="/home/dawid/projects"
export DEVPOD_REPO_BOILERPLATE="https://github.com/DawidFerchow/boilerplate.git"
```

Teraz do postawienia nowego projektu wystarczy:
```bash
./devpod-run.sh new-project
```

A jeśli w konkretnym projekcie chcę przekazać swoje dotfiles:
```bash
./devpod-run.sh -d git@github.com:mischavandenburg/devpod-dotfiles my-new-api
```

Został tylko jeden problem. Muszę być w folderze, który zawiera skrypt ``devpod-run.sh``, ale na to też jest sposób.

Sam skrypt można przenieść w kilka miejsc, natomiast na moim komputerze tylko ja będe z niego korzystał, więc:
```bash
mv devpod-run.sh ~/.local/bin/devpod-run
```

oraz nadałem uprawnienia do wykonywania:
```bash
chmod u+x devpod-run
```

teraz z dowolnej lokalizacji mogę postawić sobie nowe środowisko, wywołując:
```bash
devpod-run new-project
```

Fajnie? Jeszcze jak.
## Co dzieje się w samym skrypcie?
No dobra, to teraz trochę technikaliów i elementów, którymi chciałbym się pochawlić. Nie są one potrzebne do samego działania skryptu, ale tym właśnie cechują się dobre praktyki :)
### Fail-Fast i pre-flight checks
Na początku skrypt sprawdza, czy wymagane zależności są dostępne:
```bash
check_dependencies() {
  for cmd in git devpod; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
      echo "Error: Required command '$cmd' is not installed." >&2
      exit 1
    fi
  done
}
```

Nie ma sensu zaczynać, jeżeli od początku wiadomo, że brakuje Gita albo Devpoda. Następnie sprawdzane są argumenty, ścieżka docelowa, repozytorium oraz to, czy katalog projektu już istnieje.

A to wszystko dzieje się zanim skrypt zacznie cokolwiek modyfikować.
### Konfiguracja przez zmienne środowiskowe
Ścieżka do projektów oraz domyślny boilerplate nie są wpisane bezpośrednio w kodzie. Zamiast tego skrypt korzysta z:
```
DEVPOD_PROJECTS_DIR
DEVPOD_REPO_BOILERPLATE
```

Dzięki temu nie muszę zmieniać skryptu, gdy zmieni się lokalizacja moich projektów albo repozytorium boilerplate.
### Walidacja argumentów
Przed uruchomieniem `git clone` sprawdzam również, czy przekazane repozytorium wygląda jak adres repozytorium Git:
```
local regex_git_repo='^((https?|git|ssh|file)://|git@[[:alnum:]._-]+:|/).+$'
```

Regex pozwala jednak odrzucić oczywiście niepoprawne wartości jeszcze przed rozpoczęciem operacji. Podobnie wymuszam, żeby ścieżka do katalogu projektów była absolutna:
```
if [[ ! "$path_to_project_dir" =~ ^/ ]]; then
  echo "Error: Path '$path_to_project_dir' needs to be an absolute path." >&2
  usage 1
fi
```
### Bash ma swoje SQL Injection?
Nie do końca, ale mechanizm może być podobny. Problem pojawia się, gdy zaczynamy składać polecenie jako jeden string i później pozwalamy Bashowi go ponownie zinterpretować, np. przez `eval`.

Dlatego zamiast:
```
command="devpod up $project_path --dotfiles $dotfiles"
eval "$command"
```

użyłem tablicy:
```
local devpod_args=("$project_path")

if [ -n "$dotfiles" ]; then
    devpod_args+=(--dotfiles "$dotfiles")
fi

devpod up "${devpod_args[@]}"
```

W tym przypadku każdy element tablicy jest osobnym argumentem. Bash nie musi więc zgadywać, gdzie kończy się jeden argument, a zaczyna kolejny.
### Bash Strict Mode
Na początku skryptu znajduje się tajemnicza linijka kodu:
```
set -euo pipefail
```

Jest to nic innego jak Bash Strict Mode.

W skrócie:
- `-e` - skrypt kończy działanie po błędzie,
- `-u` - użycie niezdefiniowanej zmiennej powoduje błąd,
- `pipefail` - błąd w dowolnym elemencie pipeline'u powoduje jego niepowodzenie.

Nie rozwiązuje to wszystkich problemów, ale daje mi trochę większą kontrolę nad zachowaniem skryptu.
### Stdout i stderr
Komunikaty diagnostyczne i błędy kieruję na `stderr`:
```
echo "Error: Required command '$cmd' is not installed." >&2
```

Może to mieć znaczenie, jeśli kiedyś będę chciał wykorzystać ten skrypt w innym skrypcie albo pipeline.
### Automatyczne sprzątanie po błędzie
Ciekawy jest też `trap`
```
trap cleanup EXIT INT TERM
```

Jeżeli coś pójdzie nie tak, na przykład straci połączenie sieciowe podczas `git clone` albo przerwę operację za pomocą `Ctrl+C` - skrypt może posprzątać po sobie utworzony katalog.

Służy do tego:
```
cleanup() {
  local exit_code=$?
  if [ $exit_code -ne 0 ] && [ -n "${project_path:-}" ] && [ -d "$project_path" ]; then
    echo -e "\n==> Cleaning up incomplete project directory: $project_path" >&2
    rm -rf "$project_path"
  fi
}
```

Jeżeli skrypt się wywali, nie zostanie mi więc pusty albo częściowo sklonowany katalog.
## Co można jeszcze poprawić?
Na ten moment skrypt robi dokładnie to, czego od niego chciałem. Ale pomysłów na rozwój już trochę mam.
### Automatyczne `git init`
Obecnie po sklonowaniu boilerplate usuwam jego katalog `.git`:

```
rm -rf "${project_path}/.git"
```

Ma to sens, ponieważ nie chcę, żeby nowy projekt był powiązany z repozytorium boilerplate. To wydaje mi się sensowną zmianą i prawdopodobnie trafi do kolejnej wersji skryptu.

Nie jestem natomiast przekonany do automatycznego tworzenia pierwszego commita. Samo `git init` nie podejmuje za mnie decyzji o tym, kiedy projekt jest gotowy do pierwszego commita.
### GitHub CLI
Kusi mnie też integracja z GitHub CLI.

Na przykład możliwość wykonania:
```
devpod-run my-new-project --github
```

która automatycznie wykonałaby:
```
gh repo create
```

i utworzyła zdalne repozytorium.

Nie każdy eksperyment, test czy projekt musi od razu trafiać na GitHuba. Czasami chcę po prostu odpalić środowisko, coś sprawdzić i po godzinie usunąć cały katalog.
### Testy
Skrypt jest stosunkowo prosty, ale pojawia się już kilka elementów, które warto przetestować:
- brak wymaganych argumentów,
- brak `git` lub `devpod`,
- istniejący katalog docelowy,
- niepoprawna ścieżka,
- niepoprawny adres repozytorium,
- poprawne przekazywanie `--dotfiles`,
- poprawne używanie wartości z `DEVPOD_*`,
- sprzątanie po nieudanym `git clone`,
- przerwanie działania przez `Ctrl+C`.

Do napisania są więc zarówno testy jednostkowe, jak i integracyjne. Może to będzie kolejny wpis na bloga, kto wie :p
## Podsumowanie
Nie jest to żadne przełomowe narzędzie, które zmieni bieg historii. To tylko kilkadziesiąt linii Basha, które rozwiązują bardzo mały, ale powtarzalny problem z mojego codziennego workflow.

Pięć komend samo w sobie nie jest żadnym problemem. Ale jeżeli wykonujesz je kilka razy dziennie, zaczynają się pojawiać literówki, pomyłki i zwyczajna strata czasu.

Ostatecznie cały proces, który wcześniej wymagał:
```
mkdir
cd
git clone
rm -rf .git
devpod up
```

teraz wygląda tak:
```
devpod-run my-new-project
```

A skoro coś robię regularnie i za każdym razem dokładnie w ten sam sposób, to chyba jest to całkiem dobry kandydat do automatyzacji.

Bo po co wpisywać pięć komend, skoro można wpisać jedną

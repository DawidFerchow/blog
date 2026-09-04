---
title: "A Script to Automate Setting Up a New Environment with DevPod"
date: "2026-09-04"
slug: "script-to-automate-setting-up-a-new-environment-with-devpod"
summary: "I use DevPod every day. It works great, but every new project or experiment starts with the same boring sequence of commands in the terminal. I decided to do something about it."
---
## Problem
On a daily basis, I use DevPod, which in turn uses Development Containers. It works more or less like this: I define a devcontainer file that contains an image or a Dockerfile and starts a repeatable development environment.

Thanks to this, I can use, for example, different PHP versions on the same system. It works great, but there is one small problem.

Every new project or experiment starts with the same boring sequence in the terminal. Something like this:

```bash
# 1. Create a directory
mkdir my-app

# 2. Go into it
cd my-app

# 3. Clone the project template (boilerplate)
git clone boilerplate-repo.

# 4. Remove the git directory related to the boilerplate
rm -rf .git

# 5. Start the development container with my own dotfiles
devpod up . --dotfiles your-dotfiles-repo
```

Step 5 can be improved by defining a variable directly in DevPod:

```bash
devpod context set-options -o DOTFILES_URL=your-dotfiles
```

but I still have to type 5 commands.

Doing this once or twice a week is not a problem. However, when I have to do it 3 times a day, it is simply a waste of time, which can also increase because of things like typos.

## Solution - Bash script
I decided to write a script that would turn those 5 commands into just one. Why not? :D

It ended up looking like this:

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

## How it works and optimization
The least optimized way:
```bash
./devpod-run.sh -d git@github.com:DawidFerchow/dotfiles.git new-project /home/dawid/Projects/ git@github.com:DawidFerchow/boilerplate.git
```

but let's follow this step by step.

Instead of adding my dotfiles every time, we can define them directly in DevPod:
```bash
devpod context set-options -o DOTFILES_URL=git@github.com:DawidFerchow/dotfiles.git
```

So now we have:
```bash
./devpod-run.sh new-project /home/dawid/Projects/ git@github.com:DawidFerchow/boilerplate.git
```

Still too much :D

So, in the script I use two global variables. One for the directory where the project should be created and one for the repository that is the project boilerplate.

If I export them, for example in `~/.bashrc`:
```bash
export DEVPOD_PROJECTS_DIR="/home/dawid/projects"
export DEVPOD_REPO_BOILERPLATE="https://github.com/DawidFerchow/boilerplate.git"
```

Now creating a new project only requires:
```bash
./devpod-run.sh new-project
```

And if I want to use my dotfiles for a specific project:
```bash
./devpod-run.sh -d git@github.com:mischavandenburg/devpod-dotfiles my-new-api
```

There is only one problem left. I have to be in the directory that contains the `devpod-run.sh` script. But there is also a way to fix this.

The script can be moved to several locations, but on my computer only I will use it, so:
```bash
mv devpod-run.sh ~/.local/bin/devpod-run
```

and I gave it execute permissions:
```bash
chmod u+x devpod-run
```

Now, from any location, I can create a new development environment by running:
```bash
devpod-run new-project
```

Cool? Definitely.

## What happens inside the script?
Okay, now for some technical details and things I would like to show off a little. They are not required for the script to work, but this is what good practices are about :)

### Fail-Fast and pre-flight checks
At the beginning, the script checks if the required dependencies are available:

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

There is no point in starting if we already know that Git or DevPod is missing. Then the script checks the arguments, target path, repository, and whether the project directory already exists. And all of this happens before the script starts modifying anything.

### Configuration using environment variables
The project path and default boilerplate are not hardcoded in the code.

Instead, the script uses:
```text
DEVPOD_PROJECTS_DIR
DEVPOD_REPO_BOILERPLATE
```

Thanks to this, I don't have to change the script when the location of my projects or the boilerplate repository changes.

### Argument validation
Before running `git clone`, I also check if the provided repository looks like a Git repository address:
```bash
local regex_git_repo='^((https?|git|ssh|file)://|git@[[:alnum:]._-]+:|/).+$'
```

The regex can reject obviously invalid values before starting the operation. I also require the path to the projects directory to be absolute:
```bash
if [[ ! "$path_to_project_dir" =~ ^/ ]]; then
  echo "Error: Path '$path_to_project_dir' needs to be an absolute path." >&2
  usage 1
fi
```

### Does Bash have its own SQL Injection?
Not exactly, but the mechanism can be similar. The problem appears when we start building a command as one string and then allow Bash to interpret it again, for example with `eval`.

For example:
```bash
command="devpod up $project_path --dotfiles $dotfiles"
eval "$command"
```

Instead, I used an array:
```bash
local devpod_args=("$project_path")

if [ -n "$dotfiles" ]; then
    devpod_args+=(--dotfiles "$dotfiles")
fi

devpod up "${devpod_args[@]}"
```

In this case, every element of the array is a separate argument. Bash does not have to guess where one argument ends and another one starts.

### Bash Strict Mode
At the beginning of the script there is a mysterious line of code:
```bash
set -euo pipefail
```

It is nothing more than Bash Strict Mode.

In short:
- `-e` - the script stops after an error,
- `-u` - using an undefined variable causes an error,
- `pipefail` - an error in any part of a pipeline causes the whole pipeline to fail.

It does not solve every problem, but it gives me a little more control over how the script behaves.

### Stdout and stderr
I send diagnostic messages and errors to `stderr`:
```bash
echo "Error: Required command '$cmd' is not installed." >&2
```

This can be useful if I want to use the script inside another script or pipeline in the future.

### Automatic cleanup after an error
Another interesting part is `trap`:
```bash
trap cleanup EXIT INT TERM
```

If something goes wrong, for example I lose the network connection during `git clone` or stop the operation with `Ctrl+C`, the script can clean up the directory it created.

This is done by:
```bash
cleanup() {
  local exit_code=$?
  if [ $exit_code -ne 0 ] && [ -n "${project_path:-}" ] && [ -d "$project_path" ]; then
    echo -e "\n==> Cleaning up incomplete project directory: $project_path" >&2
    rm -rf "$project_path"
  fi
}
```

If the script fails, I will not be left with an empty or partially cloned project directory.

## What can be improved?
At the moment, the script does exactly what I wanted it to do.

But I already have a few ideas for future improvements.

### Automatic `git init`
Currently, after cloning the boilerplate, I remove its `.git` directory:
```bash
rm -rf "${project_path}/.git"
```

This makes sense because I don't want the new project to be connected to the boilerplate repository. This seems like a reasonable change and will probably be added in the next version of the script.

However, I am not sure about automatically creating the first commit. `git init` does not make a decision for me about when the project is ready for its first commit.
### GitHub CLI

I am also tempted to integrate GitHub CLI.

For example, the possibility to run:
```bash
devpod-run my-new-project --github
```

which would automatically run:
```bash
gh repo create
```

and create a remote repository.

Not every experiment, test, or project needs to go to GitHub immediately. Sometimes I just want to start an environment, test something, and delete the whole directory an hour later.

### Tests
The script is relatively simple, but there are already several things worth testing:

- missing required arguments,
- missing `git` or `devpod`,
- existing target directory,
- invalid path,
- invalid repository address,
- correct passing of `--dotfiles`,
- correct use of `DEVPOD_*` values,
- cleanup after a failed `git clone`,
- stopping the script with `Ctrl+C`.

So there are both unit and integration tests to write. Maybe this will be another blog post. Who knows :p

## Summary
This is not some groundbreaking tool that will change the course of history. It is just a few dozen lines of Bash that solve a very small but repetitive problem in my daily workflow.

Five commands are not a problem by themselves. But if you run them several times a day, typos, mistakes, and simple time loss start to appear.

In the end, the whole process, which previously required:
```bash
mkdir
cd
git clone
rm -rf .git
devpod up
```

now looks like this:
```bash
devpod-run my-new-project
```

And if I do something regularly and in exactly the same way every time, it seems like a pretty good candidate for automation.

Because why type five commands when you can type one?

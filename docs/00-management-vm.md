# Step 00: Setup Management VM

This step prepares a separate workspace on the existing R2D2 management VM. It does not reinstall Ubuntu, install Kolla-Ansible, or touch the physical nodes. If R2D2 is not your management VM, verify your own host and paths before using these lab-specific examples.

If your prompt looks like `(kolla-2026.2) joe@r2d2:~/fishbone$`, `(kolla-2026.2)` is the old Python environment and `~/fishbone` is the old working directory. Neither means the reinstalled nodes are running the old cloud. Keep the previous files intact while preparing the new build.

| Purpose | Fresh-build location |
|---|---|
| Working directory | `~/openstack-fresh` |
| Python environment | `~/.venvs/kolla-fresh` |
| Intended Kolla configuration | `~/openstack-fresh/kolla` |

**Important:** R2D2 also has an existing `/etc/kolla`. Kolla defaults to that directory unless told otherwise. A new working directory or Python environment does **not** automatically isolate the configuration or passwords. Preserve `/etc/kolla`, the old environment and `~/fishbone`; do not delete, rename, overwrite or use their credentials for the new build without a separate review.

The following creates an empty workspace and Python environment **on R2D2 only**. It does not install Kolla or make node changes. Run each command separately and stop if one fails. Before starting, check that the new paths do not already exist, including as symlinks:

```bash
for p in "$HOME/openstack-fresh" "$HOME/.venvs/kolla-fresh"; do
  if [ -e "$p" ] || [ -L "$p" ]; then
    printf 'EXISTS: %s; stop and inspect it.\n' "$p"
  else
    printf 'Absent: %s\n' "$p"
  fi
done
```

Continue only if **both** paths are absent. Check the parent environment directory without changing it:

```bash
if [ -d "$HOME/.venvs" ] && [ ! -L "$HOME/.venvs" ]; then
  stat -c '%n owner=%U mode=%a' "$HOME/.venvs"
else
  printf '%s\n' 'STOP: ~/.venvs is missing, a symlink, or not a directory; inspect it first.'
fi
```

If the owner is not your account or the parent needs to be created, stop and resolve that separately. Then create the new directories:

```bash
mkdir -m 700 "$HOME/openstack-fresh"
```

```bash
mkdir -m 700 "$HOME/openstack-fresh/kolla"
```

Create the empty environment:

```bash
mkdir -m 700 "$HOME/.venvs/kolla-fresh" &&
  /usr/bin/python3 -m venv "$HOME/.venvs/kolla-fresh"
```

If any creation step fails, stop. A partial directory may remain; inspect it before retrying. The environment contains Python tooling, but Kolla-Ansible is **not installed** by this command. Open a new terminal, then activate the new environment and enter the new directory:

```bash
source "$HOME/.venvs/kolla-fresh/bin/activate" &&
cd "$HOME/openstack-fresh" &&
printf 'Python environment: %s\nWorking directory: %s\n' "$VIRTUAL_ENV" "$PWD"
```

The displayed environment should end in `/.venvs/kolla-fresh`, and the working directory in `/openstack-fresh`. If not, stop. The old prompt may still appear in other terminal windows; close or deactivate those sessions before working on the new build.

## Path contract for this fresh build

All Kolla commands in the remaining guide use the new R2D2 workspace, not the historical `/etc/kolla` tree or the old `kolla-2026.2` virtual environment. At the start of every new R2D2 terminal, run:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
printf 'VENV=%s\nCONFIG=%s\nINVENTORY=%s\n' "$VIRTUAL_ENV" "$KOLLA_CONFIG_PATH" "$KOLLA_INVENTORY"
```

Expected paths are:

```text
VENV=/home/joe/.venvs/kolla-fresh
CONFIG=/home/joe/openstack-fresh/kolla
INVENTORY=/home/joe/openstack-fresh/kolla/multinode
```

The historical `/etc/kolla`, old virtual environment and `~/fishbone` remain out of scope. Do not copy their passwords, inventory or `clouds.yaml` into the fresh workspace.

## Next step

Continue with [Step 1: Install and identify the systems](01-foundations.md#step-1-install-and-identify-the-systems). The reinstalled nodes must be identified and checked; do not reinstall them again merely to follow this guide.

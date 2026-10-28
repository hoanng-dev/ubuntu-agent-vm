# ubuntu-agent-vm

One script that turns a fresh Ubuntu VM (tested on Ubuntu 24.04 under Hyper-V) into a box where a
coding agent such as Claude Code can run fully unattended as root.

> **Warning — disposable VMs only.** This script deliberately removes safety rails: root login with
> a password is enabled, all automatic updates (including security updates) are turned off, and
> Claude Code is allowed to run `--dangerously-skip-permissions` as root. Use it only on throwaway
> VMs that you can rebuild, never on a machine that holds real data or faces the internet.

## What it does

The script is idempotent (safe to run again) and verifies itself at the end.

1. **Updates off:** masks `unattended-upgrades`, the apt timers, `packagekit` and `snapd`, and turns
   off the release-upgrade prompt and motd news.
2. **Root access:** optional root password, `PermitRootLogin yes`, password auth on, optional SSH
   public key.
3. **Disk:** grows the root partition, LVM volume and filesystem (ext4 or xfs) to the whole disk,
   online.
4. **Internal network:** gives the second NIC (the one without the default route, e.g. a Hyper-V
   internal switch) a static IP through netplan, matched by MAC. Cloud-init network config is
   disabled so it survives reboots.
5. **Claude Code:** installs the latest native build for root, links `/usr/local/bin/claude`, sets
   `IS_SANDBOX=1` so `--dangerously-skip-permissions` works as root, and turns off auto-update.
6. **Verify:** prints PASS/FAIL for every item. Exit code 0 = all passed, 1 = a check failed,
   2 = bad arguments.

## Usage

Run on the target VM as root. Read the root password silently so it never lands in the shell
history:

```bash
read -rsp 'Root password: ' ROOT_PASSWORD; echo; export ROOT_PASSWORD
curl -fsSL https://raw.githubusercontent.com/hoanng-dev/ubuntu-agent-vm/main/setup-ubuntu-agent-vm.sh \
  | bash -s -- --internal-ip 10.0.0.10/24
```

Options:

| Option | Meaning |
|---|---|
| `--internal-ip <ip/prefix>` | Static IP for the internal NIC. Required unless `--no-internal`. Use a different IP per VM. |
| `--internal-if <name>` | Pick the internal NIC by name when auto-detection finds more than one. |
| `--ssh-key '<public key>'` | Add a public key to `/root/.ssh/authorized_keys`. |
| `--root-password <pw>` | Set the root password (prefer the `ROOT_PASSWORD` env var shown above). |
| `--no-internal` | Skip the internal network step. |
| `--no-claude` | Skip the Claude Code step. |

When it finishes, run `claude` once as root to log in.

### Pin a version

`main` always serves the newest script. To run exactly the code you reviewed, replace `main` in the
URL with a commit SHA from the history of this repository.

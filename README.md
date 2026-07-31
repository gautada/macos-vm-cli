# macos-vm-cli

A Mac assed virtual machine cli toolset.  The idea is to have a set of
utilities and commands that make virtual machine definiition and management
easier on macOS.

## Pre-Requisite

- macOS on Apple Silicon (scripts assume `hvf` acceleration and an aarch64 guest)
- [Homebrew](https://brew.sh), with `qemu` and `jq` installed
  (`brew install qemu jq`)
- An SSH key pair at `~/.config/ssh/id_ed25519.pub` — the public key is baked
  into every VM's cloud-init config so you can log in over SSH
- `sudo` access — **only if you use `NETMODE=bridged`** (see below). It's
  used once to create/own `/Users/Shared/qemu`, and again to run
  `qemu-system-aarch64` itself (it needs raw access to `vmnet-bridged`
  networking). `NETMODE=user` VMs need no `sudo` at all, ever.

## Concept

Each VM is built as a single, self-contained, mountable **sparsebundle**
disk image — a macOS disk image made of many small band files, functionally
just a folder on disk, but one that can be attached like a real volume.

Everything the VM needs to run is stored *inside* that bundle: the base OS
image (only during the build/bake, not afterward — see "Initializing a VM"
below), the virtual hard disk, UEFI firmware/vars, and (only during the
build) the cloud-init seed ISO. Nothing the VM needs lives outside the
bundle, so it can be copied, backed up, or moved to another Mac and it will
still work.

The bundle is only mounted while something is actively using it:
- `vm-init` mounts it at `/Volumes/<name>` while it builds the VM's files,
  then detaches it when done.
- `vm-start` mounts it at `/Volumes/<name>` and leaves it mounted for as
  long as the VM is running — qemu itself runs headlessly in the
  background, so the volume stays attached after `vm-start` returns.
  `vm-halt` or `vm-stop` detaches it again once the VM actually stops.

At rest (VM not built, or not running), the bundle is just an inert file
on disk — nothing is mounted, and `/Volumes/<name>` doesn't exist.

### Two network modes, two locations

Every VM is built in one of two networking modes, chosen per-VM by the
`NETMODE` field in its config file (see below). This determines both how
the guest reaches the network **and** where the VM's files live on disk:

| `NETMODE`          | Networking                          | `sudo` needed?    | Where it lives                    |
|--------------------|--------------------------------------|-------------------|------------------------------------|
| `bridged` (default) | `-netdev vmnet-bridged` onto a real host interface (`en1`), with a static IP/gateway you configure | Yes, throughout   | `/Users/Shared/qemu/vms/<name>.sparsebundle` |
| `user`              | qemu's own usermode/SLIRP networking (`-netdev user`), with an SSH port forwarded to the host | Never              | `<this repo>/build/vms/<name>.sparsebundle`  |

**`bridged`** is the original design: the guest gets a real, routable IP on
your LAN via a host interface, which needs raw network access — that's why
qemu has to run under `sudo`, and why everything qemu creates while running
(the qmp/qga/serial sockets, in particular) ends up root-owned. Every
control script — including a read-only status check — then also needs
`sudo` just to read those sockets. Shared, host-wide state for this mode
lives at `/Users/Shared/qemu` (`vms/`, `cache/`, `run/`), set up once (with
`sudo`) by the first `vm-init` run.

**`user`** is for when you don't want (or can't get) `sudo` at all — e.g.
scripted testing, CI, or a debug loop where nobody can type a password
interactively. qemu's built-in SLIRP networking needs no special privilege
whatsoever, so qemu runs entirely as your own plain user, nothing it
creates is ever root-owned, and no script in this toolkit ever invokes
`sudo` for a `user`-mode VM. Because there's no shared, privileged location
involved, `user`-mode VMs live entirely inside this repo checkout, under
`build/` (`build/vms/`, `build/cache/`, `build/run/` — mirroring the
bridged layout, just relocated and unprivileged). `build/` is gitignored;
treat it as disposable, ephemeral state, the same way `/Users/Shared/qemu`
is for bridged VMs.

SLIRP has no notion of the host reaching in to the guest by IP, so
`user`-mode VMs get an SSH port forwarded from the host instead: guest port
22 is reachable at `localhost:<20000 + INSTANCE>` (e.g. `INSTANCE=02` →
`localhost:20002`). `vm-init` and `vm-start` both print the exact port on
every boot.

Because `vm-init` is given a config file directly, it always knows a VM's
`NETMODE` up front. The `vm-<verb>` control scripts (`vm-start`,
`vm-status`, ...), though, only ever receive a bare VM **name** — they
figure out which of the two locations (and therefore which mode) a given
name lives in by checking both (see `resolve_vm` in `bin/vm-lib`), and
fail if a name exists in neither, or — a genuine naming collision — in
both. Every socket/process interaction in `bin/vm-lib` (talking to QMP,
the guest agent, killing a process) then decides for itself, per call,
whether `sudo` is actually needed by checking who owns the thing it's
about to touch (`root` → prefix `sudo`; anyone else, including yourself →
don't) — see `path_owner`/`pid_owner`/`run_as` in `bin/vm-lib`. This is
what lets a single set of scripts serve both modes without every caller
needing to know or pass the mode through explicitly.

## Configuration file

A VM is described by a small config file (see [`example.vmc`](example.vmc)
for `NETMODE=bridged`, or [`example-nosudo.vmc`](example-nosudo.vmc) for
`NETMODE=user`):

```
NAME=jupiter
DOMAIN=gautier.org
INSTANCE=03
MEMORY=8G
DISK=80G
CPUS=4
MAC=52:54:00:07:00:03
IP=10.0.7.3/22
GATEWAY=10.0.4.1
OS=24.04
NETMODE=bridged
```

| Field      | Meaning                                                              |
|------------|-----------------------------------------------------------------------|
| `NAME`     | VM name — also the bundle filename, mount point, and hostname prefix |
| `DOMAIN`   | Domain suffix used to build the guest's FQDN                          |
| `INSTANCE` | Instance number, appended to `NAME` for the hostname/FQDN (and, in `NETMODE=user`, used to derive the forwarded SSH port) |
| `MEMORY`   | RAM given to the guest (qemu `-m` value, e.g. `8G`)                   |
| `DISK`     | Guest disk size (grown into via `qemu-img resize`, e.g. `80G`)        |
| `CPUS`     | Guest vCPU count (qemu `-smp` value)                                  |
| `MAC`       | MAC address for the guest's virtio-net interface                     |
| `IP`       | **`bridged` only.** Static IP/CIDR for the guest (netplan `addresses` entry) |
| `GATEWAY`  | **`bridged` only.** Default gateway / nameserver for the guest         |
| `OS`       | Ubuntu release to base the VM on (e.g. `24.04`), matches a cloud image |
| `NETMODE`  | `bridged` (default) or `user` — see "Two network modes" above. Optional: a config file without this field is treated as `bridged`, for compatibility with configs written before this field existed. |

Every field is required **except** `NETMODE` (defaults to `bridged`) and
`IP`/`GATEWAY` (only required when `NETMODE=bridged`; ignored — and fine to
omit — when `NETMODE=user`, since SLIRP hands the guest its own address
over DHCP). `vm-init` fails fast if a field that *is* required for the
chosen mode is missing or empty.

## Initializing a VM (`vm-init`)

```
bin/vm-init <config file>
```

This is a one-time build step per VM. In order, it:

1. **Validates the config file** and loads each field into a `VM_*`
   environment variable (e.g. `VM_NAME`, `VM_DISK`), applying the
   `NETMODE`-aware rules above (defaulting `NETMODE`, requiring
   `IP`/`GATEWAY` only for `bridged`).
2. **Prepares the VM's base directory** — `/Users/Shared/qemu` for
   `bridged` (created and `chown`'d with `sudo` the first time), or this
   repo's own `build/` for `user` (already yours, no `sudo` involved) —
   creating a `vms/` bundle folder and a `cache/images/` folder used to
   cache downloaded base OS images across VMs *of the same mode* (bridged
   and user-mode VMs keep entirely separate caches, since they live in
   entirely separate locations).
3. **Refuses to clobber an existing VM** — fails if
   `vms/<name>.sparsebundle` or `/Volumes/<name>` already exists.
4. **Creates the sparsebundle**, sized to the configured `DISK` plus a 15GB
   overhead for firmware/bios/cidata/flatten-headroom, formatted APFS, then **attaches** it
   at `/Volumes/<name>` for the rest of the build.
5. **Copies the config file** into the bundle as `config` (this is what
   `vm-start` reads later — the original config file is no longer needed
   once this step is done).
6. **Downloads and verifies the base Ubuntu cloud image** (arm64 server
   image for the configured `OS` release) into the mode's shared cache if
   it isn't already there, checking its SHA256 against Ubuntu's published
   checksums, then **copies the verified image into the bundle** so the
   bundle is self-contained during the bake (see step 10 — it's removed
   again once baking finishes).
7. **Creates the virtual hard disk** — a `qcow2` copy-on-write overlay
   backed by the in-bundle base image copy — and resizes it to the
   configured `DISK` size.
8. **Copies UEFI firmware and vars** (`edk2-aarch64-code.fd` /
   `edk2-arm-vars.fd` from the Homebrew qemu package) into the bundle. The
   vars file is per-VM because UEFI writes boot-variable state into it at
   runtime.
9. **Renders the cloud-init seed** (`meta-data`, `network-config`,
   `user-data` from [`templates/`](templates), picking the `bridged` or
   `user` variant of `network-config`/`user-data` — see "Cloud-init
   profiles" below) with the VM's settings and your SSH public key, and
   packages them into a `cidata.iso` inside the bundle.
10. **Bakes a golden disk image**: boots the disk *once*, with the
    cloud-init seed attached, and lets cloud-init do its one-time
    provisioning (create the user, install packages, configure networking,
    ...) for real, directly onto the disk. Once the guest agent confirms
    it's up (see "Diagnosing a hung boot" below for how this wait is
    implemented and why), this step shuts the guest down again and
    flattens the disk with `qemu-img convert` so it no longer depends on
    the base cloud image as a backing file. The base image copy and the
    `cidata.iso` are then deleted from the bundle — neither is needed
    again, since every future boot (`vm-start`) just boots the
    already-provisioned golden disk directly, with **no cloud-init
    involved at all**.
11. **Detaches the bundle**, leaving a ready-to-run
    `vms/<name>.sparsebundle` and printing the path plus a reminder to run
    `vm-start`.

If any step fails partway through, a trap detaches the volume (stopping the
bake-boot qemu process first, if one is still running) so a failed build
doesn't leave a stray mount or a stray qemu process behind.

Once built, the bundle's volume contains:

```
/Volumes/<name>/
  config              # copy of the .vmc config file
  virtual-hard-disk   # qcow2 disk, flattened (no backing file) after baking
  firmware            # UEFI code (read-only)
  bios                # UEFI vars (read/write, per-VM)
```

(`images/` and `cidata.iso` only exist transiently, during the bake — see
step 10 above.)

### Cloud-init profiles

`network-config` and `user-data` both come in two variants, picked by
`NETMODE` (`envsubst`, which renders these templates, has no
conditionals, so this can't be a single parameterized template):

| `NETMODE`  | `network-config` template          | `user-data` template                  |
|------------|-------------------------------------|-----------------------------------------|
| `bridged`  | `templates/network-config` — static IP/GATEWAY via netplan | `templates/user-data` — full package list, `package_update`/`package_upgrade: true` |
| `user`     | `templates/network-config-dhcp` — SLIRP hands out an address over DHCP | `templates/user-data-fast` — trimmed package list (guest agent, SSH, `curl`/`jq`/`ping` only), `package_update`/`package_upgrade: false` |

The trimmed `user`-mode profile is a deliberate choice: `NETMODE=user`
exists to make a fast, no-`sudo` debug loop possible, so a bake/boot cycle
should take on the order of **1-2 minutes**, not the 5-10+ minutes a full
`apt` update+upgrade+full-package-list bake takes. `bridged` keeps full
production fidelity, unchanged.

## Starting a VM (`vm-start`)

```
bin/vm-start <name>
```

This is headless and returns control to your shell prompt once the VM is
up — it does not stay attached to the VM's console. In order, it:

1. Refuses to start if the VM already appears to be running (a live
   pidfile), but reclaims a stale mount at `/Volumes/<name>` left behind
   by an unclean previous exit (e.g. the guest was powered off directly
   instead of via `vm-halt`/`vm-stop`).
2. Attaches `vms/<name>.sparsebundle` (found via `resolve_vm` — see "Two
   network modes" above) at `/Volumes/<name>` and sources the VM's
   `config` from it.
3. **`NETMODE=bridged` only**: authenticates `sudo` once up front and
   keeps the ticket refreshed in the background for the rest of the
   script, so a slow boot doesn't hit repeated password prompts.
   **`NETMODE=user`**: none of this happens — nothing in this path ever
   calls `sudo`.
4. Launches `qemu-system-aarch64` — under `sudo` for `bridged`, as the
   plain user for `user` — backgrounded via `nohup ... & disown` (see
   "Known gotchas" below for why not `-daemonize`). `-display none` /
   `-monitor none` keep it fully headless — there's no interactive
   console attached to this terminal at all (the serial console is still
   reachable over a socket and logged to a file — see "Diagnosing a hung
   boot" below).
5. Waits for the guest agent to respond (see "Diagnosing a hung boot"
   below for how), up to a 180s budget (override with `VM_START_TIMEOUT`),
   printing progress dots. Ctrl-C during this wait just stops waiting —
   the VM keeps running in the background either way.
6. Reports whether the guest responded in time and returns to the
   prompt. **The volume stays mounted** — qemu is still using files
   there. `bin/vm-halt` or `bin/vm-stop` detach it once the VM actually
   stops (see below).

qemu's own stderr from the startup phase is captured to `qemu.log` for
debugging if something goes wrong.

Runtime state for the running qemu process — the qmp/qga/serial control
sockets, a pidfile (written by the script itself, as the plain user — see
"Known gotchas"), and logs — is host-side state, not part of the portable
VM bundle, so it's kept separately, alongside the VM's bundle (i.e. under
`/Users/Shared/qemu/run/<name>/` for `bridged`, `build/run/<name>/` for
`user`):

```
run/<name>/
  pid
  qmp.sock
  qemu-ga.sock
  serial.sock       # guest serial console socket, for interactive debugging
  serial.log        # persistent log of everything written to the serial console
  qemu.log          # qemu's own stdout/stderr
```

## Controlling a running VM

Once a VM is running (`bin/vm-start <name>` in its own terminal), these
scripts control it from any other terminal by talking to its QMP socket:

| Script                    | Effect                                                                 |
|---------------------------|-------------------------------------------------------------------------|
| `bin/vm-status <name>`    | Report running/not-running, and the guest's run state (running/paused) |
| `bin/vm-halt <name>`      | Ask the guest to shut down gracefully (ACPI power button)              |
| `bin/vm-stop <name>`      | Force-stop qemu immediately — no graceful guest shutdown               |
| `bin/vm-restart <name>`   | Hard-reset the guest (like pressing a physical reset button)           |
| `bin/vm-pause <name>`     | Suspend guest execution in place                                       |
| `bin/vm-resume <name>`    | Resume a paused guest                                                  |

Notes:

- **`vm-halt` vs `vm-stop`**: `vm-halt` sends an ACPI shutdown request and
  waits (up to 120s, override with `VM_HALT_TIMEOUT`) for the guest OS to
  shut itself down cleanly; `vm-stop` terminates qemu immediately
  (waiting up to 30s, override with `VM_STOP_TIMEOUT`), equivalent to
  pulling the power, and should only be used when the guest is
  unresponsive.
- **`vm-restart`** reboots the guest in place (`system_reset`) — it does
  not stop and re-run `vm-start`. For a graceful reboot, use `vm-halt`
  and then `vm-start` again once `vm-status` shows it has stopped.
- **`vm-halt` and `vm-stop` own detaching the bundle**: since `vm-start`
  returns as soon as the guest is up rather than staying attached for the
  VM's whole life, it's whichever script actually ends the VM that waits
  for qemu to actually exit and detaches `/Volumes/<name>` at that point.
  If the guest is shut down some other way (e.g. `sudo poweroff` from
  inside the guest instead of `vm-halt`), the mount is left stale until
  the next `bin/vm-start` of that VM, which detects and reclaims it.
- **`sudo` is automatic and per-call, not per-mode**: none of these
  scripts hardcode whether they need `sudo` — every one of them, via
  `bin/vm-lib`, checks who actually owns the socket (or process) it's
  about to touch and only prefixes `sudo` if that's `root` and you
  aren't already. For a `bridged` VM (root-owned, since qemu runs under
  `sudo`) that means every one of these — including the read-only
  `vm-status` — may prompt for your password. For a `user`-mode VM
  (everything plain-user-owned) none of them ever will.
- All of them share `bin/vm-lib` for the common QMP/guest-agent plumbing
  and the bare-name-to-location resolution described above; it's a
  library meant to be sourced, not run directly.
- In `NETMODE=user`, the guest's SSH port is forwarded to
  `localhost:<20000 + INSTANCE>` (SLIRP has no other way to reach in) —
  `vm-start` prints the exact port every time it boots the VM.

## End-to-end smoke test (`vm-fulltest`)

```
bin/vm-fulltest [config file]
```

Defaults to `example.vmc` (a disposable `bridged` VM named `vm`); pass
`example-nosudo.vmc` to exercise the `user`-mode path end to end with zero
`sudo` prompts. It tears down any existing VM of that name (stop, detach,
delete the bundle and run directory), builds a fresh one with `vm-init`,
starts it, and exercises every control script above against it, asserting
the expected state at each step — including the golden-image properties
(flattened disk, no `cidata.iso`, no leftover base-image copy) and, across
a `vm-restart`, that the qemu process itself never changed pid (only the
guest rebooted). Prints a PASS/FAIL summary and exits 0 only if everything
passed.

## Diagnosing a hung boot

This project went through a real, reproducible-feeling hang early on:
`qemu-guest-agent` would sometimes never answer a `guest-ping`, even after
waiting 900+ seconds, with `qemu.log` completely silent and no obvious
cause. The actual root cause (found from a leftover `serial.log` of one
such hang) turned out to have nothing to do with the guest's boot process
itself — the guest had fully booted, cloud-init had finished, and
`qemu-guest-agent.service` had reported `Started`, all within about 30
seconds; the console then went completely silent for the rest of the wait.
The real problem was on the *host* side: the old polling loop reconnected
a brand-new `nc` client to the guest agent's virtio-serial socket on every
single attempt, and that repeated connect/disconnect churn — especially
happening before the guest had booted far enough for `qemu-ga` to actually
open its end of the port — could leave the port permanently desynced for
the rest of that boot, with the guest-side daemon never becoming
reachable again no matter how long (or how many more times) you kept
polling. Widening the timeout (180s → 400s → 900s) never fixed it, because
it was never a "needs more time" problem.

The fix, `wait_for_qga_ready` in `bin/vm-lib`, holds a **single, long-lived
connection** to the guest agent socket for the entire wait instead of
reconnecting per attempt, so the guest only ever sees one connect event.
It's used by `vm-init`'s bake-boot wait, `vm-start`'s boot wait, and
`vm-fulltest`'s post-`vm-restart` wait — anywhere this toolkit polls for
the guest agent to come up.

If you ever do hit a genuinely stuck boot, `run/<name>/serial.log` is
where to look first: it's a **persistent, plain-file** capture of the
guest's actual console output (kernel messages, systemd unit
starts/stops, cloud-init's own output, login prompts), pre-created by the
plain user and `chmod 644`'d *before* qemu ever opens it — specifically so
it stays readable without `sudo` even for a `bridged` VM, and survives
after qemu itself is killed. Compare it against `run/<name>/qemu.log`
(qemu's own stdout/stderr — silence there just means qemu itself didn't
crash, it says nothing about the guest). For a live guest, you can also
connect directly to the socket right next to it —
`nc -U run/<name>/serial.sock` (add `sudo` for a `bridged` VM) — to get an
interactive console, including a login prompt if the guest reached one.

## Known gotchas

- **`-daemonize` crashes.** qemu's own `-daemonize` flag forks internally
  *after* HVF/vmnet have already spun up threads, and macOS's
  Objective-C runtime deliberately aborts rather than risk an unsafe fork
  mid-initialization (visible as `objc[...]: +[NSNumber initialize] ...
  Crashing instead` in `qemu.log`). The fix is shell-level backgrounding
  instead (`nohup ... & disown`), which forks at the OS level *before*
  qemu's process image and its threads exist, avoiding the crash.
- **`sudo`'s child model**: on macOS, `sudo cmd &` does NOT simply
  exec-replace itself with `cmd` — it runs a monitor process that forks
  `cmd` as a child. So `$!` right after `sudo cmd &` is the monitor's
  pid, not `cmd`'s own pid directly. This is fine for `wait`/`ps -p`
  liveness checks (the monitor only exits after its child does), and
  `run_as`/`kill` calls against that pid work too (the monitor forwards
  signals to its child) — but don't assume `$!` is qemu's literal pid in
  a `sudo`-launched (`bridged`) path. In `NETMODE=user` (no `sudo`), `$!`
  *is* the real qemu pid directly.
- **Root-owned sockets need `sudo` even to read.** In `bridged` mode,
  qemu runs under `sudo`, so the qmp/qga/serial socket files it creates
  end up root-owned with no group/other write bit — even a read-only
  status check needs `sudo nc -U` to connect at all (macOS enforces
  permission checks on `AF_UNIX` `connect()`). `bin/vm-lib` handles this
  automatically per-call (see "Controlling a running VM" above); it's
  exactly what `NETMODE=user` was built to sidestep entirely.
- **`kill -0` is not liveness-safe against another user's process.**
  `kill(pid, 0)` checks *permission to signal*, not just existence — a
  plain user's `kill -0` against a root-owned pid returns `EPERM`
  (looking exactly like "it's alive but not yours"), not "it's gone".
  Every liveness check in this toolkit therefore uses `ps -p PID`
  instead, which just checks the process table with no permission
  barrier either way.
- **Pidfiles**: qemu's own `-pidfile` option, under `sudo` (`bridged`),
  produces a root-owned, `0600` pidfile the plain user can't read back
  later. Every script here writes its own pidfile instead, as the plain
  user, right after backgrounding, using `$!` — don't reintroduce qemu's
  own `-pidfile`.
- **`bin/vm-init`'s `load_vm_variable`/`load_vm_variable_default`
  functions use bash's `${!var}` indirect-expansion** — this relies on
  macOS's `/bin/sh` actually being bash under the hood; it is not
  portable POSIX sh, but it's a deliberate existing choice, not a bug to
  "fix" into POSIX purity.
- **The base-image checksum is genuinely verified** — `vm-init` fails the
  build outright on a mismatch (an earlier version of this check was
  silently a no-op due to an unquoted shell variable inside a
  single-quoted `awk` script). Don't reintroduce that bug.
- **Cloud-init's SSH key placeholder name matters.** `templates/user-data*`
  reference `${VM_SSH_PUBLIC_KEY}`; `vm-init` must export exactly that
  name before rendering them with `envsubst`. `envsubst` silently
  substitutes an *unset* variable with an empty string rather than
  failing, so a name mismatch here previously produced a guest with no
  authorized key at all, with no error anywhere to point at it — the
  only symptom was `ci-info: no authorized SSH keys fingerprints found`,
  buried in `serial.log`.

## Bridged mode: verified structurally, not yet boot-tested

`NETMODE=bridged` cannot be exercised by an automated/no-interactive-sudo
session (there's no way to supply a sudo password non-interactively, and
this project deliberately does not work around that). Its code paths were
carefully kept structurally parallel to the now-verified `user` mode (same
`resolve_vm`/`run_as` logic, same qemu invocation shape, just
`vmnet-bridged` + `sudo` instead of `-netdev user` + none) and reviewed by
hand, but an actual `bin/vm-fulltest example.vmc` run — with a real sudo
password typed in when prompted — is still owed before treating it as
confirmed working.

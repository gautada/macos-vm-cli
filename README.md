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
- `sudo` access — **only if you use `NETMODE=bridged`** (see below), to run
  `qemu-system-aarch64` itself (it needs raw access to `vmnet-bridged`
  networking) and to talk to the sockets it creates while running.
  `NETMODE=user` VMs need no `sudo` at all, ever, for anything.

## Concept

Each VM is a single, self-contained, mountable **sparsebundle** disk image
with a `.vmsb` extension — a macOS disk image made of many small band files,
functionally just a folder on disk, but one that can be attached like a real
volume. Everything the VM needs to run is stored *inside* it: the virtual
hard disk, UEFI firmware/vars, and (only transiently, during the build) the
base OS image and cloud-init seed. Nothing the VM needs lives outside the
bundle, so it can be copied, backed up, or moved to another Mac and it will
still work.

**A VM is addressed by path when it doesn't exist yet, and by name once it's
running**:

- `bin/vm-init <config file>` creates the bundle wherever you run it from
  (the current directory), named after the config file - `vm-init
  example.vmc` from `~/Workspace/VM` creates `~/Workspace/VM/example.vmsb`.
  There's no other fixed location involved; a VM's `.vmsb` can live anywhere
  you like, including a git repo, a shared folder, wherever.
- `bin/vm-start <path to .vmsb>` takes that path directly, and mounts it at
  `/Volumes/<name>` (`<name>` being the `.vmsb`'s own basename, e.g.
  `example`) for as long as the VM runs.
- Once running, it's addressed purely **by that name** — `bin/vm-status
  example`, `bin/vm-halt example`, etc. - since `/Volumes/<name>` is a fully
  predictable location while the VM is up, none of the other control scripts
  need to know or search for where the original `.vmsb` file lives on disk.

The bundle is only mounted while something is actively using it:
- `vm-init` mounts it while it builds the VM's files, then detaches it when
  done.
- `vm-start` mounts it and leaves it mounted for as long as the VM is
  running — qemu itself runs headlessly in the background, so the volume
  stays attached after `vm-start` returns. `vm-halt` or `vm-stop` detaches it
  again once the VM actually stops.

At rest (VM not built, or not running), the bundle is just an inert file on
disk — nothing is mounted, and `/Volumes/<name>` doesn't exist.

### Two network modes

Every VM is built in one of two networking modes, chosen per-VM by the
`NETMODE` field in its config file (see below):

| `NETMODE`            | Networking                                                                                    | `sudo` needed? |
|----------------------|------------------------------------------------------------------------------------------------|----------------|
| `bridged` (default)  | `-netdev vmnet-bridged` onto a real host interface (`en1`), with a static IP/gateway you configure | Yes, throughout |
| `user`               | qemu's own usermode/SLIRP networking (`-netdev user`), with an SSH port forwarded to the host  | Never          |

**`bridged`** is the original design: the guest gets a real, routable IP on
your LAN via a host interface, which needs raw network access — that's why
qemu has to run under `sudo`, and why everything qemu creates while running
(the qmp/qga/serial sockets, in particular) ends up root-owned. Every
control script — including a read-only status check — then also needs
`sudo` just to read those sockets.

**`user`** is for when you don't want (or can't get) `sudo` at all — e.g.
scripted testing, CI, or a debug loop where nobody can type a password
interactively. qemu's built-in SLIRP networking needs no special privilege
whatsoever, so qemu runs entirely as your own plain user, nothing it creates
is ever root-owned, and no script in this toolkit ever invokes `sudo` for a
`user`-mode VM.

SLIRP has no notion of the host reaching in to the guest by IP, so
`user`-mode VMs get an SSH port forwarded from the host instead: guest port
22 is reachable at `localhost:<20000 + INSTANCE>` (e.g. `INSTANCE=02` →
`localhost:20002`). `vm-init` and `vm-start` both print the exact port on
every boot.

`NETMODE` no longer affects *where* a VM's files live (that's always just
wherever you run `vm-init` from, or whatever path you give `vm-start`) — only
the qemu invocation itself. Every socket/process interaction in `bin/vm-lib`
(talking to QMP, the guest agent, killing a process) decides for itself, per
call, whether `sudo` is actually needed by checking who owns the thing it's
about to touch (`root` → prefix `sudo`; anyone else, including yourself →
don't) — see `path_owner`/`pid_owner`/`run_as` in `bin/vm-lib`. This is what
lets a single set of scripts serve both modes without every caller needing to
know or pass the mode through explicitly.

Base OS images are cached, unprivileged, at `~/.cache/vm-cli/images/`,
shared across every VM regardless of `NETMODE` or where its `.vmsb` ends up,
so building several VMs on the same OS release only downloads it once.

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
| `NAME`     | Guest-internal identity only — used to build the guest's hostname/FQDN. **Not** the bundle filename or mount point (see "Concept" above) — those come from the config *file's* own name, e.g. `example.vmc` → `example.vmsb` → `/Volumes/example`, regardless of what `NAME=` says inside it. |
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
2. **Computes the bundle's path and name** — the current directory plus the
   config file's own basename (`.vmc` stripped, `.vmsb` appended), e.g.
   `example.vmc` → `./example.vmsb`. Refuses to clobber an existing VM —
   fails if that path, or `/Volumes/<name>`, already exists.
3. **Creates the sparsebundle**, sized to the configured `DISK` plus a 15GB
   overhead for firmware/bios/flatten-headroom, formatted APFS, then
   **attaches** it at `/Volumes/<name>` for the rest of the build. (`hdiutil
   create` unconditionally appends its own `.sparsebundle` suffix regardless
   of what extension you ask for — see "Known gotchas" — so this step
   creates it, then renames the result to the exact `.vmsb` path.)
4. **Copies the config file** into the bundle as `config` (this is what
   `vm-start` reads later — the original config file is no longer needed
   once this step is done).
5. **Downloads and verifies the base Ubuntu cloud image** (arm64 server
   image for the configured `OS` release) into `~/.cache/vm-cli/images/` if
   it isn't already there, checking its SHA256 against Ubuntu's published
   checksums, then **copies the verified image into the bundle** so the
   bundle is self-contained during the bake (removed again once baking
   finishes — see step 8).
6. **Creates the virtual hard disk** — a `qcow2` copy-on-write overlay
   backed by the in-bundle base image copy — and resizes it to the
   configured `DISK` size.
7. **Copies UEFI firmware and vars**, and **renders the cloud-init seed**
   (`meta-data`, `network-config`, `user-data` from
   [`templates/`](templates), picking the `bridged` or `user` variant of
   `network-config`/`user-data` — see "Cloud-init profiles" below) into a
   `cidata.iso` inside the bundle.
8. **Bakes a golden disk image**:
   1. Boots the disk *once*, with the cloud-init seed attached, and lets
      cloud-init do its one-time provisioning (create the user, install
      packages, configure networking, ...) for real, directly onto the disk.
   2. Waits for the guest agent to respond to a ping (confirms `qemu-ga`
      itself is up — see "Diagnosing a hung boot" below).
   3. **Separately confirms cloud-init has actually finished** — not just
      that the agent answered — via `cloud-init status --wait`, run inside
      the guest through the guest agent's `guest-exec` (see "Diagnosing a
      hung boot" for why this distinction matters).
   4. Requests a graceful shutdown (QMP `system_powerdown` — the software
      equivalent of pressing the physical power button; the guest's own OS
      decides how and when to actually respond) and waits for qemu to exit.
   5. Flattens the disk with `qemu-img convert` so it no longer depends on
      the base cloud image as a backing file, then deletes the base image
      copy and `cidata.iso` from the bundle — neither is needed again, since
      every future boot (`vm-start`) just boots the already-provisioned
      golden disk directly, with **no cloud-init involved at all**.
9. **Detaches the bundle**, leaving a ready-to-run `.vmsb` and printing its
   path plus a reminder of the name to use with `vm-start`/`vm-status`/etc.

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
  .run/               # created on demand by vm-start - see below
```

(`images/` and `cidata.iso` only exist transiently, during the bake.)

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
bin/vm-start <path to .vmsb>
```

This is headless and returns control to your shell prompt once the VM is
up — it does not stay attached to the VM's console. Unlike the control
scripts below, this takes the actual **path** to the `.vmsb`, not just a
name — there's no fixed location to search for one that isn't running yet.
In order, it:

1. Refuses to start if the VM already appears to be running (a live
   pidfile inside the mount), but reclaims a stale mount at `/Volumes/<name>`
   left behind by an unclean previous exit (e.g. the guest was powered off
   directly instead of via `vm-halt`/`vm-stop`).
2. Attaches the given `.vmsb` at `/Volumes/<name>` (`<name>` being its own
   basename) and sources the VM's `config` from it, which is where
   `NETMODE` (and everything else) actually comes from at this point.
3. Creates a hidden `.run/` directory *inside* the mounted volume for this
   run's host-side state (pidfile, sockets, logs — see below), clearing out
   anything left behind by an earlier run.
4. **`NETMODE=bridged` only**: authenticates `sudo` once up front and
   keeps the ticket refreshed in the background for the rest of the
   script, so a slow boot doesn't hit repeated password prompts.
   **`NETMODE=user`**: none of this happens — nothing in this path ever
   calls `sudo`.
5. Launches `qemu-system-aarch64` — under `sudo` for `bridged`, as the
   plain user for `user` — backgrounded via `nohup ... & disown` (see
   "Known gotchas" below for why not `-daemonize`). `-display none` /
   `-monitor none` keep it fully headless — there's no interactive
   console attached to this terminal at all (the serial console is still
   reachable over a socket and logged to a file — see "Diagnosing a hung
   boot" below).
6. Waits for the guest agent to respond (see "Diagnosing a hung boot"
   below for how), up to a 180s budget (override with `VM_START_TIMEOUT`),
   printing timestamped progress. Ctrl-C during this wait just stops
   waiting — the VM keeps running in the background either way.
7. Reports whether the guest responded in time and returns to the
   prompt. **The volume stays mounted** — qemu is still using files
   there. `bin/vm-halt` or `bin/vm-stop` detach it once the VM actually
   stops (see below).

qemu's own stderr from the startup phase is captured to `qemu.log` for
debugging if something goes wrong.

Runtime state for the running qemu process — the qmp/qga/serial control
sockets, a pidfile (written by the script itself, as the plain user — see
"Known gotchas"), and logs — is host-side state, not part of the portable
VM bundle conceptually, but it lives *inside* the mounted volume anyway (in
a hidden `.run/` directory), so it always travels with the VM for as long
as it's running with no separate location to keep track of:

```
/Volumes/<name>/.run/
  pid
  qmp.sock
  qemu-ga.sock
  serial.sock       # guest serial console socket, for interactive debugging
  serial.log        # persistent log of everything written to the serial console
  qemu.log          # qemu's own stdout/stderr
```

## Controlling a running VM

Once a VM is running (`bin/vm-start <path>` in its own terminal), these
scripts control it from any other terminal, addressing it **by name** (the
`.vmsb`'s own basename — the same name it's mounted at, `/Volumes/<name>`):

| Script                    | Effect                                                                 |
|---------------------------|-------------------------------------------------------------------------|
| `bin/vm-status <name>`    | Report running/not-running, and the guest's run state (running/paused) |
| `bin/vm-halt <name>`      | Ask the guest to shut down gracefully (ACPI power button)              |
| `bin/vm-stop <name>`      | Force-stop qemu immediately — no graceful guest shutdown               |
| `bin/vm-restart <name>`   | Hard-reset the guest (like pressing a physical reset button)           |
| `bin/vm-pause <name>`     | Suspend guest execution in place                                       |
| `bin/vm-resume <name>`    | Resume a paused guest                                                  |

Notes:

- **Why a name here, but a path for `vm-start`**: a VM's mount point
  (`/Volumes/<name>`) is fully predictable from its name alone once it's
  running, and everything these scripts need (pidfile, sockets) lives right
  there in `.run/` — there's nothing to search for. `vm-start` doesn't have
  that luxury: the VM isn't mounted yet, so the only way to find it is to be
  told exactly where its `.vmsb` is.
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
  and the name-to-path resolution described above; it's a library meant to
  be sourced, not run directly.
- In `NETMODE=user`, the guest's SSH port is forwarded to
  `localhost:<20000 + INSTANCE>` (SLIRP has no other way to reach in) —
  `vm-start` prints the exact port every time it boots the VM.

## End-to-end smoke test (`vm-fulltest`)

```
bin/vm-fulltest [config file]
```

Defaults to `example.vmc`; pass `example-nosudo.vmc` to exercise the
`user`-mode path end to end with zero `sudo` prompts. Run from wherever you
want the throwaway `.vmsb` to land (same as a plain `vm-init` call). It
tears down any existing VM of that name first (stop, detach, delete the
bundle), builds a fresh one with `vm-init`, starts it, and exercises every
control script above against it, asserting the expected state at each
step — including the golden-image properties (flattened disk, no
`cidata.iso`, no leftover base-image copy) and, across a `vm-restart`, that
the qemu process itself never changed pid (only the guest rebooted). Prints
a PASS/FAIL summary and exits 0 only if everything passed.

## Diagnosing a hung boot

This project went through two real, reproducible hangs early on, each with
a different root cause.

**Hang #1: the guest agent never responds.** `qemu-guest-agent` would
sometimes never answer a `guest-ping`, even after waiting 900+ seconds,
with `qemu.log` completely silent and no obvious cause. The actual root
cause (found from a leftover `serial.log` of one such hang) turned out to
have nothing to do with the guest's boot process itself — the guest had
fully booted, cloud-init had finished, and `qemu-guest-agent.service` had
reported `Started`, all within about 30 seconds; the console then went
completely silent for the rest of the wait. The real problem was on the
*host* side: the old polling loop reconnected a brand-new `nc` client to
the guest agent's virtio-serial socket on every single attempt, and that
repeated connect/disconnect churn — especially happening before the guest
had booted far enough for `qemu-ga` to actually open its end of the port —
could leave the port permanently desynced for the rest of that boot, with
the guest-side daemon never becoming reachable again no matter how long
(or how many more times) you kept polling. Widening the timeout (180s →
400s → 900s) never fixed it, because it was never a "needs more time"
problem.

The fix, `wait_for_qga_ready` in `bin/vm-lib`, holds a **single, long-lived
connection** to the guest agent socket for the entire wait instead of
reconnecting per attempt, so the guest only ever sees one connect event.
It's used everywhere this toolkit polls for the guest agent to come up.

**Suspect #2: "guest agent responded" isn't the same as "cloud-init is
actually done".** After fixing hang #1, `vm-init`'s bake boot still hit an
intermittent failure: the graceful shutdown it requests once the guest
agent responds sometimes doesn't complete within its timeout. The guest
agent answering a ping only proves `qemu-ga` itself is up and listening —
it says nothing about whether cloud-init has genuinely finished its own
final-stage work. Requesting a shutdown while cloud-init might still be
doing something is a real, separate possible cause of a slow or stuck
poweroff, distinct from hang #1 above (`wait_for_pid_exit`, what the
shutdown wait actually uses, never touches the QMP or QGA sockets at all,
so the persistent-connection fix doesn't apply to it).

To close this gap, `wait_for_cloud_init_done` in `bin/vm-lib` runs
`cloud-init status --wait` *inside the guest*, via the guest agent's
`guest-exec` feature, and waits for cloud-init itself to report a real
exit code (0 = done, 2 = degraded/recoverable warnings only, anything else
= a real error) before `vm-init` ever requests a shutdown. This is expected
to resolve quickly in practice — cloud-init's `runcmd` module, which is
what starts the guest agent, runs late in cloud-init's own sequence, so by
the time the agent answers there should be little left for cloud-init to
do — which is exactly why its timeout (`VM_BAKE_CLOUDINIT_TIMEOUT`,
default 120s) is much smaller than the guest-agent wait itself. Whether
this fully explains the intermittent bake-shutdown slowness is **not yet
confirmed** — see "Known open issue" below.

**If you ever do hit a genuinely stuck boot**, `.run/serial.log` (inside
the mounted volume) is where to look first: it's a **persistent, plain-file**
capture of the guest's actual console output (kernel messages, systemd unit
starts/stops, cloud-init's own output, login prompts), pre-created by the
plain user and `chmod 644`'d *before* qemu ever opens it — specifically so
it stays readable without `sudo` even for a `bridged` VM, and survives
after qemu itself is killed. Compare it against `.run/qemu.log` (qemu's own
stdout/stderr — silence there just means qemu itself didn't crash, it says
nothing about the guest). For a live guest, you can also connect directly
to the socket right next to it — `nc -U /Volumes/<name>/.run/serial.sock`
(add `sudo` for a `bridged` VM) — to get an interactive console, including
a login prompt if the guest reached one.

**Reading wait progress**: every wait in this toolkit now prints a
timestamped start line (what it's waiting for, the wall-clock time, and the
configured timeout) and a periodic `NNs / NNs elapsed (HH:MM:SS)` progress
line roughly every 15 real seconds — computed from actual wall-clock time,
not counted in fixed per-iteration increments. This replaced a plain
dot-per-poll counter that was genuinely easy to misread — e.g. a
2-seconds-per-iteration loop's 60 dots is 120 elapsed seconds, not 60, and
there was nothing printed to catch that at a glance.

## Timeout reference

Every timeout below is real wall-clock seconds, overridable via the listed
environment variable, and printed explicitly (start time + budget) by
whichever wait function is watching it — see "Reading wait progress" above.

| Variable                       | Default | What it waits for                                                        |
|---------------------------------|---------|----------------------------------------------------------------------------|
| `VM_START_TIMEOUT`               | 180s    | `vm-start`: guest agent responds after booting the golden disk (no cloud-init involved, so this should normally resolve fast) |
| `VM_HALT_TIMEOUT`                | 120s    | `vm-halt`: qemu exits after a graceful ACPI shutdown request               |
| `VM_STOP_TIMEOUT`                 | 30s     | `vm-stop`: qemu exits after a hard QMP `quit` (should be near-instant)     |
| `VM_BAKE_TIMEOUT`                 | 900s    | `vm-init` bake: guest agent responds during first boot (covers cloud-init's package_update/upgrade on `bridged`, which can genuinely take minutes) |
| `VM_BAKE_CLOUDINIT_TIMEOUT`       | 120s    | `vm-init` bake: `cloud-init status --wait` reports done, *after* the guest agent already answered - see "Diagnosing a hung boot" for why this is a separate, smaller budget |
| `VM_BAKE_SHUTDOWN_TIMEOUT`        | 120s    | `vm-init` bake: qemu exits after the bake's own graceful shutdown request - matches `VM_HALT_TIMEOUT`'s default deliberately, since it's the same underlying operation (ACPI shutdown wait) just on a freshly-provisioned guest instead of an already-settled one |
| `VM_FULLTEST_BOOT_TIMEOUT`        | 240s    | `vm-fulltest`: overrides `VM_START_TIMEOUT` for its own boot assertions (both boots) - deliberately larger than the plain default for test-environment margin |
| `VM_FULLTEST_RESTART_TIMEOUT`     | 240s    | `vm-fulltest`: guest agent responds again after `vm-restart`               |
| `VM_FULLTEST_HALT_TIMEOUT`        | 120s    | `vm-fulltest`: overrides `VM_HALT_TIMEOUT` - matches `vm-halt`'s own default exactly |
| `VM_FULLTEST_STOP_TIMEOUT`        | 30s     | `vm-fulltest`: overrides `VM_STOP_TIMEOUT` - matches `vm-stop`'s own default exactly |

A single `vm-init` bake, worst case (every phase timing out), is bounded by
`VM_BAKE_TIMEOUT + VM_BAKE_CLOUDINIT_TIMEOUT + VM_BAKE_SHUTDOWN_TIMEOUT`
(≈ 19 minutes at the defaults above) before it gives up and fails the build.

One deliberate *inconsistency*, not a bug: `vm-halt`'s own timeout leaves the
volume mounted if it's exceeded (the VM might still be alive and legitimate -
you may just want to wait longer or investigate), while `vm-init`'s bake
timeouts force-kill the bake-boot qemu process instead. This difference is
intentional — a bake is a one-shot internal build step, not something meant
to be left running, so leaving a stray build-time qemu process behind on
failure would be the wrong default there.

## Known gotchas

- **`hdiutil create -type SPARSEBUNDLE` ignores your extension.** Asking it
  to create `example.vmsb` actually produces `example.vmsb.sparsebundle` —
  it unconditionally appends its own `.sparsebundle` suffix to any path
  that doesn't already end in exactly that, with no flag to suppress it.
  `vm-init` works around this by creating it normally, then renaming the
  result to the exact `.vmsb` path (a sparsebundle is just a package
  directory; renaming it doesn't affect its validity as a disk image).
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

## Known open issue: intermittent bake-shutdown timeout

Two consecutive automated `bin/vm-fulltest example-nosudo.vmc` runs (back-
to-back: tear down, rebuild, tear down, rebuild) both hit `vm-init`'s bake
boot failing to power off within `VM_BAKE_SHUTDOWN_TIMEOUT` (120s), even
though a *later* `vm-halt` in the same test run — same guest OS, but
already-settled rather than freshly-provisioned — powered off fine within
120s both times. A manual (non-automated, non-back-to-back) run by this
project's owner completed cleanly instead, so this looks more like a timing-
or load-sensitive race than a deterministic bug.

Since then, `wait_for_cloud_init_done` (see "Diagnosing a hung boot" above)
was added specifically to test the hypothesis that the shutdown request was
racing against cloud-init's own final-stage work still being genuinely in
progress. **This has not yet been re-tested against a real reproduction** —
if you hit the bake-shutdown timeout again, check whether it's now
consistently getting past the new "cloud-init reported done after Ns"
message before requesting shutdown (in which case this hypothesis is
disproven and something else is the cause), and check `.run/serial.log`
for what the guest's console was actually doing in the gap between
"Requesting shutdown so the disk can be finalized..." and either
completion or the `Guest didn't power off within Ns` failure. That log
gets overwritten by the *next* boot, so to actually catch it, run
`bin/vm-init` on its own (not via `vm-fulltest`, which immediately runs
`vm-start` afterward and clears it) and inspect the log right after a
failure, before doing anything else with that VM.

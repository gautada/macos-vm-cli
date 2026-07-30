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
- `sudo` access — used once to create/own `/Users/Shared/qemu`, and again to
  run `qemu-system-aarch64` itself (it needs raw access to `vmnet-bridged`
  networking)

## Concept

Each VM is built as a single, self-contained, mountable **sparsebundle**
disk image — a macOS disk image made of many small band files, functionally
just a folder on disk, but one that can be attached like a real volume. It
lives at:

```
/Users/Shared/qemu/vms/<name>.sparsebundle
```

Everything the VM needs to run is stored *inside* that bundle: the base OS
image, the virtual hard disk, UEFI firmware/vars, and the cloud-init seed
ISO. Nothing the VM needs lives outside the bundle, so it can be copied,
backed up, or moved to another Mac and it will still work.

The bundle is only mounted while something is actively using it:
- `vm-init` mounts it at `/Volumes/<name>` while it builds the VM's files,
  then detaches it when done.
- `vm-start` mounts it at `/Volumes/<name>` and leaves it mounted for as
  long as the VM is running — qemu itself runs headlessly in the
  background, so the volume stays attached after `vm-start` returns.
  `vm-halt` or `vm-stop` detaches it again once the VM actually stops.

At rest (VM not built, or not running), the bundle is just an inert file
on disk — nothing is mounted, and `/Volumes/<name>` doesn't exist.

## Configuration file

A VM is described by a small config file (see [`example.vmc`](example.vmc)):

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
```

| Field      | Meaning                                                              |
|------------|-----------------------------------------------------------------------|
| `NAME`     | VM name — also the bundle filename, mount point, and hostname prefix |
| `DOMAIN`   | Domain suffix used to build the guest's FQDN                          |
| `INSTANCE` | Instance number, appended to `NAME` for the hostname/FQDN             |
| `MEMORY`   | RAM given to the guest (qemu `-m` value, e.g. `8G`)                   |
| `DISK`     | Guest disk size (grown into via `qemu-img resize`, e.g. `80G`)        |
| `CPUS`     | Guest vCPU count (qemu `-smp` value)                                  |
| `MAC`       | MAC address for the guest's virtio-net interface                     |
| `IP`       | Static IP/CIDR for the guest (netplan `addresses` entry)              |
| `GATEWAY`  | Default gateway / nameserver for the guest                            |
| `OS`       | Ubuntu release to base the VM on (e.g. `24.04`), matches a cloud image |

Every field is required — `vm-init` fails fast if any is missing or empty.

## Initializing a VM (`vm-init`)

```
bin/vm-init <config file>
```

This is a one-time build step per VM. In order, it:

1. **Validates the config file** and loads each field into a `VM_*`
   environment variable (e.g. `VM_NAME`, `VM_DISK`).
2. **Prepares the shared QEMU directory** at `/Users/Shared/qemu`, creating
   (and taking ownership of) the `vms/` bundle folder and a `cache/images/`
   folder used to cache downloaded base OS images across VMs.
3. **Refuses to clobber an existing VM** — fails if
   `vms/<name>.sparsebundle` or `/Volumes/<name>` already exists.
4. **Creates the sparsebundle**, sized to the configured `DISK` plus a 10GB
   overhead for firmware/bios/cidata, formatted APFS, then **attaches** it
   at `/Volumes/<name>` for the rest of the build.
5. **Copies the config file** into the bundle as `config` (this is what
   `vm-start` reads later — the original config file is no longer needed
   once this step is done).
6. **Downloads and verifies the base Ubuntu cloud image** (arm64 server
   image for the configured `OS` release) into the shared cache if it
   isn't already there, checking its SHA256 against Ubuntu's published
   checksums, then **copies the verified image into the bundle** so the
   bundle is self-contained.
7. **Creates the virtual hard disk** — a `qcow2` copy-on-write overlay
   backed by the in-bundle base image copy — and resizes it to the
   configured `DISK` size.
8. **Copies UEFI firmware and vars** (`edk2-aarch64-code.fd` /
   `edk2-arm-vars.fd` from the Homebrew qemu package) into the bundle. The
   vars file is per-VM because UEFI writes boot-variable state into it at
   runtime.
9. **Renders the cloud-init seed** (`meta-data`, `network-config`,
   `user-data` from [`templates/`](templates)) with the VM's settings and
   your SSH public key, and packages them into a `cidata.iso` inside the
   bundle.
10. **Detaches the bundle**, leaving a ready-to-run
    `vms/<name>.sparsebundle` and printing the path plus a reminder to run
    `vm-start`.

If any step fails partway through, a trap detaches the volume so a failed
build doesn't leave a stray mount at `/Volumes/<name>`.

Once built, the bundle's volume contains:

```
/Volumes/<name>/
  config              # copy of the .vmc config file
  images/ubuntu-<os>.img   # base cloud image (read-only backing file)
  virtual-hard-disk   # qcow2 overlay disk, backed by images/ubuntu-<os>.img
  firmware            # UEFI code (read-only)
  bios                # UEFI vars (read/write, per-VM)
  cidata.iso          # cloud-init seed (NoCloud datasource)
```

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
2. Attaches `vms/<name>.sparsebundle` at `/Volumes/<name>` and sources
   the VM's `config` from it.
3. Authenticates `sudo` once up front and keeps the ticket refreshed in
   the background for the rest of the script, so a slow first boot
   (cloud-init installing/upgrading packages) doesn't hit repeated
   password prompts.
4. Launches `qemu-system-aarch64` with `-daemonize`, so qemu itself forks
   into the background once the machine has initialized and this script
   regains the terminal. `-display none` / `-monitor none` and dropping
   the old `mon:stdio` serial console keep it fully headless — there's no
   interactive console attached to this terminal at all.
5. Polls the guest agent (over `qemu-ga.sock`) with a `guest-ping` every
   few seconds, up to a 300s budget (override with `VM_START_TIMEOUT`),
   printing progress dots. Ctrl-C during this wait just stops waiting —
   the VM keeps running in the background either way.
6. Reports whether the guest responded in time and returns to the
   prompt. **The volume stays mounted** — qemu is still using files
   there. `bin/vm-halt` or `bin/vm-stop` detach it once the VM actually
   stops (see below).

qemu's own stderr from the startup phase (before it daemonizes) is
captured to `qemu.log` for debugging if something goes wrong.

Runtime state for the running qemu process — the qmp/qga/serial control
sockets, a pidfile (qemu's own real pid, via `-pidfile`), and the startup
log — is host-side state, not part of the portable VM bundle, so it's
kept separately at:

```
/Users/Shared/qemu/run/<name>/
  pid
  qmp.sock
  qemu-ga.sock
  serial.sock       # guest serial console, for debugging - e.g. `sudo nc -U serial.sock`
  qemu.log
```

## Controlling a running VM

Once a VM is running (`bin/vm-start <name>` in its own terminal), these
scripts control it from any other terminal by talking to its QMP socket
at `/Users/Shared/qemu/run/<name>/qmp.sock`:

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
- The QMP socket is owned by root (qemu itself runs under `sudo`), so
  every one of these scripts — including the read-only `vm-status` —
  needs `sudo` to talk to it and may prompt for your password.
- All of them share `bin/vm-lib` for the common QMP plumbing; it's a
  library meant to be sourced, not run directly.

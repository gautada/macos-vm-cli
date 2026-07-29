# macos-vm-cli

A Mac assed virtual machine cli toolset.  The idea is to have a set of
utilities and commands that make virtual machine definiition and management
easier on macOS.

## Pre-Requisite

homebrew, qemu, ...

## Build VM

- Create a sparsebundle with everything needed define a vm


# macOS Virtual Machine CLI

This is a set of scripts that form a utility for managing and maitaining a virtual machine in macOS.

## Virtual Machine Management

Create a VM environment.  This script formulates the inital setup of a the VM environment. 

```
vm-init <vm template file example.vm>
```

example.vm

```
NAME=jupiter
DOMAIN=gautier.org
INSTANCE=03
MEMORY=8G
DISK=80G
CPUS=4
NETWORK=52:54:00:07:00:03
IP=10.0.7.0322
GATEWAY=10.0.4.1
OS=24.04
```

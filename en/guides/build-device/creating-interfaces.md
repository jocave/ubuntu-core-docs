---
title: Developing snapd Interfaces
table_of_contents: true
---

# Developing Interfaces

This guide introduces the terminology and technologies used in an essential component of snapd called _Interfaces_. Having read the guide you should be in a position to consider creating Interfaces of you own. An accompanying codelabs tutorial will be available to walk you through the exact steps needed to create an example Interface.

[Codelabs Link]

## The sandbox

When using a snap package to distribute software, applications in a production scenario are run in a sandbox. This state is referred to in snapcraft terminology as runing under _strict confinement_.

A sandbox is a logical space where device owners can let processes run with tighly controlled access to system resources like data and hardware. In fact the default condition is that the process can only access files delivered in their own snap, a set of base applications in the _core snap_ and some writable storage.

## Getting familiar with the Security Systems

The sandbox environment is maintained by a number of systems that are referred to as Security Systems in snapd. They can be grouped in to two sets:

Those responsible for the sandbox process confinement:

* AppArmor
* seccomp

Those that populate the sandbox with resources that the process can use:

* dbus
* kmod
* mount
* systemd
* udev

## Accessing more resources through Interfaces

Default sandbox is restrictive
Processes that are run in the sandbox can read other files from the same package
… and very little else

How do we securely expose more resources to the sandbox?


The means by which extra resources are revealed to the sandbox

The Interface defines how the resource is accessed by the process
Operation follows a provider/consumer model
The resource consumer side is called a plug
The resource provider is called a slot



## How are Interfaces defined?

Interfaces defined in the snapd source tree only There is no loading of Interfaces from snaps when the system is booted, no plugin architecture

Makes security an intrinsic part of the snappy system
Interfaces are subject to review both from a code quality and security perspective
https://github.com/snapcore/snapd/tree/master/interfaces/builtin
To create a new Interface you need some basic knowledge of Go programming language
Will need to submit any new interfaces to the snapd project on github via a pull request



## How do Interfaces modify Security Systems?

Security systems are modified by receiving Snippets from Interfaces
A snippet is an array of bytes
Most interfaces expect this data to be some text
Connections persist across reboots so any modified system state must be able to be restored


## What does an Interface defintion look like?

https://github.com/snapcore/snapd/blob/master/interfaces/core.go#L83

snapd declares a Go interface called Interface
In Go interface implies methods that should be implemented by all compatible types

go is generally considered to NOT be object-oriented

Not inheritance - An interface indicates that to be compatible you should implement all the specified methods

The Interface interface

```go
type Interface interface {
    Name() string

    SanitizePlug(plug *Plug) error

    SanitizeSlot(slot *Slot) error

    PermanentPlugSnippet(plug *Plug, securitySystem SecuritySystem) ([]byte, error)

    ConnectedPlugSnippet(plug *Plug, slot *Slot, securitySystem SecuritySystem) ([]byte, error)

    PermanentSlotSnippet(slot *Slot, securitySystem SecuritySystem) ([]byte, error)

    ConnectedSlotSnippet(plug *Plug, slot *Slot, securitySystem SecuritySystem) ([]byte, error)

    AutoConnect(plug *Plug, slot *Slot) bool
}
```


Check the plug/slot declaration
The Sanitize{Plug,Slot} methods receive the {Plug,Slot}Info structs
Allows the interface validate the declarations and veto the connection attempt
Snap, Name, Label, Attrs, Apps, Hooks

slots:
  my-serial-port:
    interface: serial-port
    path: /dev/ttyS0

Producing snippets (1)
Connected{Plug,Slot}Snippet
Most common
User control (user/assertion control)
Access resources only when a plug/slot connection is established
e.g. app should be able to access a hardware resource only after connection granted




Producing snippets (2)
Permanent{Plug,Slot}Snippet
Less common
Implies unconditional acceptance of security policy defined in the interface when installed
Access resources as soon as installation is complete
e.g. allow management service to run when no client is connected

Producing Snippets (3)
Keep in mind ordering
On install/connection event:
For each snap
Call snippet function for each Security System on each plug
Call snippet function for each Security System on each slot
Move to next snap

### AppArmor
http://apparmor.net

An example of a LSM kernel module for enforcing policies
Main enforcer of the sandbox
All apps declared in a snap receive a default policy file:
https://github.com/snapcore/snapd/.../interfaces/apparmor/template.go
New policy entries are appended to this base template
The profiles can be found at:
/var/lib/snapd/apparmor/profiles


### dbus
https://www.freedesktop.org/wiki/Software/dbus/

Framework for IPC communication
Default policy prevents connecting to either system or session bus
The backend allows interfaces to claim a name on the system bus (no session support)
On connection new policy added to /etc/dbus-1/system.d/

### kmod
Allows snaps to request extra modules are loaded
The snippet is a newline separated list of modules names (not file locations)
The backend runs modprobe <module-name>
Requested modules stored in files at:
	/etc/modules-load.d/

### mount
Allows creation fstab format files
Request a mount to take place within the snap namespace
Key example is the Content Interface - solely based on mount backend, creates bind mounts between snaps
fstab file created per snap in:
	/var/lib/snapd/mount/

### seccomp
https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt

Limit the system calls a userspace process can make
Kills any process that attempts an unauthorised call
Profile contains all system calls, those not commented are allowed
Profile maintained in per app file at:
/var/lib/snapd/mount/

### systemd
https://www.freedesktop.org/software/systemd/man/systemd.unit.html

Backend allows creation of systemd units
Unit currently limited in function to services (no timers, devices, … yet)
The snippet is a map containing just some of the settings used to configure a service
https://github.com/snapcore/snapd/.../interfaces/systemd/snippet.go#L34
Unit file created in:
	/etc/systemd/system


### udev
https://www.freedesktop.org/software/systemd/man/udev.html

Manage device related events from the kernel
Maintain database of device information
Allow userspace to act predictably
Rules files define actions
Backend allows per snap rules files to be created in:
	/etc/udev/rules.d/


## Simple Interfaces

https://github.com/snapcore/snapd/.../interfaces/builtin/common.go#L36

Reduce the boilerplate code when all you need is to return a snippet
Understand this calls all the methods of type Interface interface
Only basic Santize{Slot,Plug) tests applied
Only supports AppArmor, seccomp, kmod snippets

## Tips for submitting you Interface Pull Request

Comprehensive tests
Maximise coverage <interface>_tests.go implementation
See Konrads talk!
You’ve written your interface - what do you need to do for it to land?

Register Interface
https://github.com/snapcore/snapd/.../interfaces/builtin/all{,_test}.go

Add your Interface to the list of those snapd expects, same goes for the test suite


Installation policy
snap-declaration assertion defines policy
there is a base declaration within snapd
https://github.com/snapcore/snapd/.../interfaces/builtin/basedeclaration.go
Four typical types of base declaration:
manually connected implicit slots (eg, bluetooth-control)
auto-connected implicit slots (eg, network)
manually connected app-provided slots (eg, bluez)
auto-connected app-provided slots (eg, mir)
See snap-declaration assertion specification for possible rules


Submission via github
Provide some context in your comments
Let the Spread based test suite run and modify your branch as necessary to keep them green
Don’t squash/rebase commits on a branch once PR opened to keep conversation clear

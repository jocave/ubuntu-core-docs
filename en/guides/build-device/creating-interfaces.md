---
title: Creating Interfaces
table_of_contents: true
---

# Creating Interfaces

This guide introduces the terminology and technologies used in an essential component of snapd called _Interfaces_. Having read the guide you should be in a position to consider creating Interfaces of you own. An accompanying codelabs tutorial will be available to walk you through the exact steps needed to create an example Interface.

[Codelabs Link]

## The sandbox

When using a snap package to distribute software, applications in a production scenario are run in a sandbox. This state is referred to in snapcraft terminology as running under _strict confinement_.

A sandbox is a logical space where device owners can let processes run with tighly controlled access to system resources like data and hardware. In fact the default condition is that the process can only access files delivered in their own snap, a set of base applications in the _core snap_ and some writable storage.

## Getting familiar with the Security Systems

The sandbox environment is maintained by a number of systems that are referred to as _Security Systems_ in snapd. They can be grouped in to two sets:

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

As described on the main [Interfaces](interfaces.md) page, the mechanism by which a snap can get access to more resources is Interfaces. Let's begin looking at how an Interfaces is defined.

Interfaces are defined in the snapd source tree _only_. There is no means by which a Interfaces can be declared and distributed by a snap to a running system. This explicit requirement has been made to ensure security an intrinsic part of the snapd based systems. Interfaces are subject to review both from a code quality and security perspective by subject matter experts and the wider community.

The snapd source tree is hosted on [github](http://github.com/snapcore/snapd). You will find the current Interface implemenations under [interfaces/builtin](https://github.com/snapcore/snapd/tree/master/interfaces/builtin).

If you followed last two hyperlinks you will quickly identify that the majority of the components of snapd are implemented in the Go programming language. Hence to submit a new Interface you will need to write some Go code.

## What does an Interface defintion look like?

As mentioned previously various Security Systems are responsible for maintaining the sandboxed environment. The Interface modifies the configuration of these systems by passing _snippets_ to them. Snippets are implemented as a collection of untyped data, but their purpose is to carry some modification to the policy that the Security System is applying to apps. For most of the security systems this data is actually some text which will be processed and stored in a configuration file. Plug/Slot connections persist across reboots so any modified Security System state must be able to be restored, use of the configuration files enables this.

The Go language uses a type called ```interface``` to allow the definition of a set of methods. All user types that wish to be compatible with this interface should have these methods. Here is the interface named Interface that defines the method set that anyone writing a interface will need to implement:

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
_[Ref: interfaces/core.go](https://github.com/snapcore/snapd/blob/master/interfaces/core.go)_


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

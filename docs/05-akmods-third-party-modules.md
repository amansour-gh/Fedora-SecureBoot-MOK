# akmods & Third-Party Modules

## Introduction

The previous chapters explained Secure Boot, MOK, kernel module signing, and the practical MOK enrollment workflow.

This chapter focuses on the next part of the process:

**building, installing, signing, and maintaining third-party kernel modules with `akmods`.**

On Fedora, external kernel modules often need to be rebuilt when a new kernel is installed.

The important distinction is:

```text
Build
  ↓
Install
  ↓
Sign
  ↓
Load
```

Each stage can fail independently.

A successful build does not automatically mean that the module is trusted by the kernel.

Likewise, a correctly signed module cannot load if it was built for the wrong kernel or has another dependency problem.

This chapter explains how `akmods` fits into that workflow and how to diagnose failures systematically.

---

## What Is an External Kernel Module?

A kernel module is code that can be loaded into the running Linux kernel.

Fedora provides many modules as part of the kernel packages themselves.

An **external kernel module** is built separately from the Fedora kernel source tree and is installed separately from the standard kernel package.

Examples can include:

* Proprietary graphics drivers
* Additional hardware drivers
* Virtualization-related modules
* Filesystem modules
* Other third-party kernel extensions

The exact modules installed on a system depend on the hardware and software configuration.

---

## Why External Modules Need to Be Rebuilt

A kernel module is built against a particular kernel interface and build environment.

When Fedora installs a new kernel, an external module built for an older kernel may not be available for the new one.

Conceptually:

```text
Kernel 6.x
   ↓
External module built for Kernel 6.x

Kernel 6.y
   ↓
External module must be available for Kernel 6.y
```

The module may need to be rebuilt even when the source code has not changed.

This is one of the main reasons `akmods` exists.

---

## What Is `akmods`?

`akmods` is a Fedora-oriented system for automatically rebuilding third-party kernel modules when required.

It is designed around the situation where an external module must remain available across kernel updates.

The general workflow is:

```text
New kernel installed
        ↓
akmods detects the new kernel
        ↓
External module is built
        ↓
Module is installed
        ↓
Module signing workflow is applied
        ↓
Module can be loaded
```

The exact build and signing behavior depends on the installed module packaging and system configuration.

`akmods` should therefore be understood primarily as a **module build and installation framework**, not as a replacement for Secure Boot or MOK.

---

## `akmods` Does Not Replace MOK

These mechanisms solve different problems.

### `akmods`

Builds and installs external kernel modules.

### Module signing

Adds a cryptographic signature to a kernel module.

### MOK

Provides an additional trusted certificate through the Secure Boot boot chain.

Conceptually:

```text
                 ┌──────────────┐
                 │    akmods    │
                 │ build/install│
                 └──────┬───────┘
                        ↓
                 External module
                        ↓
                 Module signing
                        ↓
                 Trusted certificate
                        ↓
                 Kernel verification
```

The separation is important when troubleshooting.

---

## The Kernel Build Environment

Building an external module requires a kernel development environment appropriate for the target kernel.

The exact packages depend on the module and Fedora release, but a typical system needs:

* The target kernel
* Matching kernel development files
* A compiler and build tools
* The module source or source package
* Any module-specific dependencies

The important requirement is **matching the module build environment to the target kernel**.

Check the running kernel:

```bash
uname -r
```

List installed kernels:

```bash
rpm -q kernel
```

Check installed kernel development packages:

```bash
rpm -q kernel-devel
```

You may have more than one `kernel-devel` package installed because multiple kernels can remain installed at the same time.

---

## Matching `kernel-devel` to the Kernel

A common source of confusion is assuming that any installed `kernel-devel` package is sufficient.

For an external module, the relevant development environment should correspond to the kernel for which the module is being built.

Check the running kernel:

```bash
uname -r
```

Then check the matching development package:

```bash
rpm -q "kernel-devel-$(uname -r)"
```

If the matching package is installed, Fedora can provide the appropriate build environment for that kernel.

If it is missing, a module build may fail.

---

## Check `akmods` Installation

Verify that `akmods` is installed:

```bash
rpm -q akmods
```

Check the command:

```bash
command -v akmods
```

Display the installed version:

```bash
akmods --version
```

The exact version depends on the Fedora release and package updates.

---

## Checking `akmods` Status

One of the most useful commands when diagnosing external module problems is:

```bash
akmods --verbose
```

This provides information about the state of modules managed by `akmods`.

Use it when:

* A module is missing after a kernel update
* A driver does not load
* A new kernel was installed
* A build may have failed
* You suspect that an external module was not rebuilt

Do not interpret a successful `akmods` status as proof that Secure Boot will accept the module.

Build status and trust status are separate questions.

---

## Building Modules Manually

`akmods` normally handles builds automatically when the appropriate trigger occurs.

A manual build can be requested when troubleshooting or when a module needs to be rebuilt before the normal automatic process occurs.

A general command is:

```bash
sudo akmods
```

This asks `akmods` to process the available module packages for the installed kernels.

A specific kernel can also be targeted when required:

```bash
sudo akmods --kernels "$(uname -r)"
```

The exact options supported by `akmods` can vary by version.

When troubleshooting, consult:

```bash
akmods --help
```

before relying on options that may differ between Fedora releases.

---

## Build Output and Logs

When an `akmods` build fails, the terminal output may provide the immediate reason.

For deeper investigation, inspect the system journal:

```bash
journalctl -u akmods
```

You can also inspect the current boot's kernel messages:

```bash
journalctl -k -b
```

When looking specifically for module-related messages:

```bash
journalctl -k -b | grep -Ei 'module|akmods|signature|verification'
```

Build logs may also be available under the `akmods` log directories depending on the installed package configuration.

Check:

```bash
sudo find /var/cache/akmods -maxdepth 3 -type f
```

Do not assume that every Fedora release or module package stores logs in exactly the same location.

---

## Build Failure vs Module Loading Failure

These two problems should not be mixed together.

### Build failure

The module was not successfully compiled.

Typical causes include:

* Missing development packages
* Missing compiler or build tools
* Kernel API changes
* Incompatible module source
* Incorrect build configuration
* Module-specific dependency problems

### Loading failure

The module exists but the kernel cannot load it.

Possible causes include:

* Invalid signature
* Untrusted signing key
* Missing dependency
* Incompatible module
* Hardware or driver conflict
* Kernel configuration restrictions

A useful diagnostic sequence is:

```text
Did the module build?
        ↓
Was it installed?
        ↓
Is it signed?
        ↓
Is the signing key trusted?
        ↓
Can the kernel load it?
```

---

## Verify That the Module Exists

Before investigating signatures, verify that the module actually exists for the target kernel.

For a specific module:

```bash
modinfo <module>
```

For example:

```bash
modinfo nvidia
```

If the module is associated with a particular kernel, inspect its filename:

```bash
modinfo -n <module>
```

The result should point to a module under the appropriate kernel module tree.

A missing module is not a MOK problem.

---

## Verify the Module's Kernel

Check the module information:

```bash
modinfo <module> | grep -E '^filename:|^vermagic:'
```

The `vermagic` field contains information about the kernel environment for which the module was built.

Compare it with:

```bash
uname -r
```

A mismatch does not automatically mean that the module is unusable in every circumstance, but it is an important diagnostic signal.

Do not modify the module simply to hide a version mismatch.

The correct approach is normally to rebuild it for the appropriate kernel.

---

## Verify the Module Signature

Once the module exists, inspect its signature metadata:

```bash
modinfo -F signer <module>
```

You can also inspect:

```bash
modinfo -F sig_id <module>
```

and:

```bash
modinfo -F sig_key <module>
```

For example:

```bash
modinfo -F signer nvidia
```

If no signer information is returned, the module may be unsigned.

However, signature metadata alone does not prove that the kernel trusts the corresponding certificate.

The trust relationship must be checked separately.

---

## Verify the Enrolled MOK

If the module is signed, check whether the expected certificate is enrolled:

```bash
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der
```

You can also inspect the enrolled certificates:

```bash
sudo mokutil --list-enrolled
```

This creates a useful diagnostic relationship:

```text
Module
  ↓
Signer
  ↓
Corresponding certificate
  ↓
MOK enrollment
```

If one of these links is missing, Secure Boot can prevent the module from loading.

---

## Check Secure Boot

Always verify the actual Secure Boot state when investigating a Secure Boot-related problem:

```bash
mokutil --sb-state
```

Expected output when enabled:

```text
SecureBoot enabled
```

Do not infer Secure Boot state from the fact that the system booted successfully.

A system can boot Fedora with Secure Boot disabled.

---

## Kernel Updates and `akmods`

Kernel updates are one of the most important reasons to understand `akmods`.

A typical update can look like:

```text
Fedora update
      ↓
New kernel installed
      ↓
New kernel-devel installed
      ↓
akmods rebuilds external modules
      ↓
Module installed for new kernel
      ↓
Module signing applied
      ↓
New kernel boots
```

If the rebuild succeeds, the external module should be available for the new kernel.

If the rebuild fails, the new kernel may boot without the external module.

This can be especially noticeable with graphics drivers.

---

## Do Not Remove the Old Kernel Immediately

When troubleshooting an external module after a kernel update, the previous working kernel can be useful as a diagnostic reference.

For example:

```text
Old kernel
   ↓
Module works

New kernel
   ↓
Module fails
```

This strongly suggests that something changed in the new kernel or the module build process.

The old kernel should not be treated as a permanent substitute for fixing the new kernel, but keeping a known-good kernel available can make recovery easier.

Avoid deleting the last known-working kernel while diagnosing a driver or module problem.

---

## Rebuilding After a Kernel Update

If a new kernel has been installed and the external module is missing, first inspect the state:

```bash
uname -r
```

Then:

```bash
rpm -q kernel-devel
```

Then:

```bash
akmods --verbose
```

If the required build environment is available, request a rebuild:

```bash
sudo akmods --kernels "$(uname -r)"
```

After the build completes, verify the module:

```bash
modinfo <module>
```

Then inspect the signature:

```bash
modinfo -F signer <module>
```

Finally, check kernel messages if the module still does not load:

```bash
journalctl -k -b | grep -Ei 'module|signature|verification'
```

---

## Signing After Rebuilding

Rebuilding and signing are related but separate stages.

Conceptually:

```text
Source
  ↓
Build
  ↓
Kernel module
  ↓
Sign
  ↓
Signed kernel module
  ↓
Install
  ↓
Kernel loads module
```

Depending on the module package and `akmods` configuration, signing may be integrated into the build/install workflow.

Do not assume that every externally built module is automatically signed merely because it was built through `akmods`.

Verify the actual module:

```bash
modinfo -F signer <module>
```

---

## Third-Party Modules Are Not All Identical

The phrase "third-party module" covers many different packaging and build systems.

A module can be supplied through:

* A Fedora package
* A third-party repository
* An RPM package containing an `akmods` source package
* A locally built module
* A vendor-specific installation system

These approaches do not necessarily use the same signing or update workflow.

Therefore, do not assume that instructions for one driver automatically apply to every external module.

This guide focuses on Fedora systems using the `akmods` workflow.

---

## Package Ownership Matters

Before manually replacing or rebuilding a module, determine where it came from.

For a module file:

```bash
modinfo -n <module>
```

Then inspect the owning RPM:

```bash
rpm -qf "$(modinfo -n <module>)"
```

This can help determine whether the module came from a Fedora package, an `akmods` package, or another RPM-managed source.

Do not overwrite a package-managed module manually without understanding how the package is expected to maintain it.

---

## Avoid Mixing Installation Methods

A common source of problems is installing the same driver through multiple independent methods.

For example, a system may accidentally contain:

```text
Distribution package
       +
Third-party package
       +
Manual vendor installation
```

These methods can interfere with each other.

Possible consequences include:

* Conflicting module files
* Different signing keys
* Different update mechanisms
* Unexpected module versions
* Difficult rollback procedures

Choose one supported installation path for a particular driver whenever possible.

---

## When a Module Works With Secure Boot Disabled

This is an important diagnostic clue.

Suppose:

```text
Secure Boot disabled
        ↓
Module loads

Secure Boot enabled
        ↓
Module fails
```

This strongly suggests that Secure Boot enforcement may be involved, but it does not prove that the signing key is the only problem.

Possible causes include:

* Unsigned module
* Invalid signature
* Untrusted signing key
* Module dependency problems exposed by a different boot configuration

The correct response is to inspect the module and kernel logs rather than permanently disabling Secure Boot.

Useful checks:

```bash
mokutil --sb-state
modinfo -F signer <module>
modinfo -F sig_id <module>
journalctl -k -b | grep -Ei 'module|signature|verification|secure boot'
```

---

## Understanding the Complete Workflow

The complete `akmods` and Secure Boot relationship can be represented as:

```text
Kernel update
     ↓
Matching kernel-devel
     ↓
akmods build
     ↓
External module
     ↓
Module signing
     ↓
Signed module
     ↓
MOK certificate available
     ↓
Kernel verifies signature
     ↓
Module loads
```

Each arrow represents a separate condition.

A failure at one stage does not necessarily mean that another stage is broken.

---

## Systematic Diagnostic Workflow

When an external module does not work after a kernel update, use the following sequence.

### Step 1 — Identify the running kernel

```bash
uname -r
```

---

### Step 2 — Check the development environment

```bash
rpm -q kernel-devel
```

Confirm that the required development environment for the target kernel exists.

---

### Step 3 — Check `akmods`

```bash
akmods --verbose
```

Look for failed or missing builds.

---

### Step 4 — Check the module

```bash
modinfo <module>
```

If the module cannot be found, investigate the build and installation process first.

---

### Step 5 — Check the module path

```bash
modinfo -n <module>
```

This identifies the actual module file being inspected.

---

### Step 6 — Check the module version information

```bash
modinfo -F vermagic <module>
```

Compare the result with:

```bash
uname -r
```

---

### Step 7 — Check the signature

```bash
modinfo -F signer <module>
```

Then:

```bash
modinfo -F sig_id <module>
```

---

### Step 8 — Check MOK enrollment

```bash
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der
```

---

### Step 9 — Check Secure Boot

```bash
mokutil --sb-state
```

---

### Step 10 — Read kernel messages

```bash
journalctl -k -b | grep -Ei 'module|signature|verification|secure boot'
```

Only after these checks should you start changing the configuration.

---

## Common Failure Categories

### The module does not exist

Likely areas:

* `akmods` build failure
* Missing development files
* Package installation problem
* Incompatible module source

---

### The module exists but is unsigned

Likely areas:

* Signing configuration
* Module packaging
* Build workflow
* Wrong module being inspected

---

### The module is signed but the key is not enrolled

Likely area:

* MOK enrollment

Check:

```bash
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der
```

---

### The module is signed and the key is enrolled

Continue investigating:

* Module compatibility
* Kernel version
* Dependencies
* Kernel configuration
* Driver conflicts
* Kernel logs

Do not assume every module-loading failure is caused by Secure Boot.

---

## Useful Commands at a Glance

### Kernel

```bash
uname -r
rpm -q kernel
rpm -q kernel-devel
```

### `akmods`

```bash
rpm -q akmods
akmods --version
akmods --verbose
sudo akmods --kernels "$(uname -r)"
```

### Module

```bash
modinfo <module>
modinfo -n <module>
modinfo -F vermagic <module>
modinfo -F signer <module>
modinfo -F sig_id <module>
modinfo -F sig_key <module>
```

### Secure Boot and MOK

```bash
mokutil --sb-state
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der
sudo mokutil --list-enrolled
```

### Logs

```bash
journalctl -u akmods
journalctl -k -b
journalctl -k -b | grep -Ei 'module|akmods|signature|verification'
```

---

## Practical Checklist

When an external module fails after a kernel update:

```text
[ ] Identify the running kernel
[ ] Confirm matching kernel development files
[ ] Check akmods status
[ ] Confirm the module exists
[ ] Confirm the module path
[ ] Check module vermagic
[ ] Check module signature
[ ] Verify the expected MOK is enrolled
[ ] Confirm Secure Boot state
[ ] Inspect kernel messages
[ ] Rebuild the module if required
[ ] Recheck the module after rebuilding
```

This approach avoids changing multiple variables at the same time.

---

## Security Considerations

Do not solve an external-module problem by permanently disabling Secure Boot without understanding the underlying issue.

Disabling Secure Boot changes the security model of the system.

If a module fails to load, investigate:

```text
Build
  ↓
Install
  ↓
Sign
  ↓
Trust
  ↓
Load
```

before changing Secure Boot configuration.

Also avoid downloading random prebuilt kernel modules from untrusted sources.

Kernel modules run with kernel-level privileges.

The source, package origin, signing process, and update mechanism all matter.

---

## Summary

`akmods` helps Fedora maintain external kernel modules across kernel updates.

Its role is primarily to:

* Build external modules
* Rebuild them for new kernels
* Install the resulting modules
* Integrate with the module packaging workflow

Secure Boot adds another requirement:

```text
External module
      ↓
Correctly built
      ↓
Correctly signed
      ↓
Signing certificate trusted
      ↓
Kernel accepts module
```

The most important troubleshooting principle is to separate these stages.

If the module was never built, MOK is not the first problem to investigate.

If the module exists but is unsigned, investigate signing.

If the module is signed but the certificate is not trusted, investigate MOK enrollment.

If the module is built, signed, and trusted but still fails, investigate compatibility, dependencies, kernel configuration, and kernel logs.

The next chapter applies these concepts to one of the most common external kernel-module cases on Fedora: **NVIDIA graphics drivers and Secure Boot**.
---

[← Previous: Practical MOK](04-practical-mok.md) | [Next: NVIDIA + Secure Boot →](06-nvidia-secure-boot.md)

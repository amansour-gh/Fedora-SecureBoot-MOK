# NVIDIA + Secure Boot

## Introduction

NVIDIA graphics drivers require special attention on Fedora systems when Secure Boot is enabled.

Unlike the built-in kernel drivers shipped with Fedora, the NVIDIA kernel modules are external kernel modules. They must therefore be built for the running kernel and, when Secure Boot is enabled, must be signed with a key trusted by the firmware.

On Fedora systems using RPM Fusion, this process is normally handled through `akmods`, with the resulting kernel modules signed using the local `akmods` signing key.

This chapter focuses specifically on NVIDIA and Secure Boot.

For the general concepts behind external kernel modules, `akmods`, module signing, MOK enrollment, and Secure Boot, see:

* `01-secure-boot.md`
* `05-akmods-third-party-modules.md`

The goal of this chapter is not to provide a collection of unrelated NVIDIA fixes. Instead, it provides a predictable installation and diagnostic workflow.

---

## NVIDIA on Fedora

Fedora does not include NVIDIA's proprietary driver in the standard Fedora repositories.

A common approach on Fedora Workstation is to use the NVIDIA packages provided through RPM Fusion.

The RPM Fusion packages integrate the driver with Fedora's packaging and kernel-update workflow and provide `akmod-nvidia` for building the NVIDIA kernel modules.

The exact package versions change over time. Do not copy a driver version from an old tutorial.

Instead, allow DNF to select the version appropriate for the current Fedora release and enabled repositories.

---

## Why `akmod-nvidia` Matters

The important package in a Secure Boot installation is:

```text
akmod-nvidia
```

`akmods` uses the NVIDIA kernel module source provided by the package to build a kernel-specific module.

The resulting module is associated with a particular kernel version.

Conceptually:

```text
NVIDIA package
      |
      v
akmods
      |
      v
NVIDIA kernel module
      |
      v
module signing
      |
      v
MOK / firmware trust
      |
      v
kernel loads module
      |
      v
NVIDIA driver becomes available
```

This is why installing the userspace NVIDIA packages alone is not enough.

The kernel module must exist, match the running kernel, be trusted by Secure Boot, and successfully load.

---

## Secure Boot Changes the Requirements

With Secure Boot disabled, the kernel can load an unsigned external module.

With Secure Boot enabled, the kernel enforces module-signing policy.

Therefore, an NVIDIA installation can appear to be complete while the driver still fails to load.

For example:

```text
NVIDIA packages installed
        |
        v
akmod build succeeded
        |
        v
module exists
        |
        v
module is unsigned
        |
        v
Secure Boot rejects module
        |
        v
nvidia-smi fails
```

This is why checking only the installed packages is insufficient.

A complete verification must cover:

1. the package installation,
2. the kernel development environment,
3. the `akmods` build,
4. the generated module,
5. the module signature,
6. the enrolled MOK,
7. Secure Boot state,
8. and the loaded NVIDIA driver.

---

## Before Installing NVIDIA

Before installing the driver, identify the current system state.

### Check the Fedora release

```bash
cat /etc/fedora-release
```

### Check the running kernel

```bash
uname -r
```

### Check Secure Boot

```bash
mokutil --sb-state
```

A system with Secure Boot enabled should report:

```text
SecureBoot enabled
```

### Check whether NVIDIA hardware exists

```bash
lspci -nnk | grep -A3 -i 'VGA\|3D\|NVIDIA'
```

This helps distinguish an NVIDIA driver problem from a system that does not actually contain an NVIDIA GPU.

---

## Avoid Mixing NVIDIA Installation Methods

Do not mix multiple NVIDIA driver sources unless you have a specific reason and understand the package ownership involved.

Common sources include:

* RPM Fusion
* NVIDIA's CUDA repository
* NVIDIA's `.run` installer
* third-party repositories

Mixing them can result in conflicting packages.

For example, installing some components from RPM Fusion and other components from the CUDA repository can produce package conflicts or leave the system with an inconsistent driver stack.

A Fedora system should normally use one coherent packaging strategy.

For the workflow in this guide, RPM Fusion is the expected source.

---

## Enable RPM Fusion

If RPM Fusion is not already enabled, install the standard Fedora RPM Fusion release packages.

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Then refresh the package metadata:

```bash
sudo dnf makecache
```

Do not add unrelated repositories simply because a third-party NVIDIA tutorial tells you to.

---

## Install the NVIDIA Driver

For the standard RPM Fusion workflow:

```bash
sudo dnf install akmod-nvidia
```

Depending on the workload, additional NVIDIA userspace packages may be required.

For example:

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda
```

Do not install packages simply because they appear in an old tutorial.

First determine what your workload actually requires.

For example:

* normal desktop acceleration does not require CUDA packages;
* CUDA applications may require additional CUDA-related packages;
* gaming and desktop use may require different supporting packages.

The core kernel-module package for the RPM Fusion workflow is `akmod-nvidia`.

---

## Verify the Package Source

After installation, inspect the installed package:

```bash
rpm -q akmod-nvidia
```

Then inspect its repository information:

```bash
dnf info installed akmod-nvidia
```

The package should come from the expected RPM Fusion NVIDIA repository.

You can also inspect all installed NVIDIA packages:

```bash
rpm -qa | grep -i nvidia
```

Look for unexpected packages from another repository.

If you see packages that clearly belong to another NVIDIA installation method, stop before attempting additional fixes.

Do not solve a repository conflict by repeatedly installing more NVIDIA packages.

---

## Check the Kernel Development Environment

`akmods` needs the development environment corresponding to the kernel for which the module is being built.

Check:

```bash
uname -r
```

Then:

```bash
rpm -q kernel-devel-$(uname -r)
```

Also check:

```bash
rpm -q kernel-headers
```

If the matching `kernel-devel` package is missing:

```bash
sudo dnf install kernel-devel-$(uname -r)
```

On Fedora, kernel updates can temporarily create a situation where the newest kernel and its development packages are not yet synchronized.

If a matching `kernel-devel` package cannot be found, do not force an unrelated version.

Check the installed kernels and available packages first.

---

## Build and Verify the NVIDIA Module

After installing or updating `akmod-nvidia`, Fedora normally builds the NVIDIA kernel module automatically.

Check `akmods`:

```bash
rpm -q akmods
akmods --verbose
```

If a manual build is required for the running kernel:

```bash
sudo akmods --kernels "$(uname -r)"
```

For a rebuild when troubleshooting an existing build:

```bash
sudo akmods --rebuild --force --kernels "$(uname -r)"
```

Use `--force` only when a rebuild is actually needed.

Do not reboot immediately after installing or updating an `akmod` package. Give the build time to complete and check:

```bash
akmods --verbose
```

If the build fails, inspect the `akmods` cache and journal:

```bash
sudo find /var/cache/akmods -maxdepth 3 -iname "*nvidia*" -print
journalctl -b | grep -i akmods
```

A failed build is different from a module that was built but rejected by Secure Boot. A module that never built cannot be fixed by enrolling another MOK.

---

## Verify the NVIDIA Module

Once the build completes, check:

```bash
modinfo nvidia
```

If the module exists, `modinfo` should return information about it.

A useful check is:

```bash
modinfo -F filename nvidia
```

This displays the module path.

Also check the module version:

```bash
modinfo -F version nvidia
```

If `modinfo nvidia` reports:

```text
modinfo: ERROR: Module nvidia not found.
```

do not proceed to Secure Boot troubleshooting yet.

First determine why `akmods` did not produce the module.

---

Before troubleshooting Secure Boot, verify that the module belongs to the kernel environment you intend to boot.

Check the running kernel:

```bash
uname -r
```

Then inspect the NVIDIA module:

```bash
modinfo nvidia | grep -E "filename|vermagic|version"
```

The `vermagic` information should correspond to the target kernel. A module built for a different kernel should not be treated as proof that the current kernel is ready.

---

## Verify NVIDIA Signing and MOK

With Secure Boot enabled, the NVIDIA module must be signed with a key trusted by the firmware.

Check the module signature:

```bash
modinfo nvidia | grep -E "signer|sig_key|sig_id"
```

The module should expose signature-related information.

The standard `akmods` certificate can be checked with:

```bash
ls -l /etc/pki/akmods/certs/public_key.der
```

Check the enrolled Machine Owner Keys (MOKs):

```bash
mokutil --list-enrolled
```

You can inspect the certificate details with:

```bash
openssl x509 -inform DER -in /etc/pki/akmods/certs/public_key.der -noout -subject -fingerprint
```

If the `akmods` certificate is not enrolled, import it:

```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

The import operation schedules the key for enrollment. It does not enroll the key immediately.

Reboot and complete the MOK enrollment screen when prompted. After returning to Fedora, verify Secure Boot:

```bash
mokutil --sb-state
```

The expected result is:

```text
SecureBoot enabled
```

---

## Load the NVIDIA Module

After verifying the module and its signature, check whether it is loaded:

```bash
lsmod | grep nvidia
```

You can also inspect the kernel module directly:

```bash
modinfo nvidia
```

If necessary for diagnosis, attempt to load it:

```bash
sudo modprobe nvidia
```

If `modprobe` fails, inspect the kernel log immediately:

```bash
sudo journalctl -k -b | grep -i nvidia
```

Also check:

```bash
sudo dmesg | grep -i nvidia
```

Some systems restrict access to `dmesg`; in that case, prefer `journalctl -k`.

---

## Verify the NVIDIA Driver

The standard NVIDIA diagnostic command is:

```bash
nvidia-smi
```

A successful result normally shows:

* the NVIDIA GPU,
* the driver version,
* supported CUDA information,
* and currently running GPU processes.

If `nvidia-smi` works, the NVIDIA kernel driver is communicating with the GPU.

However, `nvidia-smi` should not be the only diagnostic command used during installation.

It confirms the end result, not necessarily the reason a failure occurred.

---

## Verify the Kernel Driver Binding

Inspect PCI driver binding:

```bash
lspci -nnk | grep -A3 -i nvidia
```

Look for:

```text
Kernel driver in use: nvidia
```

If the system instead reports another driver, determine which driver is actually bound to the device before making changes.

---

## Nouveau and NVIDIA

Fedora systems may also contain the open-source `nouveau` driver.

The presence of nouveau does not automatically mean that the NVIDIA installation is broken.

The important question is which driver is actually bound to the GPU.

Check:

```bash
lspci -nnk | grep -A3 -i nvidia
```

Do not manually blacklist nouveau merely because an old tutorial recommends it.

Modern RPM Fusion NVIDIA packaging handles the necessary integration differently depending on the driver and hardware.

If nouveau is actually interfering with a specific installation, diagnose that situation from the kernel and driver state before modifying `/etc/modprobe.d/`.

---

## NVIDIA Open Kernel Modules

Recent NVIDIA driver packaging includes support for NVIDIA's open kernel modules for supported GPUs.

RPM Fusion provides `akmod-nvidia-open` through its `nonfree-tainted` repository.

Do not automatically switch to `akmod-nvidia-open` simply because it exists.

First determine whether your GPU and driver branch support the open kernel module and whether it is appropriate for your use case.

Check the installed package:

```bash
rpm -q akmod-nvidia
```

or:

```bash
rpm -q akmod-nvidia-open
```

Do not install both as competing solutions without understanding the package relationships.

If changing between them, use DNF's package transaction mechanisms rather than manually deleting individual NVIDIA files.

---

## NVIDIA and Kernel Updates

Kernel updates are one of the most common points where NVIDIA + Secure Boot problems appear.

When Fedora installs a new kernel, the normal workflow is:

```text
new kernel
    |
    v
matching kernel-devel
    |
    v
akmods builds NVIDIA module
    |
    v
module is signed
    |
    v
module installed for new kernel
```

The existing MOK key normally remains valid. You do not normally need to enroll a new MOK key after every kernel update.

The important requirement is that the NVIDIA module for the new kernel is successfully built and signed.

After a kernel or NVIDIA driver update, do not reboot immediately. Give `akmods` time to finish:

```bash
akmods --verbose
```

You can inspect the installed kernels and NVIDIA modules with:

```bash
rpm -q kernel
find /usr/lib/modules -type f -iname "*nvidia*.ko*" -print
```

A working NVIDIA module for an older kernel does not prove that the new kernel is ready.

---

## Rebuilding NVIDIA for a Specific Kernel

If a particular installed kernel is missing the NVIDIA module, rebuild it explicitly:

```bash
sudo akmods --kernels <kernel-version>
```

For the currently running kernel:

```bash
sudo akmods --kernels "$(uname -r)"
```

For a forced rebuild when troubleshooting an existing build:

```bash
sudo akmods --rebuild --force --kernels "$(uname -r)"
```

Replace `$(uname -r)` with an explicit kernel version when troubleshooting a kernel that is not currently running.

Do not regenerate the entire initramfs as a first response to every NVIDIA problem. If troubleshooting specifically indicates that the initramfs is stale or missing the required NVIDIA components, use the appropriate Fedora tooling, for example:

```bash
sudo dracut --force
```

An initramfs rebuild is not a substitute for fixing a failed `akmods` build or an unsigned module.

---

## Troubleshooting

### `akmods` Build Failure

If:

```bash
akmods --verbose
```

shows a failed NVIDIA build, first inspect the logs.

```bash
sudo find /var/cache/akmods -iname '*nvidia*' -print
```

Then inspect the relevant failure log.

Also check:

```bash
uname -r
rpm -q kernel-devel-$(uname -r)
```

Common causes include:

* missing matching `kernel-devel`,
* incompatible NVIDIA module source,
* a new kernel that is not yet supported by the installed driver version,
* mixed NVIDIA package sources,
* interrupted package transactions.

Do not treat every build failure as a Secure Boot problem.

---

### `modinfo nvidia` Cannot Find the Module

If:

```bash
modinfo nvidia
```

fails, determine whether the module was ever built.

Check:

```bash
akmods --verbose
```

Then:

```bash
find /usr/lib/modules -type f -iname '*nvidia*.ko*' -print
```

If no module exists for the target kernel, fix the build first.

---

### The Module Exists but Does Not Load

Check:

```bash
sudo modprobe nvidia
```

Then:

```bash
sudo journalctl -k -b | grep -i nvidia
```

Look for messages about:

* signature verification,
* module loading,
* unresolved symbols,
* version mismatch,
* device initialization,
* or conflicting drivers.

The kernel log is more useful than repeatedly reinstalling the driver.

---

### The Module Is Unsigned

Check:

```bash
modinfo nvidia | grep -E 'signer|sig_key|sig_id'
```

If the module is unsigned, determine why the `akmods` signing step did not occur.

Check:

```bash
sudo ls -l /etc/pki/akmods/certs/
```

Then review the `akmods` logs.

Do not disable Secure Boot merely to hide a signing problem.

---

### The Module Is Signed but the Key Is Not Enrolled

A signed module still cannot be trusted if the signing key is not enrolled.

Check:

```bash
mokutil --list-enrolled
```

Then compare the enrolled keys with the certificate used by `akmods`.

If necessary:

```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

Reboot and complete the MOK enrollment process.

---

### Secure Boot Rejects the Module

Check:

```bash
mokutil --sb-state
```

Then:

```bash
sudo journalctl -k -b | grep -Ei 'nvidia|secure|lockdown|module'
```

A Secure Boot rejection normally leaves useful evidence in the kernel log.

The correct response is to identify the trust-chain failure:

```text
module
  -> signature
  -> signing certificate
  -> enrolled MOK
  -> Secure Boot
```

Do not immediately disable Secure Boot.

---

### `nvidia-smi` Cannot Communicate With the Driver

Start with:

```bash
nvidia-smi
```

Then:

```bash
lsmod | grep nvidia
```

Then:

```bash
modinfo nvidia
```

Then:

```bash
lspci -nnk | grep -A3 -i nvidia
```

Finally:

```bash
sudo journalctl -k -b | grep -i nvidia
```

These commands distinguish between:

* no module,
* module present but not loaded,
* module rejected by Secure Boot,
* incorrect driver binding,
* or a deeper hardware/driver initialization problem.

---

## Avoid Blind Reinstallation

Repeatedly running:

```bash
sudo dnf reinstall '*nvidia*'
```

is not a diagnostic method.

Before reinstalling anything, determine which layer is failing.

Use this sequence:

```text
Package
  |
  v
akmods build
  |
  v
module exists
  |
  v
module matches kernel
  |
  v
module signed
  |
  v
MOK enrolled
  |
  v
Secure Boot trusts key
  |
  v
module loads
  |
  v
GPU initializes
  |
  v
nvidia-smi works
```

This prevents a working part of the installation from being destroyed while trying to fix an unrelated part.

---

## Avoid Mixing RPM Fusion and NVIDIA CUDA Packages

A particularly important failure category is package-source mixing.

Check installed repositories:

```bash
dnf repolist
```

Inspect installed NVIDIA packages:

```bash
rpm -qa | grep -i nvidia
```

Inspect the source of a specific package:

```bash
dnf info installed akmod-nvidia
```

If NVIDIA packages come from multiple unrelated repositories, stop and resolve the package ownership issue before continuing.

A clean package source is easier to maintain and troubleshoot.

---

## Diagnostic Workflow

When NVIDIA + Secure Boot fails, use this order.

### Step 1 — Identify the GPU

```bash
lspci -nnk | grep -A3 -i 'VGA\|3D\|NVIDIA'
```

### Step 2 — Identify the kernel

```bash
uname -r
```

### Step 3 — Verify `kernel-devel`

```bash
rpm -q kernel-devel-$(uname -r)
```

### Step 4 — Verify NVIDIA packages

```bash
rpm -qa | grep -i nvidia
```

### Step 5 — Check `akmods`

```bash
akmods --verbose
```

### Step 6 — Check the NVIDIA module

```bash
modinfo nvidia
```

### Step 7 — Check the module signature

```bash
modinfo nvidia | grep -E 'signer|sig_key|sig_id'
```

### Step 8 — Check MOK enrollment

```bash
mokutil --list-enrolled
```

### Step 9 — Check Secure Boot

```bash
mokutil --sb-state
```

### Step 10 — Check module loading

```bash
lsmod | grep nvidia
```

### Step 11 — Check driver binding

```bash
lspci -nnk | grep -A3 -i nvidia
```

### Step 12 — Check the kernel log

```bash
sudo journalctl -k -b | grep -Ei 'nvidia|secure|lockdown|module'
```

### Step 13 — Check NVIDIA userspace communication

```bash
nvidia-smi
```

This order is deliberate.

There is little value in debugging `nvidia-smi` when the kernel module does not exist.

---

## Common Failure Categories

Use this table as a quick reference before starting deeper troubleshooting.

| Symptom | First check | Likely layer |
|---|---|---|
| `rpm -q akmod-nvidia` reports the package is missing | `rpm -q akmod-nvidia` | Package installation |
| `kernel-devel` is missing | `rpm -q kernel-devel-$(uname -r)` | Kernel development environment |
| `modinfo nvidia` cannot find the module | `akmods --verbose` | Module build |
| The module exists but does not load | `sudo journalctl -k -b | grep -i nvidia` | Kernel module loading |
| The module has no signature information | `modinfo nvidia | grep -E "signer|sig_key|sig_id"` | Module signing |
| The module is signed but Secure Boot rejects it | `mokutil --list-enrolled` | MOK trust |
| Another driver is bound to the GPU | `lspci -nnk | grep -A3 -i nvidia` | Driver binding |
| `nvidia-smi` cannot communicate with the driver | `lsmod | grep nvidia` | Driver/runtime |
| The previous kernel works but the new kernel does not | `akmods --verbose` | Kernel-specific module build |
| NVIDIA packages come from different sources | `rpm -qa | grep -i nvidia` | Package source consistency |

---

## Security Considerations

Secure Boot is part of the trust chain between firmware and the operating system.

Disabling Secure Boot can make an unsigned NVIDIA module load, but it also changes the security model of the system.

Therefore, disabling Secure Boot should not be treated as the default NVIDIA troubleshooting step.

A better approach is:

1. identify the module,
2. verify its signature,
3. identify the signing key,
4. verify MOK enrollment,
5. verify Secure Boot state,
6. inspect kernel logs.

Only after understanding the failure should the system's Secure Boot configuration be intentionally changed.

---

## Practical Checklist

Before rebooting after an NVIDIA installation or update:

```bash
uname -r
```

```bash
rpm -q kernel-devel-$(uname -r)
```

```bash
rpm -q akmod-nvidia
```

```bash
akmods --verbose
```

```bash
modinfo nvidia
```

```bash
modinfo nvidia | grep -E 'signer|sig_key|sig_id'
```

```bash
mokutil --list-enrolled
```

```bash
mokutil --sb-state
```

If everything is ready, reboot.

After reboot:

```bash
lsmod | grep nvidia
```

```bash
lspci -nnk | grep -A3 -i nvidia
```

```bash
nvidia-smi
```

If the final command works and the expected NVIDIA driver is bound to the GPU, the basic NVIDIA + Secure Boot workflow is functioning.

---

## Useful Commands at a Glance

### NVIDIA package

```bash
rpm -q akmod-nvidia
```

### NVIDIA packages

```bash
rpm -qa | grep -i nvidia
```

### Running kernel

```bash
uname -r
```

### Matching development package

```bash
rpm -q kernel-devel-$(uname -r)
```

### `akmods`

```bash
akmods --verbose
```

### Build for the current kernel

```bash
sudo akmods --kernels "$(uname -r)"
```

### Force a rebuild

```bash
sudo akmods --rebuild --force --kernels "$(uname -r)"
```

### NVIDIA module

```bash
modinfo nvidia
```

### Module signature

```bash
modinfo nvidia | grep -E 'signer|sig_key|sig_id'
```

### MOK enrollment

```bash
mokutil --list-enrolled
```

### Secure Boot state

```bash
mokutil --sb-state
```

### NVIDIA module state

```bash
lsmod | grep nvidia
```

### GPU driver binding

```bash
lspci -nnk | grep -A3 -i nvidia
```

### NVIDIA status

```bash
nvidia-smi
```

### Kernel messages

```bash
sudo journalctl -k -b | grep -Ei 'nvidia|secure|lockdown|module'
```

---

## Summary

An NVIDIA installation with Secure Boot should be understood as a chain rather than a single package installation.

The complete chain is:

```text
NVIDIA package
      |
      v
akmods builds module
      |
      v
module matches kernel
      |
      v
module is signed
      |
      v
signing key is enrolled as MOK
      |
      v
Secure Boot trusts the key
      |
      v
kernel loads NVIDIA module
      |
      v
GPU driver initializes
      |
      v
nvidia-smi works
```

When something fails, find the first broken link in that chain.

This approach is more reliable than repeatedly reinstalling the driver or disabling Secure Boot.

For the underlying concepts of external modules, `akmods`, signing, MOK, and Secure Boot, refer back to:

* `01-secure-boot.md`
* `05-akmods-third-party-modules.md`

---

[← Previous: akmods & Third-Party Modules](05-akmods-third-party-modules.md)

# Kernel Module Signing & `akmods`

## Introduction

A Linux kernel module is code that can be loaded into the running kernel to provide additional functionality.

Examples include:

* Hardware drivers
* Filesystem modules
* Network modules
* Virtualization modules
* Third-party drivers

When Secure Boot and kernel module signature enforcement are involved, the kernel may require a module to contain a valid cryptographic signature from a trusted key.

This creates an important distinction:

> **A module can exist on disk, and it can even contain a signature, without being trusted by the kernel.**

On Fedora, external kernel modules are commonly handled through the `akmods` framework. `akmods` primarily builds and installs external modules for installed kernels, while module signing provides the cryptographic trust mechanism required by the kernel.

---

## What Is a Kernel Module?

The Linux kernel can provide functionality either directly inside the kernel or through loadable modules.

A loadable kernel module is commonly represented by a file ending in:

```text
.ko
```

For example:

```text
example.ko
```

A module contains compiled kernel code that can be loaded when required.

Modules allow Linux to support additional hardware and functionality without requiring every component to be built directly into the kernel image.

---

## External Kernel Modules

Fedora provides a large collection of kernel modules as part of its kernel packages.

Some modules, however, are developed or distributed outside the standard Fedora kernel packages.

Examples include:

* Proprietary hardware drivers
* Third-party drivers
* Locally developed kernel modules
* Modules distributed through external repositories

These are commonly referred to as **external kernel modules**.

External modules normally need to be built against a specific kernel environment.

For example:

```text
Kernel
  │
  ├── kernel-devel
  ├── kernel headers
  └── external module source
```

The exact requirements depend on the module being built.

---

## Why Kernel Modules Are Signed

A kernel module runs with kernel-level privileges.

Loading malicious code into the kernel could therefore compromise the operating system.

Module signing provides a mechanism for the kernel to verify the authenticity and integrity of a module before loading it.

The basic relationship is:

```text
Private key
     │
     │ signs
     ▼
Kernel module
     │
     │ verified using
     ▼
Public key
```

The private key creates the signature.

The corresponding public key is used to verify it.

The Linux kernel documentation describes module signing as a mechanism that allows the kernel to reject unsigned modules or modules whose signatures cannot be validated with a trusted key.

---

## Signature vs Trust

A critical distinction is that **having a signature does not automatically make a module trusted**.

A module can be:

* Unsigned
* Signed with a trusted key
* Signed with an unknown key
* Signed with an invalid signature

Conceptually:

```text
Module
  │
  ├── unsigned
  │
  └── signed
       ├── trusted key
       ├── unknown key
       └── invalid signature
```

When signature enforcement applies, the kernel requires a signature that it can successfully verify using a trusted public key.

Therefore:

> **Signature presence and signature trust are separate questions.**

---

## How Module Signing Works

The signing process uses asymmetric cryptography.

A key pair contains:

* **Private key** — used to create signatures
* **Public key** — used to verify signatures

The private key must remain protected.

The public key can be distributed and enrolled into an appropriate trusted-key infrastructure.

The complete conceptual process is:

```text
Build module
    ↓
Finalize module
    ↓
Sign module
    ↓
Install module
    ↓
Kernel verifies signature
    ↓
Trusted key?
```

Signing should happen after the module has reached its final form.

---

## Protecting the Private Key

The private signing key is the security-sensitive part of the process.

Anyone who obtains it may be able to create modules whose signatures are recognized as coming from that key.

Never:

* Commit the private key to Git
* Upload it to a public repository
* Store it in a publicly accessible directory
* Share it unnecessarily

For example, files such as:

```text
mok.key
private.key
signing.key
```

should never be committed to a public repository when they contain real signing keys.

The Linux kernel documentation also warns that compromise of the private key can allow malicious modules to be signed.

---

## Where MOK Fits

MOK provides an owner-controlled mechanism for enrolling an additional public key in the Secure Boot ecosystem.

The important distinction is:

> **MOK is not the private signing key.**

The private key is used to sign the module.

The corresponding public key is enrolled as a MOK and can become part of the trusted key path used when the kernel verifies the module.

The exact path by which keys become available to the kernel depends on the platform, boot components, kernel configuration, and distribution implementation.

The important requirement is that the kernel must have access to a trusted public key corresponding to the key used to sign the module.

---

## Module Verification Happens in the Kernel

The final signature check is performed by the kernel.

A userspace program such as `modprobe` can request that a module be loaded, but the kernel makes the security decision.

```text
modprobe
   ↓
Kernel module loader
   ↓
Signature verification
   ↓
Load or reject
```

This means that simply providing a module to `modprobe` cannot bypass kernel signature verification.

The Linux kernel documentation specifies that module signature checking is performed inside the kernel.

---

## Signature Enforcement

Kernel module signature behavior depends on kernel configuration and boot parameters.

For example, `CONFIG_MODULE_SIG_FORCE` and `module.sig_enforce=1` can require valid module signatures.

A simplified model is:

```text
Signature enforcement
        │
        ├── Valid trusted signature → accepted
        ├── Unsigned module         → rejected
        ├── Unknown signing key     → rejected
        └── Invalid signature       → rejected
```

The exact behavior should always be determined from the running kernel and its configuration rather than assumed solely from the presence of Secure Boot.

---

## Invalid Signatures

An invalid signature is different from an unsigned module.

A signature can fail because:

* The module was modified after signing
* The module was corrupted
* The wrong signature was attached
* The build or packaging process changed the file after signing

The kernel cannot establish the integrity of a module whose signature does not match its contents.

The kernel documentation notes that signed modules should not be stripped after signing because the signature covers the module contents.

Therefore, signing should be one of the final stages of the module build process.

---

## Inspecting a Module

The `modinfo` utility can display metadata about a kernel module:

```bash
modinfo <module>
```

The output may include:

* Module filename
* Description
* License
* Version
* Dependencies
* Kernel information
* Signature information

To obtain only the module filename:

```bash
modinfo -F filename <module>
```

This is useful for determining whether a module exists and which file is being referenced.

---

## Checking the Module Signer

Use:

```bash
modinfo -F signer <module>
```

For example:

```bash
modinfo -F signer nvidia
```

This displays signer information stored in the module metadata.

A non-empty result indicates that signer information is present.

It does **not** by itself prove that the kernel trusts the signing key.

---

## Checking Signature Metadata

Additional signature-related fields can be inspected with:

```bash
modinfo -F sig_id <module>
modinfo -F sig_key <module>
```

For example:

```bash
modinfo -F sig_id nvidia
modinfo -F sig_key nvidia
```

These fields provide additional information about the module's signature.

They are diagnostic information, not a replacement for the kernel's actual trust decision.

---

## Three Questions During Troubleshooting

When inspecting a module, keep these questions separate:

1. **Does the module contain a signature?**
2. **Which key or signer is associated with that signature?**
3. **Does the kernel trust that key?**

This prevents a common troubleshooting mistake:

```text
Signed
  ≠
Trusted
```

---

## What Is `akmods`?

`akmods` is an automatic kernel module build and installation system used by Fedora and related distributions.

Its primary purpose is to build external kernel modules for installed kernels and make those modules available when new kernels are installed.

The Fedora `akmods` package includes tools such as:

```text
akmods
akmodsbuild
kmodgenca
```

and systemd units used by the build process.

---

## Why `akmods` Is Needed

External modules normally need to be built for a particular kernel environment.

For example:

```text
Kernel A
   ↓
Module built for Kernel A
```

After a kernel update:

```text
Kernel A
   ↓
Kernel B
```

the external module may need to be rebuilt for Kernel B.

`akmods` automates much of this process:

```text
New kernel
    ↓
akmods
    ↓
Build external module
    ↓
Install module for new kernel
```

This is particularly useful for third-party drivers that need to be rebuilt after kernel updates.

---

## `akmods` and Kernel Development Files

Building an external kernel module requires an appropriate development environment.

The Fedora `akmods` package depends on kernel development packages and build tools required by the build process.

The general relationship is:

```text
Kernel
   ↓
Matching development environment
   ↓
External module source
   ↓
Build
   ↓
Kernel module
```

The exact requirements depend on the module.

---

## `akmods` Does Not Replace Module Signing

`akmods` and module signing solve different problems.

**`akmods`:**

> Builds and installs external kernel modules.

**Module signing:**

> Provides cryptographic authentication for those modules.

Therefore, a successful `akmods` build does not by itself prove that the resulting module will be accepted by a kernel enforcing trusted signatures.

The module must also satisfy the kernel's signature requirements.

---

## `akmods` and Secure Boot

On a Secure Boot system, an external module may need to satisfy both requirements:

```text
Correctly built module
        +
Trusted module signature
```

A simplified workflow is:

```text
Kernel update
    ↓
akmods builds module
    ↓
Module signing
    ↓
Module installation
    ↓
Kernel verifies signature
```

The signing step should not be treated as an unconditional property of every `akmods` build.

The actual signing behavior depends on the system's Secure Boot configuration and the configured `akmods` signing workflow.

---

## Fedora's `akmods` Secure Boot Support

The Fedora `akmods` package includes:

```text
README.secureboot
kmodgenca
/etc/pki/akmods/
```

These are part of Fedora's infrastructure for handling module signing with `akmods`.

The exact key-generation and enrollment procedure will be covered in the practical chapters of this guide.

This separation is intentional:

* Chapter 2 explains **what MOK is**.
* Chapter 3 explains **how module signing and `akmods` work**.
* A later practical chapter will explain **how to configure the actual Fedora workflow**.

---

## Kernel Updates and External Modules

Kernel updates are one of the main reasons external modules need automated build support.

Consider:

```text
Kernel A
   │
   └── External module A
```

After an update:

```text
Kernel B
   │
   └── External module B
```

The module may need to be rebuilt because it can depend on:

* Kernel interfaces
* Symbol versions
* Kernel configuration
* Architecture
* Kernel release

Therefore, a module built for one kernel should not automatically be assumed to work with another kernel.

---

## Build Failure vs Signature Failure

These are different problems.

### Build failure

The external module was not successfully built.

```text
Source
  ↓
akmods
  ↓
Build failure
```

There may be no usable module for the current kernel.

### Signature failure

The module exists, but its signature is missing, invalid, or not trusted.

```text
Module exists
      ↓
Signature verification
      ↓
Rejected
```

The troubleshooting procedure should identify which category the problem belongs to before making changes.

---

## Systematic Diagnostic Workflow

When an external module fails to load, inspect the system in this order:

```text
Secure Boot state
       ↓
Running kernel
       ↓
Module existence
       ↓
Module metadata
       ↓
Signature information
       ↓
Trusted keys
       ↓
akmods status
       ↓
Kernel messages
```

This approach avoids changing Secure Boot or regenerating keys before the actual problem is understood.

---

## Step 1 — Check Secure Boot

Use:

```bash
mokutil --sb-state
```

Typical output is similar to:

```text
SecureBoot enabled
```

or:

```text
SecureBoot disabled
```

This establishes the security context for the rest of the investigation.

---

## Step 2 — Check the Running Kernel

Use:

```bash
uname -r
```

This identifies the kernel currently running.

The module being investigated should be appropriate for this kernel.

---

## Step 3 — Check Whether the Module Exists

Use:

```bash
modinfo <module>
```

If the module is available to the module system, `modinfo` can display its metadata.

You can also obtain its filename:

```bash
modinfo -F filename <module>
```

This helps distinguish a missing module from a module that exists but fails during loading.

---

## Step 4 — Inspect Signature Information

Use:

```bash
modinfo -F signer <module>
modinfo -F sig_id <module>
modinfo -F sig_key <module>
```

These commands provide evidence about the module's signature.

They do not independently establish that the kernel will accept the module.

---

## Step 5 — Check Enrolled MOKs

Use:

```bash
mokutil --list-enrolled
```

This displays certificates enrolled as MOKs.

When troubleshooting a signed external module, the relevant question is whether the key associated with the module signature corresponds to a trusted key available to the kernel.

---

## Step 6 — Check `akmods`

Use:

```bash
akmods --status
```

This provides information about the state of external kernel module builds.

If the expected module is missing for the current kernel, investigate the `akmods` build process before focusing on signature problems.

---

## Step 7 — Inspect `akmods` Logs

The `akmods` package provides logs under:

```text
/var/log/akmods/
```

Start by listing them:

```bash
ls -lah /var/log/akmods/
```

Then inspect the relevant log for the affected module or kernel.

Build errors here can explain why an external module is missing.

---

## Step 8 — Inspect Kernel Messages

Use:

```bash
journalctl -k -b
```

When the module name is known:

```bash
journalctl -k -b | grep -i <module>
```

Signature-related messages can also be searched for:

```bash
journalctl -k -b | grep -Ei 'module|signature|secure boot|verification'
```

The exact kernel message is often more useful than assuming the cause from the visible symptom.

---

## Common Failure: Module Missing

If:

```bash
modinfo <module>
```

cannot find the module, possible causes include:

* The module was never installed
* The module was not built
* `akmods` failed
* The module was built for another kernel
* Required packages are missing

The investigation should begin with the build and installation path.

---

## Common Failure: Module Exists but Does Not Load

If `modinfo` works but loading fails, possible causes include:

* Signature problems
* Kernel/module incompatibility
* Missing dependencies
* Module configuration issues
* Hardware-specific problems

Secure Boot should not automatically be assumed to be the cause.

---

## Common Failure: Unsigned Module

If:

```bash
modinfo -F signer <module>
```

returns no value, the module may not contain signer information.

If signature enforcement is active, the kernel may reject the module.

The next step is to determine why the module was not signed.

---

## Common Failure: Unknown Signing Key

A module can have a valid signature while being signed by a key that the kernel does not trust.

In that case:

```text
Valid signature
      ↓
Signing key
      ↓
Not trusted
```

The cryptographic signature itself may be valid, but the kernel cannot use it to establish trust.

---

## Common Failure: Invalid Signature

An invalid signature indicates that the module contents and signature do not match.

Possible causes include:

* Module modification after signing
* File corruption
* Incorrect signing process
* Packaging or build steps that altered the module

The module must be rebuilt or correctly signed again rather than simply enrolling another unrelated key.

---

## Common Failure: Wrong Kernel

Always check:

```bash
uname -r
```

An external module may exist on disk while having been built for a different kernel.

For example:

```text
Running kernel
    ↓
Kernel B

Module
    ↓
Built for Kernel A
```

This is a compatibility problem, not necessarily a Secure Boot problem.

---

## Understanding Trusted Keys

The Linux kernel maintains trusted public keys through its key infrastructure.

The kernel documentation describes `.builtin_trusted_keys` as one of the keyrings containing trusted public keys used for verification. Other trust sources can also be available depending on kernel configuration and platform.

The exact contents of the keyrings vary between systems.

For that reason, troubleshooting should focus on the actual trust configuration of the running system rather than assuming that every certificate visible on disk is trusted.

---

## A Certificate File Is Not Automatically Trusted

Suppose a certificate exists as:

```text
/path/to/key.der
```

The existence of that file does not prove that the key is enrolled or trusted.

These are separate states:

```text
Certificate file
      ↓
MOK enrollment
      ↓
Trusted key available to kernel
```

The precise relationship depends on the boot and kernel trust configuration.

This distinction is particularly important when troubleshooting manually generated signing keys.

---

## Advanced Keyring Inspection

Advanced troubleshooting may require inspecting the kernel's keyrings.

The kernel documentation identifies `.builtin_trusted_keys` as one of the trusted keyrings.

However, keyring inspection should be treated as an advanced diagnostic step.

Do not modify kernel keyrings simply because a module fails to load.

First establish:

1. Which module is failing
2. Which kernel is running
3. Whether the module exists
4. Whether it was successfully built
5. Whether it is signed
6. Which key signed it
7. Whether the key is trusted

---

## Manual Module Signing

Linux provides `scripts/sign-file` for manually signing kernel modules.

The general syntax is:

```bash
scripts/sign-file <hash> <private-key> <public-key> <module>
```

For example:

```bash
scripts/sign-file sha256 \
    private.key \
    public.x509 \
    module.ko
```

The kernel documentation describes `scripts/sign-file` as the standard utility for manually adding module signatures.

Manual signing is useful for understanding the underlying mechanism.

For normal Fedora external-driver workflows, however, the distribution's `akmods` integration should generally be preferred over manually signing every module.

---

## Signature Placement

A module signature is appended to the module file.

Conceptually:

```text
┌─────────────────────────┐
│ Kernel module           │
│                         │
│ Code + metadata         │
├─────────────────────────┤
│ Module signature        │
└─────────────────────────┘
```

Because the signature covers the module contents, modifying the module after signing invalidates the signature.

This is why signing should happen after the module has been finalized.

---

## Security Considerations

### Protect the Private Key

The private signing key is sensitive and should be protected against unauthorized access.

### Verify Keys Before Enrollment

Do not enroll a certificate simply because an installation script asks you to.

Determine:

* Where the key came from
* Who controls it
* What software it signs
* Why the system needs to trust it

### Do Not Disable Secure Boot as the First Response

If a module fails to load, disabling Secure Boot may hide the actual problem.

A better approach is to identify whether the failure is caused by:

* Build
* Installation
* Compatibility
* Dependencies
* Signature
* Trust
* Hardware or configuration

Then address the actual cause.

---

## Practical Diagnostic Checklist

When a third-party module fails:

```text
[ ] Is Secure Boot enabled?
[ ] What kernel is currently running?
[ ] Does the module exist?
[ ] Is the module appropriate for this kernel?
[ ] Does it contain signature information?
[ ] Who signed it?
[ ] Is the signing key trusted?
[ ] Did akmods successfully build it?
[ ] Are there errors in /var/log/akmods/?
[ ] What does the kernel journal report?
```

Useful read-only commands include:

```bash
mokutil --sb-state
uname -r
modinfo <module>
modinfo -F filename <module>
modinfo -F signer <module>
modinfo -F sig_id <module>
modinfo -F sig_key <module>
mokutil --list-enrolled
akmods --status
journalctl -k -b
```

The principle is simple:

> **Inspect first. Change later.**

---

## Summary

The important concepts from this chapter are:

1. **Kernel modules** are loadable pieces of kernel functionality.
2. **External modules** may need to be rebuilt for each installed kernel.
3. **Module signing** provides cryptographic authentication and integrity checking.
4. **The private key** creates the signature and must be protected.
5. **The public key** is used to verify the signature.
6. **A signature is not automatically trusted.**
7. **The kernel performs the final signature verification.**
8. **MOK can provide an owner-controlled trusted public key.**
9. **`akmods` builds and installs external kernel modules.**
10. **`akmods` and module signing solve different problems.**
11. **Kernel updates can require external modules to be rebuilt.**
12. **`modinfo`, `mokutil`, `akmods`, and `journalctl` provide useful diagnostic information.**

The next chapter will move from concepts to the practical Fedora workflow: **MOK keys, `kmodgenca`, enrollment, and verifying the resulting trust relationship**.
---

[← Previous: MOK](02-mok.md) | [Next: Practical MOK →](04-practical-mok.md)

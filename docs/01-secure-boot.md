# Secure Boot

## Introduction

**Secure Boot** is a security feature provided by the UEFI firmware that helps ensure that only trusted software is loaded during the system boot process.

On a Fedora system, Secure Boot is part of a chain of trust that begins in the system firmware and continues through the bootloader, kernel, and, where applicable, kernel modules.

Understanding this chain is important before working with **Machine Owner Keys (MOK)**, third-party kernel modules, or tools such as `akmods`.

> **Important:** Secure Boot, MOK enrollment, and kernel module signing are related concepts, but they are not the same thing.

---

## What Is Secure Boot?

Secure Boot is a UEFI security mechanism designed to prevent unauthorized or modified boot software from being executed during the early stages of system startup.

When Secure Boot is enabled, the firmware verifies the digital signature of the next component in the boot chain before executing it.

In simplified form:

```text
UEFI Firmware
      │
      ▼
Bootloader
      │
      ▼
Kernel
      │
      ▼
Operating System
```

Each stage can establish trust in the next stage according to the platform's configured trust model.

The purpose is not to determine whether software is "good" or "bad" in general. Instead, Secure Boot verifies whether the software is signed by a key that is trusted for that stage of the boot process.

---

## The Role of UEFI

**UEFI (Unified Extensible Firmware Interface)** is the firmware interface used by modern computers to initialize hardware and start the operating system.

Secure Boot is implemented at the UEFI firmware level.

This means that Secure Boot operates **before Fedora itself has fully started**.

The firmware maintains databases of trusted and prohibited signing information. These databases are used when deciding whether a signed EFI executable is allowed to run.

A simplified model is:

```text
Firmware
   │
   ├── Trusted signing information
   │
   ├── Prohibited signing information
   │
   ▼
Verify EFI boot component
   │
   ├── Trusted → Execute
   │
   └── Not trusted → Reject
```

The exact trust chain can vary depending on the platform and configuration, but the important point is that the firmware is the first authority involved in Secure Boot verification.

---

## What Does Secure Boot Verify?

Secure Boot primarily verifies **digitally signed executable components** involved in the boot process.

For a typical Fedora installation, this can include components such as:

* UEFI boot applications
* Fedora's bootloader components
* The Linux kernel, depending on the boot configuration and trust chain

The signature provides a way to verify that the component was signed by a trusted key and that its contents have not been modified since signing.

This provides two important properties:

### Authenticity

The signature can establish that the software was signed by an entity whose signing key is trusted.

### Integrity

Changing the signed software invalidates the signature.

Therefore, Secure Boot is fundamentally about establishing **cryptographic trust and integrity during boot**.

---

## Where Does Fedora Fit?

Fedora provides its own signed components that participate in the Secure Boot chain.

A simplified Fedora boot sequence can be represented as:

```text
UEFI Firmware
      │
      │ verifies trusted EFI component
      ▼
Fedora Bootloader
      │
      │ continues the trusted boot chain
      ▼
Linux Kernel
      │
      ▼
Fedora Userspace
```

The exact implementation details depend on the Fedora release, boot configuration, and hardware firmware.

For this reason, it is useful to think of Secure Boot as a **chain of trust**, rather than a single switch that simply "protects Fedora."

---

## Where Does MOK Fit?

**MOK (Machine Owner Key)** is a mechanism commonly used on Linux systems to allow the system owner to add additional trusted signing keys.

MOK is particularly useful when software needs to be trusted but is not signed by a key already included in the platform's default trust configuration.

This is important for third-party kernel modules.

For example, a system may need to load a kernel module that was built locally or provided by a third-party driver.

The module can be digitally signed with a private key, while the corresponding public key can be enrolled as a MOK.

The simplified relationship is:

```text
MOK private key
      │
      │ signs
      ▼
Kernel module
      │
      │ verified using
      ▼
MOK public key
```

The private key should remain protected and should never be distributed with the public key.

---

## MOK Is Not the Same as Secure Boot

It is important to keep these concepts separate.

### Secure Boot

A UEFI security mechanism that establishes trust during the boot process.

### MOK

A mechanism that allows additional owner-controlled keys to participate in the system's trust model.

### Kernel Module Signing

The process of digitally signing a kernel module so that its authenticity and integrity can be verified.

These concepts work together, but they solve different parts of the problem.

A useful simplified model is:

```text
             Secure Boot
                  │
                  ▼
          Establish boot trust
                  │
                  ▼
              Fedora
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼
     Fedora-signed     Third-party
       components       components
                           │
                           ▼
                    Module signing
                           │
                           ▼
                         MOK
```

---

## Why Do Kernel Modules Matter?

A Linux kernel module is code that can be loaded into the running kernel.

Modules are commonly used for:

* Hardware drivers
* Filesystem support
* Virtualization features
* Third-party drivers
* Other functionality that can be provided as loadable kernel code

Because a kernel module executes with kernel-level privileges, allowing an untrusted module to load could undermine the security guarantees provided by Secure Boot.

For this reason, systems using Secure Boot may enforce module signature verification.

This creates another part of the trust chain:

```text
Signed kernel
      │
      ▼
Running kernel
      │
      ▼
Verify module signature
      │
      ├── Trusted → Module may load
      │
      └── Not trusted → Module rejected
```

The exact enforcement behavior depends on the kernel configuration and platform trust setup.

---

## Why Third-Party Drivers Can Require MOK

A driver supplied outside the standard Fedora kernel packages may not be signed by a key that the system already trusts.

This can happen with:

* Third-party hardware drivers
* Locally built kernel modules
* Modules built through external packaging systems
* Modules generated by tools such as `akmods`

When Secure Boot is enabled, the module may need to be signed with a trusted key.

A common approach is:

```text
Third-party source/package
          │
          ▼
   Build kernel module
          │
          ▼
      Sign module
          │
          ▼
Enroll public key as MOK
          │
          ▼
      Load module
```

This is one of the main reasons MOK becomes relevant on Fedora systems using third-party kernel modules.

---

## Secure Boot vs. MOK vs. Module Signing

The following table summarizes the distinction:

| Component          | Purpose                                                                            |
| ------------------ | ---------------------------------------------------------------------------------- |
| **Secure Boot**    | Establishes trust during the UEFI boot process                                     |
| **UEFI firmware**  | Performs the initial verification of boot components                               |
| **MOK**            | Provides an owner-controlled mechanism for additional trusted keys                 |
| **Module signing** | Cryptographically signs kernel modules                                             |
| **Kernel**         | Can enforce signature requirements for modules                                     |
| **akmods**         | Builds external kernel modules, which may need signing when Secure Boot is enabled |

These components should not be treated as interchangeable.

---

## The Trust Chain

A useful way to understand the complete picture is to think in terms of a chain of trust:

```text
UEFI Firmware
      │
      ▼
Trusted Boot Component
      │
      ▼
Fedora Bootloader
      │
      ▼
Linux Kernel
      │
      ▼
Kernel Module Verification
      │
      ├── Fedora-trusted key
      │
      └── Owner-enrolled MOK
```

The purpose of this chain is to prevent unauthorized code from being silently introduced into privileged parts of the boot and kernel environment.

The important question at each stage is:

> **Who signed this component, and does the system trust that signing key?**

---

## What MOK Does Not Mean

MOK does not mean that Secure Boot has been disabled.

It also does not mean that every module is automatically trusted.

Instead, enrolling a MOK adds a specific public key to the system's trusted key material.

A module signed with the corresponding private key can then be recognized as trusted where that key is accepted.

This distinction becomes particularly important when troubleshooting module-loading problems.

---

## Summary

Secure Boot is a UEFI-based mechanism for establishing trust during system startup.

On Fedora, the relevant concepts can be separated into three layers:

1. **Secure Boot** — establishes trust during the boot process.
2. **MOK** — allows the system owner to add an additional trusted signing key.
3. **Kernel module signing** — allows kernel modules to be cryptographically verified before they are loaded.

Understanding these distinctions makes it easier to understand why a third-party driver can work normally with Secure Boot disabled but fail to load when Secure Boot is enabled.

The next chapters will move from these concepts to the practical Fedora workflow, including MOK enrollment, module signing, `akmods`, verification, and troubleshooting.

---

[Next: MOK →](02-mok.md)

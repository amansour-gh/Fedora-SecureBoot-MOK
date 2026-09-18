# Fedora Secure Boot & MOK

A practical guide for managing **Secure Boot, MOK enrollment, and third-party kernel modules on Fedora Linux**.

## About

This project documents practical and reliable ways to understand, configure, verify, and troubleshoot Secure Boot on Fedora.

The guide focuses on situations where Fedora needs to load kernel modules that are not included in the standard Fedora kernel, such as third-party drivers and modules built locally.

## Topics

* Secure Boot fundamentals
* Machine Owner Key (MOK)
* Kernel module signing
* `akmods`
* Third-party kernel modules
* NVIDIA drivers
* Kernel updates
* Secure Boot verification
* Troubleshooting and recovery

## Goals

The goal is to explain:

* What Secure Boot is doing
* Why MOK may be required
* How Fedora handles third-party kernel modules
* How to verify that modules are properly signed
* How to diagnose problems after kernel or driver updates
* How to recover from common Secure Boot and MOK problems

The guide prefers **understanding and verification over blindly running commands**.

## Supported System

The primary target is:

* Fedora Linux
* UEFI systems
* Secure Boot enabled systems
* `x86_64` architecture

The instructions will be tested against current Fedora releases and may require adjustments for older versions.

## Documentation

The documentation is organized from core concepts to practical configuration and troubleshooting:

1. [Secure Boot](docs/01-secure-boot.md) — Secure Boot fundamentals and how the trust chain works.
2. [MOK](docs/02-mok.md) — Machine Owner Key enrollment and management.
3. [Module Signing & akmods](docs/03-module-signing-akmods.md) — Kernel module signing and the role of `akmods`.
4. [Practical MOK](docs/04-practical-mok.md) — Practical MOK setup, verification, and recovery.
5. [akmods & Third-Party Modules](docs/05-akmods-third-party-modules.md) — Building, signing, and troubleshooting external kernel modules.
6. [NVIDIA + Secure Boot](docs/06-nvidia-secure-boot.md) — NVIDIA driver installation, module signing, Secure Boot verification, and troubleshooting.

The chapters are designed to build on each other. Chapters 5 and 6 focus on practical third-party kernel module workflows and NVIDIA-specific troubleshooting.

## Status

🚧 Work in progress.

Six documentation chapters are currently available. The project is being developed as a focused Fedora reference for Secure Boot, MOK, and third-party kernel modules.

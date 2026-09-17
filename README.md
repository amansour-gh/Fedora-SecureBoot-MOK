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

Documentation will be added gradually as each topic is tested and verified.

## Status

🚧 Work in progress.

This project is being developed as a focused Fedora reference for Secure Boot, MOK, and third-party kernel modules.

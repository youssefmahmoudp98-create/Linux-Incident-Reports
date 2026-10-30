# Linux-Incident-Reports

A collection of Linux incident reports and root cause analyses (RCAs) documenting real-world troubleshooting methodology.

## Purpose

This repository documents the methodology for investigating, diagnosing, and resolving system-level incidents. It serves as a portfolio of my troubleshooting process, demonstrating a focus on:

* Progressive evidence-based investigation (rather than blind cleanup)
* Understanding the relationship between application, OS, and security layers

## Current Reports

* [Snap Discord Disk Exhaustion](./disk-exhaustion-incident.md) — Diagnosed a 31GB /var/log exhaustion caused by repeated AppArmor denials from a Snap-confined application.
* [Chromium & Electron Renderer CPU Exhaustion](./cpu-exhaustion-incident.md) — Investigated recurring 100% CPU usage and thermal throttling caused by runaway Chromium and Electron renderer processes (futex spinlocks).

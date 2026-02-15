---
title: "BPFBox"
date: "2023-06-01T00:00:00Z"
description: "Process-level mandatory access control for Linux using eBPF and LSM hooks."
tags: ["ebpf", "security", "linux", "go"]
repo: "https://github.com/willfindlay/bpfbox"
status: "archived"
featured: true
---

BPFBox is a prototype process confinement system for Linux that uses eBPF programs attached to LSM (Linux Security Module) hooks to enforce mandatory access control policies.

## How It Works

BPFBox attaches eBPF programs to LSM hooks in the kernel. When a process attempts a security-sensitive operation — opening a file, creating a network socket, executing a binary — the eBPF program checks the operation against a policy loaded into BPF maps.

Policies are defined in a simple YAML format:

```yaml
profile: /usr/bin/myapp
rules:
  - action: allow
    object: /var/log/myapp/**
    operations: [read, write, append]
  - action: allow
    object: /etc/myapp/config.yaml
    operations: [read]
  - action: deny
    object: /etc/shadow
    operations: [read]
```

## Architecture

The system consists of three components:

1. **eBPF programs** attached to LSM hooks (`file_open`, `socket_create`, `bprm_check_security`, etc.)
2. **BPF maps** storing per-process policies, loaded from userspace
3. **A Go daemon** that manages policies, handles logging, and provides a CLI interface

## Research Context

BPFBox was developed as part of my Master's thesis research at Carleton University. It explores how eBPF can provide flexible, low-overhead security enforcement without modifying the kernel or requiring custom kernel modules.

The key insight is that eBPF's safety guarantees (verified programs, bounded execution, no kernel crashes) make it practical to deploy custom security policies in production without the risks traditionally associated with kernel-level security tools.

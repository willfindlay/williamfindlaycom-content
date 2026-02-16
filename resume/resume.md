---
title: "Resume"
---

# William Findlay

Software Engineer · Kernel Hacker · Security Researcher · BPF Developer

Senior Software Engineer at Cisco with 5 years building runtime security infrastructure in Go and eBPF. Core contributor to [Tetragon](https://github.com/cilium/tetragon), the open-source CNCF eBPF security platform powering Cisco Hypershield. Published researcher with upstream contributions to the Linux kernel's BPF subsystem.

## Experience

**Senior Software Engineer**
Cisco Systems, Inc., Kanata, ON (Remote) — 2024 – Present

- Own and maintain Tetragon's real-time process lifecycle tracking subsystem in Go and eBPF, forming the foundation of Cisco Hypershield's runtime security enforcement
- Designed and implemented system call profiling capabilities in Tetragon, using tracepoint and kprobe-based eBPF programs to model per-process syscall behavior
- Drove technical continuity on [Tetragon](https://github.com/cilium/tetragon) through Cisco's acquisition of Isovalent, maintaining development velocity on a core CNCF security project
- Championed responsible AI-assisted development within the enterprise engineering team, establishing agent-centric documentation standards that streamline onboarding for both human developers and AI agents

**Software Engineer**
Isovalent, Inc., Kanata, ON (Remote) — 2021 – 2024

- Joined the [Tetragon](https://github.com/cilium/tetragon) team as an early member, building Isovalent's runtime security and observability platform in Go and eBPF
- Shipped layer 7 protocol parsers in eBPF and SIEM ingestion filtering for enterprise customers
- Co-authored upstream patches for the Linux kernel's BPF networking stack
- Overhauled the project's testing framework and CI pipeline
- Mentored new engineers and helped scale the team through a period of rapid growth

**Teaching Assistant, Operating Systems & Distributed Systems**
Carleton University, Ottawa, ON — 2018 – 2021

- Nominated three times for the [Outstanding Teaching Assistant Award](https://carleton.ca/tasupport/taawards/edc-outstanding-ta-awards/)
- Developed course tutorials on eBPF, kernel memory management, concurrency, and rootkits
- Led tutorials and created coursework for classes of up to 50 students

## Education

**Doctor of Philosophy in Computer Science** *(left to join the founding Tetragon team at Isovalent)*
Carleton University, Ottawa, Canada — Sept. 2021 – Dec. 2021

**Master of Computer Science**
Carleton University, Ottawa, Canada — Sept. 2020 – Aug. 2021

- CGPA: 12.00 (A+) — Accelerated Master's Program
- Nominated for a Senate Medal for Academic Achievement
- Thesis: [A Practical, Flexible, and Lightweight Confinement Framework in eBPF](https://www.cisl.carleton.ca/~will/written/techreport/mcs-thesis.pdf)

**B.Sc. Computer Science, Honours**
Carleton University, Ottawa, Canada — Sept. 2015 – Apr. 2020

- CGPA: 11.05 (A) — Accelerated Master's Program
- Graduated with High Distinction, Dean's Honour List
- Thesis: [Host-Based Anomaly Detection with Extended BPF](https://www.cisl.carleton.ca/~will/written/coursework/undergrad-ebpH-thesis.pdf)

## Skills

- **Linux Kernel**: eBPF (kprobes, tracepoints, LSM, XDP), kernel module development, libbpf, bcc
- **Systems Programming**: Go, C, Rust, Python
- **Security**: Runtime enforcement, process confinement, intrusion detection, syscall profiling
- **Research**: Published at ACM CCSW, invited speaker at IBM Research and eBPF Summit
- **AI-Assisted Development**: Claude Code power user, custom agent orchestration, prompt engineering, automated workflow design
- **Languages**: English, French

## Research

**Research Assistant**
Carleton University, Ottawa, Canada — Apr. 2019 – Aug. 2021

- Member of the [CCSL/CISL](https://www.cisl.carleton.ca/) research group
- Researched Extended BPF for runtime security within the Linux kernel
- Co-supervised by [Dr. Anil Somayaji](https://people.scs.carleton.ca/~soma/) and [Dr. David Barrera](https://dbarrera.xyz/)
- Designed and developed ebpH, an anomaly detection system for Linux using eBPF
  - Subject of undergraduate [Honours Thesis](https://www.cisl.carleton.ca/~will/written/coursework/undergrad-ebpH-thesis.pdf)
- Designed and developed bpfbox, a process confinement tool for Linux using eBPF
  - [Published at ACM CCSW 2020](https://dl.acm.org/doi/10.1145/3411495.3421358)

## Awards

- Nominee, Senate Medal for Academic Achievement, Carleton University (2021)
- Three-time nominee, Outstanding Teaching Assistant Award, Carleton University (2019 – 2021)
- Dean's Honour List, Carleton University (2019, 2020)
- Multiple academic scholarships and research assistantships, Carleton University (2015 – 2021)

## Presentations and Invited Talks

**bpfbox: Simple Precise Process Confinement with eBPF**
Invited Talk, IBM Research — Security and Privacy — Dec. 2020

**bpfbox: Simple Precise Process Confinement with eBPF**
Conference Presentation, ACM CCSW 2020 — Nov. 2020

**bpfbox: Simple Precise Process Confinement with eBPF and KRSI**
Lightning Talk, [First Annual eBPF Summit](https://ebpf.io/summit-2020/#lightning-talks) — Oct. 2020

## Publications

### Conference Proceedings

\[1\] William Findlay, Anil Somayaji, and David Barrera. "bpfbox: Simple Precise Process Confinement with eBPF". In: *Proceedings of the 2020 ACM SIGSAC Conference on Cloud Computing Security Workshop*. CCSW'20. Virtual Event, USA: Association for Computing Machinery, 2020, pp. 91–103. [DOI: 10.1145/3411495.3421358](https://doi.org/10.1145/3411495.3421358).

### Technical Reports and Theses

\[1\] William Findlay. "Host-Based Anomaly Detection with Extended BPF". Honours Thesis. Carleton University, 2020. [URL](https://www.cisl.carleton.ca/~will/written/coursework/undergrad-ebpH-thesis.pdf).

## Open-Source Software

### Creator/Maintainer

**BPFContain** — Experimental Container Security Mechanism using eBPF

- Container security framework using eBPF programs and LSM hooks
- Tech stack: Rust, libbpf-rs, eBPF
- Subject of Master's Thesis research
- [Source code](https://github.com/willfindlay/bpfcontain-rs)

**bpfbox** — eBPF-Based Process Confinement Mechanism

- First eBPF-based policy enforcement engine with a high-level policy language for process confinement
- [Published at ACM CCSW 2020](https://dl.acm.org/doi/10.1145/3411495.3421358)
- [Source code](https://github.com/willfindlay/bpfbox)

**ebpH** — eBPF-Based Intrusion Detection System

- Intrusion detection system for Linux that establishes per-executable system call profiles to detect anomalies
- [Source code](https://github.com/willfindlay/ebpH)

### Core Contributor

**Tetragon** — eBPF-Based Security Observability and Runtime Enforcement (CNCF)

- One of the first maintainers; core contributor since the pre-open-source phase (2021)
- Own the real-time process lifecycle tracking and syscall profiling subsystems
- [Source code](https://github.com/cilium/tetragon)

### Contributor

**bcc** — eBPF Programming Framework for Python

- Implemented ringbuf eBPF map support, enhanced LSM probe support, Python support for stack and queue eBPF maps
- [Source code](https://github.com/iovisor/bcc)

**libbpf-rs** — Rust Implementation of libbpf

- Implemented ringbuf eBPF map support, enhanced LSM probe support, bugfixes and API improvements
- [Source code](https://github.com/libbpf/libbpf-rs)

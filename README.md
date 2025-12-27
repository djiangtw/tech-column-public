# Tech Column

**In-Depth System Architecture and Hardware Design**

[![Language](https://img.shields.io/badge/Language-繁體中文-blue)]()
[![Series](https://img.shields.io/badge/Series-6-blue)]()
[![Articles](https://img.shields.io/badge/Articles-90-blue)]()
[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey)](https://creativecommons.org/licenses/by/4.0/)
[![Author](https://img.shields.io/badge/Author-Danny%20Jiang-orange)]()
[![Updated](https://img.shields.io/badge/Updated-Dec%202025-green)]()

---

## 📖 About This Column

Tech Column is a technical writing project focused on system architecture, hardware design, and performance optimization. The goal is to explain complex technical concepts clearly using vivid analogies and real-world cases, helping readers understand not just "what" but "why."

**About the Cases**: All case scenarios in this column are **mock scenarios**, written based on industry best practices with all sensitive information removed. All content complies with professional ethics and NDA requirements.

### Features

- **Vivid Analogies**: Understand Cache through libraries, Associativity through parking lots, NoC through city traffic
- **Real-World Cases**: Practical problems and solutions from 20+ years of industry experience
- **Progressive Learning**: From beginner to advanced, systematically building knowledge
- **Practice-Oriented**: Not just theory, but actionable optimization advice and design principles

---

## 📊 Project Statistics

| Series | Articles | Word Count |
|--------|----------|------------|
| Cache Architecture | 6 | ~20,800 |
| Network-on-Chip | 6 | ~14,100 |
| Storage Architecture | 9 | ~39,200 |
| Embedded RTOS | 8 | ~24,000 |
| Bluetooth & IoT | 21 | ~70,000 |
| Building danieRTOS | 40 | ~170,000 |

**Total**: 90 articles, ~338,100 words (Traditional Chinese)

---

## 📚 Article Series

### 1. Cache Architecture Series

Deep dive into CPU Cache design and optimization, from basics to practice.

1. [Cache Basics: Understanding CPU Cache Through Libraries](topics/cache-architecture/01-cache-basics.md)
2. [Understanding Cache Associativity: The Wisdom of Parking Lots](topics/cache-architecture/02-cache-associativity.md)
3. [Modern CPU Cache Architecture: Design Philosophy from L1 to L3](topics/cache-architecture/03-modern-cache-design.md)
4. [Cache Coherency and MESI Protocol: Consistency Challenges in Multi-Core Era](topics/cache-architecture/04-cache-coherency-mesi.md)
5. [Cache Performance Optimization in Practice](topics/cache-architecture/05-cache-optimization.md)
6. [False Sharing and Multi-Threading Optimization: The Invisible Performance Killer](topics/cache-architecture/06-false-sharing.md)

---

### 2. Network-on-Chip Series

Exploring on-chip communication architecture, from Bus to Network evolution.

1. [Network-on-Chip Introduction: Evolution from Bus to Network](topics/network-on-chip/01-noc-introduction.md)
2. [NoC Topology Analysis with Graph Theory](topics/network-on-chip/02-topology-graph-theory.md)
3. [NoC Routing Algorithms and Deadlock Avoidance](topics/network-on-chip/03-routing-deadlock.md)
4. [Router Microarchitecture Design: From Pipeline to Hardware](topics/network-on-chip/04-router-microarchitecture.md)
5. [NoC and Cache Coherency Integration](topics/network-on-chip/05-noc-cache-coherency.md)
6. [NoC and Advanced Packaging: Breaking Physical Boundaries](topics/network-on-chip/06-noc-advanced-packaging.md)

---

### 3. Storage Architecture Series

Complete perspective from hardware to software on modern storage systems.

1. [Storage Systems Introduction: From HDD to SSD](topics/storage-architecture/01-introduction.md)
2. [SATA and AHCI: Deep Dive into Traditional Storage Interfaces](topics/storage-architecture/02-sata-ahci.md)
3. [PCIe Architecture: Foundation of High-Speed Interconnect](topics/storage-architecture/03-pcie.md)
4. [NVMe Protocol: Interface Born for SSDs](topics/storage-architecture/04-nvme.md)
5. [CXL Technology: Fusion of Memory and Storage](topics/storage-architecture/05-cxl.md)
6. [FTL Deep Dive: The Soul of SSDs](topics/storage-architecture/06-ftl.md)
7. [GC and Wear Leveling: SSD Longevity Secrets](topics/storage-architecture/07-gc-wear-leveling.md)
8. [Error Correction Codes: Guardians of Data Integrity](topics/storage-architecture/08-error-correction.md)
9. [Advanced Topics: ZNS, Computational Storage](topics/storage-architecture/09-advanced-topics.md)

---

### 4. Embedded RTOS Series

Practice-oriented embedded RTOS development with FreeRTOS + RISC-V.

1. [RTOS Introduction: Why Real-Time Operating Systems](topics/embedded-rtos/01-rtos-introduction.md)
2. [Scheduler Deep Dive: The Art of Task Scheduling](topics/embedded-rtos/02-scheduler-deep-dive.md)
3. [Interrupt Handling: The Heartbeat of Real-Time Systems](topics/embedded-rtos/03-interrupt-handling.md)
4. [Memory Management: From heap_1 to heap_5](topics/embedded-rtos/04-memory-management.md)
5. [Debugging with GDB + QEMU](topics/embedded-rtos/05-debugging-with-gdb-qemu.md)
6. [RTOS SMP: Multi-Core Challenges](topics/embedded-rtos/06-rtos-smp.md)
7. [Context Switch Assembly Deep Dive](topics/embedded-rtos/07-context-switch-assembly.md)
8. [RISC-V Privilege Modes: M/S/U Mode](topics/embedded-rtos/08-privilege-modes.md)

---

### 5. Bluetooth & IoT Series

BLE protocol stack, wireless communication, IoT system integration.

See [topics/bluetooth-wireless-iot/](topics/bluetooth-wireless-iot/) for complete article list (21 articles).

---

### 6. Building danieRTOS Series

Building a RISC-V RTOS from scratch, narrative-style writing, 40 complete tutorials.

**danieRTOS** is an educational minimal RTOS running on RISC-V architecture.

| Version | Alias | Chapters | Core Features |
|---------|-------|----------|---------------|
| v0.x | Nano | 01-12 | Basic RTOS: Task, Scheduler, Semaphore, Mutex, Queue |
| v1.x | Secure | 13-19 | User Mode: PMP, Syscall, Fault Handling |
| v2.x | MSMP | 20-30 | SMP: Spinlock, IPI, Multi-core Scheduler |
| v3.x | SMP | 31-40 | Integration: SMP + User Mode + Fault Isolation |

See [topics/building-daniertos/README.md](topics/building-daniertos/README.md) for complete article list.

---

## 🎯 Target Audience

This column is suitable for:

- **System Software Engineers**: Understanding how hardware affects software performance
- **Embedded Engineers**: RTOS, drivers, firmware development
- **Hardware Engineers**: CPU, SoC design and verification
- **IoT Developers**: Bluetooth, wireless communication, IoT development
- **Computer Architecture Students**: Learning system architecture in real-world contexts

**Prerequisites**:

- Basic computer organization concepts
- Understanding of CPU, memory, bus components
- C programming experience (required for some series)

---

## 📄 License

**Copyright © 2025 Danny Jiang**

All articles are licensed under **Creative Commons Attribution 4.0 International License (CC BY 4.0)**.

**You are free to**:

- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material for any purpose, including commercial

**Under the following terms**:

- **Attribution** — You must give appropriate credit, provide a link to the license, and indicate if changes were made

**License**: <https://creativecommons.org/licenses/by/4.0/>

---

## 📖 How to Use This Column

### Online Reading

Browse Markdown files directly on GitHub, starting from the first article of each series.

### Offline Reading

Clone this repository:

```bash
git clone https://github.com/djiangtw/tech-column-public.git
cd tech-column-public
```

### Recommended Reading Order

**Hardware Architecture Beginners**: Cache Architecture → Network-on-Chip → Storage Architecture

**Embedded Systems**: Embedded RTOS → Building danieRTOS

**Wireless Communication**: Bluetooth & IoT Series

---

## 🤝 Contributing

This is a read-only public repository. The column is developed in a private repository.

**Feedback Welcome**:

- Open issues for typos, errors, or suggestions
- Discussion and questions are encouraged

**Note**: Pull requests cannot be accepted as this is synced one-way from the private development repository.

---

## 👨‍💻 About the Author

**Danny Jiang**

System software engineer focused on RISC-V architecture, embedded systems, and performance optimization. 20+ years of industry experience, passionate about explaining complex technical concepts through vivid analogies.

**Other Works**:

- [See RISC-V Run: Fundamentals](https://github.com/djiangtw/see-riscv-run-public) - Complete RISC-V Architecture Guide
- [Data Structures in Practice](https://github.com/djiangtw/data-structures-in-practice-public) - Hardware-Oriented Data Structures

---

## 🔗 Links

- **GitHub**: <https://github.com/djiangtw/tech-column-public>
- **Email**: djiang.tw@gmail.com
- **LinkedIn**: [linkedin.com/in/danny-jiang-26359644](https://www.linkedin.com/in/danny-jiang-26359644/)

---

## 📝 Citation

If you cite this column in research, teaching, or articles:

```text
Danny Jiang. (2025). Tech Column: In-Depth System Architecture and Hardware Design.
Licensed under CC BY 4.0. https://github.com/djiangtw/tech-column-public
```

---

**Happy Reading!** 📖

For any questions or suggestions, feel free to contact me through GitHub Issues.

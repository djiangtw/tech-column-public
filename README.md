# Tech Column

**In-Depth System Architecture and Hardware Design**

[![Language](https://img.shields.io/badge/Language-繁體中文-blue)]()
[![Series](https://img.shields.io/badge/Series-6-blue)]()
[![Articles](https://img.shields.io/badge/Articles-93-blue)]()
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
| Storage Architecture | 12 | ~52,000 |
| Embedded RTOS | 8 | ~24,000 |
| Bluetooth & IoT | 21 | ~70,000 |
| Building danieRTOS | 40 | ~170,000 |

**Total**: 93 articles, ~350,900 words (Traditional Chinese)

---

## 📚 Article Series

### 1. Cache Architecture Series (6 articles)

Deep dive into CPU Cache design and optimization, from basics to practice.

Topics: Cache basics, Associativity, Modern cache design (L1-L3), MESI protocol, Performance optimization, False sharing

---

### 2. Network-on-Chip Series (6 articles)

Exploring on-chip communication architecture, from Bus to Network evolution.

Topics: NoC introduction, Topology with graph theory, Routing and deadlock, Router microarchitecture, Cache coherency integration, Advanced packaging

---

### 3. Storage Architecture Series (12 articles)

Complete perspective from hardware to software on modern storage systems.

Topics: HDD to SSD evolution, SATA/AHCI, PCIe architecture, NVMe protocol, CXL technology, FTL, GC and wear leveling, Error correction, ZNS, Database optimization, AI/ML workloads, Cloud storage

---

### 4. Embedded RTOS Series (8 articles)

Practice-oriented embedded RTOS development with FreeRTOS + RISC-V.

Topics: RTOS introduction, Scheduler deep dive, Interrupt handling, Memory management, GDB+QEMU debugging, SMP challenges, Context switch assembly, RISC-V privilege modes

---

### 5. Bluetooth & IoT Series (21 articles)

BLE protocol stack, wireless communication, IoT system integration.

Topics: BLE protocol stack (HCI, L2CAP, ATT/GATT, SMP), PHY/RF, WiFi/BT coexistence, Hardware interfaces (SPI, MIPI, I2C/UART/GPIO), Power optimization, Debugging, Certification, Zigbee comparison, Thread/Matter, AIoT, Security

---

### 6. Building danieRTOS Series (40 articles)

Building a RISC-V RTOS from scratch, narrative-style writing, 40 complete tutorials.

**danieRTOS** is an educational minimal RTOS running on RISC-V architecture.

| Version | Alias | Chapters | Core Features |
|---------|-------|----------|---------------|
| v0.x | Nano | 01-12 | Basic RTOS: Task, Scheduler, Semaphore, Mutex, Queue |
| v1.x | Secure | 13-19 | User Mode: PMP, Syscall, Fault Handling |
| v2.x | MSMP | 20-30 | SMP: Spinlock, IPI, Multi-core Scheduler |
| v3.x | SMP | 31-40 | Integration: SMP + User Mode + Fault Isolation |

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

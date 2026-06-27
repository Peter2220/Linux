# Linux Hardware Discovery and RAS Diagnostics Reference Guide

This reference document synthesizes hardware exploration commands and logs executed on a virtualized Linux node. It translates raw configurations into architectural insights and expands upon underlying hardware diagnostic subsystems like **RAS (Reliability, Availability, and Serviceability)**.

---

## 1. Architectural Summary & System Profile

Based on the parsed command outputs (`lspci`, `lscpu`, `lsscsi`), this system is a **highly provisioned Guest Virtual Machine** running on a **VMware vSphere Hypervisor** host.

### Core Specifications
* **CPU:** Intel(R) Xeon(R) CPU E5-2698 v4 (Broadwell architecture) @ 2.20GHz
* **Compute Allocation:** 12 Logical CPUs grouped as 6 Sockets × 2 Cores/Socket × 1 Thread/Core.
* **Storage Controller:** VMware Paravirtual SCSI (PVSCSI) & VMware SATA AHCI Controller.
* **Network Controller:** VMware VMXNET3 Virtual Ethernet Adapter (10 Gbps capable).
* **Storage Layout:** * `/dev/sda`: Primary VMware Virtual Disk.
  * `/dev/sr0`: Virtual CD/DVD-ROM.

---

## 2. Low-Level Subsystem Breakdown

### 2.1 Peripheral Component Interconnect (`lspci`)
The `lspci` output reveals that the hypervisor is emulating a vintage, highly stable baseboard topology (`Intel 440BX/ZX/DX` chipset) combined with modernized paravirtualized high-performance devices:


```

00:00.0 Host bridge: Intel Corporation 440BX/ZX/DX - 82443BX/ZX/DX Host bridge (rev 01)
...
03:00.0 Serial Attached SCSI controller: VMware PVSCSI SCSI Controller (rev 02)
0b:00.0 Ethernet controller: VMware VMXNET3 Ethernet Controller (rev 01)

```

#### Explanatory Deep Dive: Paravirtualization vs. Emulation
* **Emulated Hardware (e.g., Intel 440BX):** The hypervisor mimics real-world physical microchips legacy components bit-for-bit. This yields maximum OS compatibility but imposes high virtualization overhead.
* **Paravirtualized Drivers (`PVSCSI`, `VMXNET3`):** The VM knows it is a VM. Instead of mimicking registers, these devices use highly optimized rings/shared memory lanes directly with the hypervisor kernel, dropping latency and boosting throughput significantly.

### 2.2 CPU Topography & Performance Matrix (`lscpu`)

```

Architecture:        x86_64
CPU(s):              12
Thread(s) per core:  1
Core(s) per socket:  2
Socket(s):           6

```
* **Hyper-Threading:** Disabled (`Thread(s) per core: 1`). Every assigned CPU operates as an isolated physical execution core unit on the host.
* **Cache Architecture:** * **L1 Data / Instruction:** 384 KiB total spread across 12 instances (32 KiB dedicated per core).
  * **L2 Cache:** 3 MiB total (256 KiB dedicated per core).
  * **L3 Cache:** 300 MiB huge pool shared over 6 instances. 

### 2.3 CPU Vulnerabilities & Mitigations
The Linux kernel monitors and implements microcode/software workarounds for hardware-level speculative execution vulnerabilities. The output confirms active defenses:
* **Meltdown:** Mitigated using **PTI** (Page Table Isolation), which completely splits user and kernel space page tables.
* **Spectre v1/v2:** Mitigated using **IBRS** (Indirect Branch Restricted Speculation) barriers, preventing unauthorized branch target injections across processes.

---

## 3. RAS (Reliability, Availability, Serviceability) & Hardware Error Tracking

The administrative commands track installation and deployment of advanced hardware telemetry via **RASdaemon**.

### 3.1 What is RAS?
In enterprise servers, **RAS** refers to a suite of hardware capabilities aimed at maintaining stable uptime:
* **Reliability:** Detects faults early and corrects transient memory errors before data corruption occurs.
* **Availability:** Uses alternate paths or redundant hardware structures to keep the OS active despite a hardware failure.
* **Serviceability:** Provides clear logs pin-pointing precise components needing hot-swap replacement (e.g., DIMM slot location labels).

### 3.2 Component Breakdown: `rasdaemon` vs `EDAC`
The daemon works tightly with the Linux Kernel's **EDAC (Error Detection and Correction)** subsystem:

```bash
# Installing the daemon
dnf reinstall rasdaemon

# Enabling and running immediately
systemctl enable rasdaemon.service --now

```

1. **Kernel Space (EDAC):** Built-in Linux drivers probe specific CPU memory controllers or PCI buses for hardware indicators. When a single-bit error happens in RAM, the hardware corrects it dynamically via ECC (Error-Correcting Code) and increments an internal counter.
2. **User Space (`rasdaemon`):** A daemon background worker that hooks into the kernel’s `/sys/kernel/debug/tracing/events/ras/` tracepoints. It extracts these events and records them cleanly into a structured local SQLite3 database for long-term reporting.

### 3.3 Practical Administration Commands

| Command | Objective / Use Case |
| --- | --- |
| `rasdaemon --record` | Starts error collection and outputs records directly into an sqlite3 backend store. |
| `ras-mc-ctl --status` | Queries the status of memory-controller EDAC drivers to see if ECC monitoring is running. |
| `ras-mc-ctl --layout` | Displays the physical topology map of DIMM slots, Banks, and Channels. |
| `ras-mc-ctl --summary` | Summarizes all logged corrected errors (CE) and uncorrected errors (UE) on the system. |

*Note: In virtualized guest topologies (like this VMware guest), physical memory details are rarely directly visible inside the VM unless explicit hardware pass-through/vNUMA optimizations are configured by the hypervisor infrastructure administrator.*

---

## 4. Supplementary Hardware Commands Reference

For a complete sysadmin toolbox, the following low-level commands complement your discovery toolkit:

### `lsusb`

* **Purpose:** Enumerates USB controllers and connected universal serial bus layout devices.
* *Context:* Logged an empty response on this host, signifying that no USB controllers or passive host USB root-hubs are exposed to this cluster virtual node.

### `dmidecode`

* **Purpose:** Dumps Desktop Management Interface (DMI) / SMBIOS tables into human-readable configuration data.
* **Usage Example:**
```bash
# View detailed RAM speed, manufacturing part numbers, and slot sizes
dmidecode -t memory

# Extract System and Motherboard identification
dmidecode -t system

```



### `dmesg`

* **Purpose:** Displays the kernel ring buffer logs.
* **Usage Example:**
```bash
# Check for boot-time hardware mapping, disk attachment, or memory faults
dmesg | grep -iE 'error|warn|scsi|pci'

```



---

## 5. Appendix: Sample `dmesg` Hardware Logs

Below is a production-style example of how the kernel ring buffer registers hardware initializations matching the exact footprint of your system profile:

```text
[    0.000000] Linux version 5.14.0-362.8.1.el9_3.x86_64 (mockbuild@builder) (gcc version 11.4.1)
[    0.000000] Command line: BOOT_IMAGE=(hd0,msdos1)/vmlinuz-5.14.0 root=/dev/mapper/rl-root ro crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M resume=/dev/mapper/rl-swap
[    0.000000] CPU0: Intel(R) Xeon(R) CPU E5-2698 v4 @ 2.20GHz (family: 0x6, model: 0x4f, stepping: 0x1)
[    0.000000] Performance Counters: Broadwell PMU driver installed
[    0.000000] x86/CPU: Spectre v1 mitigation: usercopy/swapgs barriers and __user pointer sanitization
[    0.000000] x86/CPU: Spectre v2 mitigation: IBRS microcode
[    0.000000] x86/CPU: Meltdown mitigation: PTI
[    0.050412] Hypervisor detected: VMware
[    0.122340] ACPI: Core revision 20210730
[    0.450128] smp: Bringing up secondary CPUs ...
[    0.510820] x86: Booting SMP configuration:
[    0.512104] .... node  #0, CPUs:      #1  #2  #3  #4  #5  #6  #7  #8  #9 #10 #11
[    0.589144] smp: Brought up 1 node, 12 CPUs
[    1.020419] pci 0000:00:00.0: [8086:7110] type 00 class 0x060000 (Intel 440BX Host Bridge)
[    1.024510] pci 0000:03:00.0: [15ad:07c0] type 00 class 0x010000 (VMware PVSCSI Controller)
[    1.028912] pci 0000:0b:00.0: [15ad:07b0] type 00 class 0x020000 (VMware VMXNET3 Ethernet Controller)
[    1.540112] scsi host0: VMware PVSCSI driver v1.0.7.0-k
[    1.890330] scsi 0:0:0:0: Direct-Access     VMware   Virtual disk     2.0  PQ: 0 ANSI: 6
[    1.892404] sd 0:0:0:0: [sda] 167772160 512-byte logical blocks: (85.8 GB/80.0 GiB)
[    1.895112] sd 0:0:0:0: Attached scsi generic sg0 type 0
[    1.901140] sd 0:0:0:0: [sda] Write Protect is off
[    1.954010] vmxnet3 0000:0b:00.0 ens192: renamed from eth0
[    2.102390] EDAC MC: Ver: 3.0.0
[    2.410982] RAS: Correctable Errors collector initialized.

```

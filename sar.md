# sar Summary & Reference Sheet

## Installation & Setup

To start using `sar`, you must install the `sysstat` utilities package and enable its background collection service engine.

```bash
# Install the sysstat utilities package
dnf install -y sysstat

# Enable and start the background data collection engine immediately
systemctl enable --now sysstat

```

---

## Real System Outputs (Condensed)

### 1. Network Statistics (`sar -n DEV 1 5`)

Monitors network traffic metrics across available interface cards.

```text
Linux 6.12.0-124.8.1.el10_1.x86_64 (localhost.localdomain)      (2 CPU)

09:32:48 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   %ifutil
09:32:49 AM    ens160      7.00     10.00      0.50      1.19      0.00
...
Average:       ens160      5.00      8.20      0.36      0.98      0.00

```

### 2. Disk Storage Metrics (`sar -d 1 5`)

Monitors throughput, processing speeds, and utilization rates across block devices.

```text
Linux 6.12.0-124.8.1.el10_1.x86_64 (localhost.localdomain)      (2 CPU)

09:33:27 AM        DEV       tps     rkB/s     wkB/s     await     %util
09:33:28 AM    nvme0n1      0.00      0.00      0.00      0.00      0.00
...
Average:      nvme0n1      0.00      0.00      0.00      0.00      0.00

```

---

## Expanded Diagnostic Options

*Note: The arguments `1 3` below specify a `1`-second measurement interval repeated a total of `3` times.*

| Command | Performance Focus | Critical Fields Tracked |
| --- | --- | --- |
| `sar -u 1 3` | **CPU Utilization** | `%user`, `%system`, `%iowait`, `%idle` |
| `sar -r 1 3` | **Memory Allocation** | `kbmemfree`, `kbmemused`, `%memused`, `kbbuffers` |
| `sar -q 1 3` | **System Load Queue** | `runq-sz` (Run-queue depth), `ldavg-1`, `ldavg-5` |
| `sar -w 1 3` | **Context Switching** | `proc/s` (Process creation rate), `cswch/s` (Switches) |
| `sar -B 1 3` | **Kernel Paging** | `pgpgin/s` (Page-ins), `pgpgout/s`, `fault/s` (Page faults) |
| `sar -S 1 3` | **Swap Space** | `kbswpfree`, `kbswpused`, `%swpused` |
| `sar -A` | **Full Dump** | Emits all metrics combined (CPU, Mem, I/O, Net, Apps). |

---

## Historical Data Parsing

The `sysstat` daemon automatically dumps binary performance logs locally to `/var/log/sa/saDD` (where `DD` represents the numerical day of the month). Use the `-f` flag to read these archives retroactively:

```bash
# Inspect CPU metrics captured on the 23rd day of the current month
sar -u -f /var/log/sa/sa23

# Filter historical load metrics specifically between 8:00 AM and 12:00 PM
sar -q -s 08:00:00 -e 12:00:00

```

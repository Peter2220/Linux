# Linux Boot Troubleshooting and Kernel Management Guide (Rocky Linux / RHEL)

This guide provides a comprehensive breakdown of troubleshooting kernel boot entries, editing GRUB2 configurations, dealing with the Boot Loader Specification (BLS), and managing UEFI boot variables.
---


## 1. Auditing the Running vs. Default Kernel

When a system boots into an unexpected kernel, the first step is checking what is currently running versus what GRUB intends to boot by default.

### Check Current Running Kernel

```bash
uname -r

```

*Output:* `6.12.0-124.8.1.el10_1.x86_64`

### Check GRUB Default Target Kernel

```bash
grubby --default-kernel

```

*Output:* `/boot/vmlinuz-6.12.0-211.26.1.el10_2.x86_64`

> ⚠️ **Problem Identified:** The system is currently running an older `el10_1` kernel, but the bootloader is configured to load a newer `el10_2` kernel on the next reboot. If the `el10_2` kernel is corrupted or failing, you may need to force a rollback.

---

## 2. Inspecting and Modifying Kernel Boot Entries with `grubby`

The `grubby` tool is the recommended command-line tool in RHEL/Rocky Linux for updating and displaying information about the boot loader configuration files.

### List All Available Boot Entries

```bash
grubby --info ALL

```

This displays defined indexes for your available kernels (including the fallback **Rescue Kernel**):

* **Index 0:** Rocky Linux 10.2 (`vmlinuz-6.12.0-211.26.1.el10_2.x86_64`)
* **Index 1:** Rocky Linux 10.1 (`vmlinuz-6.12.0-124.8.1.el10_1.x86_64`)
* **Index 2:** Rescue Kernel (`vmlinuz-0-rescue-...`)

### Changing the Default Kernel

To force the system to reliably boot into the working `el10_1` kernel, change the default using:

```bash
grubby --set-default=/boot/vmlinuz-6.12.0-124.8.1.el10_1.x86_64

```

---

## 3. Editing GRUB2 Command Line Parameters

If you need to pass specific arguments to the kernel at boot time (e.g., forcing a system break for root password resets or emergency debugging via `rd.break`), modify the `/etc/default/grub` file.

### Step 1: Open `/etc/default/grub`

```bash
vi /etc/default/grub

```

### Step 2: Append the parameter

Locate the `GRUB_CMDLINE_LINUX` line and append your debugging flag (e.g., `rd.break`):

```text
GRUB_CMDLINE_LINUX="crashkernel=2G-64G:256M,64G-:512M resume=UUID=5694c7f7-409f-42c6-82b3-5928adef386c rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet rd.break"

```

### Step 3: Regenerate the GRUB2 Configuration File

> 🛑 **Important Note on Modern UEFI Systems:** Modern enterprise distributions use a stub/wrapper config inside the EFI partition (`/boot/efi/EFI/rocky/grub.cfg`) that redirects processing to the main file.
> **Do not overwrite the wrapper file** or you will break the automatic dynamic configurations.

If you attempt to write directly to the EFI path:

```bash
grub2-mkconfig -o /boot/efi/EFI/rocky/grub.cfg

```

The system will throw an error telling you to use the standard path instead:

> *Running 'grub2-mkconfig...' will overwrite the GRUB wrapper. Please run 'grub2-mkconfig -o /boot/grub2/grub.cfg' instead.*

**Correct Command to Regenerate Configuration:**

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg

```

---

## 4. Deep Dive: Architectural Concepts

### What is the Boot Loader Specification (BLS)?

In your `/etc/default/grub`, you have the parameter: `GRUB_ENABLE_BLSCFG=true`.

Historically, every time a new kernel was installed, scripts would completely rebuild `/boot/grub2/grub.cfg`. This was error-prone. Under **BLS**, the main `grub.cfg` remains static. Instead, individual boot entries are stored as fragments in distinct files under `/boot/loader/entries/`.

* Each kernel update simply drops or deletes a tiny `.conf` fragment file in that directory. GRUB parses these files dynamically at boot time to build the menu.

### Understanding UEFI Wrapper Setup (`/boot/efi/EFI/rocky/grub.cfg`)

Your EFI wrapper configuration looks like this:

```text
search --no-floppy --root-dev-only --fs-uuid --set=dev ebf3049a-cf8e-417c-9c44-80f7c4732cbc
set prefix=($dev)/grub2
export $prefix
configfile $prefix/grub.cfg

```

* **How it works:** This small script runs while the machine is still in UEFI mode. It searches for the drive partition with the unique UUID containing your `/boot` directory, assigns it to the variable `$dev`, points to the `/boot/grub2/` directory, and hands execution off to the main `grub.cfg`.

---

## 5. UEFI Variables and Reinstalling Boot Files

If your system's NVRAM or boot order gets corrupted, or you lose the hardware pathways to boot, use `efibootmgr` and package recovery.

### Managing the NVRAM Boot Entries

```bash
efibootmgr

```

*Output snippet:*

```text
BootCurrent: 0003
BootOrder: 0003,0000,0001,0002
Boot0003* Rocky Linux   HD(1,GPT,...)/\EFI\rocky\shimx64.efi

```

* **`BootCurrent`**: The entry used to boot the active session.
* **`BootOrder`**: The sequential order the motherboard attempts to boot. Here, `0003` (Rocky Linux via `shimx64.efi`) is prioritised.

### Fixing Missing or Corrupted Bootloader/EFI Binaries

If files like `shimx64.efi` or `grubx64.efi` are deleted or corrupted from the `/boot/efi/EFI/rocky/` directory, you can restore them natively from your active repositories by forcing a package re-installation:

```bash
dnf reinstall grub2 shim

```

This restores the mandatory UEFI secure boot chain files directly to your EFI system partition.

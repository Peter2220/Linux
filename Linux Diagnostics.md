# Linux System Administration Diagnostics & Security Auditing Log
---

## 1. Package Installation & Setup

Before running diagnostics or integrity checks, ensure the required tools are installed:

```bash
# Install AIDE (Advanced Intrusion Detection Environment)
dnf install -y aide

# Install system diagnostics and support tools
dnf install -y sos subscription-manager redhat-support-tool
````

---

## 2. Scenario 1: Apache HTTPD Failure (Permission Denied)

### Overview

The Apache HTTP Server (`httpd`) fails to start due to incorrect file permissions on its executable binary.

### Reproduction Steps

```bash
# 1. Check current permissions
ls -ltrh /sbin/httpd
-rwxr-xr-x. 1 root root 568K May 11 03:00 /sbin/httpd

# 2. Remove execute permissions
chmod -x /sbin/httpd

# 3. Verify permission change
ls -ltrh /sbin/httpd
-rw-r--r--. 1 root root 568K May 11 03:00 /sbin/httpd

# 4. Attempt to start service
systemctl start httpd
```

### Error Output

```text
Job for httpd.service failed because the control process exited with error code.
```

### Log Analysis

```bash
journalctl -xeu httpd.service
```

```text
Failed at step EXEC spawning /usr/sbin/httpd: Permission denied
status=203/EXEC
```

From `/var/log/messages`:

```text
Unable to locate executable '/usr/sbin/httpd': Permission denied
```

### Root Cause

* **Error 203/EXEC**: Executable exists but cannot be executed
* **Error 13 (EACCES)**: Permission denied at kernel level
* Missing execute (`+x`) permission on the binary

### Resolution

```bash
chmod +x /usr/sbin/httpd
systemctl restart httpd
```

---

## 3. Scenario 2: File Integrity Violations (AIDE)

### Overview

AIDE detects changes in critical system configuration files, specifically `/etc/ssh/sshd_config`.

### Reproduction Steps

```bash
# 1. Set insecure permissions and initialize AIDE database
chmod 777 /etc/ssh/sshd_config
aide --init

# 2. Restore secure permissions
chmod 600 /etc/ssh/sshd_config

# 3. Run integrity check
aide --check
```

### AIDE Report Summary

```text
Changed entries: 1

File: /etc/ssh/sshd_config
Perm: -rwxrwxrwx -> -rw-------
ACL:  modified
```

### Interpretation

* `f`: Regular file
* `p`: Permission change detected
* `A`: ACL change detected

This indicates a deviation from the stored baseline.

### Resolution

If the change is legitimate, update the AIDE baseline:

```bash
cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

---

## 4. Scenario 3: Root Password Recovery Using `rd.break`

### Overview

Recover access to a system when the root password is lost by booting into an emergency shell.

### Procedure

1. **Interrupt GRUB Boot Menu**

   * Reboot the system
   * Press `e` on the selected kernel entry

2. **Modify Kernel Parameters**

   * Locate the line starting with `linux` or `linux16`
   * Append:

   ```text
   rd.break
   ```

3. **Boot Into Emergency Mode**

   * Press `Ctrl + X` or `F10`

4. **Remount Root Filesystem as Read/Write**

```bash
mount -o remount,rw /sysroot
```

5. **Switch Root Environment**

```bash
chroot /sysroot
```

6. **Reset Root Password**

```bash
passwd
```

7. **Relabel SELinux (Important)**

```bash
touch /.autorelabel
```

8. **Exit and Reboot**

```bash
exit
exit
reboot
```

---

## Summary

This document covered:

* Package preparation for diagnostics and auditing
* Troubleshooting service failures caused by permission issues
* Detecting and validating file integrity changes using AIDE
* Recovering root access via kernel boot interruption

These scenarios represent common and critical Linux system administration tasks related to security, troubleshooting, and recovery.

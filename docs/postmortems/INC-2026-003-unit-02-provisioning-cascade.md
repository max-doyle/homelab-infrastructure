# Incident Post-Mortem: INC-2026-004
## unit-02 Provisioning Cascade: IOMMU DMAR Bus Faults, Virtual Media Severance, & Input Subsystem Failures

| Metadata Field | Value |
| :--- | :--- |
| **Incident ID** | `INC-2026-004` |
| **Target System** | `unit-02` (HP ProLiant MicroServer Gen8) |
| **Severity** | P2 (Deployment Blocker / Host Controller Hang) |
| **Impact** | Storage bus drops, read-only rootfs remounts, and installer aborts |
| **Resolution Status** | Resolved via kernel flags (`intremap=off intel_iommu=off`) and Terminal UI fallback |

---

## 1. Executive Summary
During the initial deployment of Proxmox Backup Server (PBS) onto `unit-02`, multiple concurrent failure modes emerged across three subsystems:
1. An Intel VT-d interrupt remapping hardware/firmware bug on the HP B120i storage controller caused the SATA bus to stall, triggering the Linux kernel to remount the target root SSD filesystem as read-only.
2. An iLO 4 virtual media network drop severed the underlying SquashFS live image, crashing the installer mid-extraction.
3. An X11 WebSockets scan code translation error inside the iLO 4 HTML5 console corrupted keyboard input, aborting the graphical installer.
All issues were mitigated by pivoting to a physical MBR USB installer, forcing Linux kernel parameters to bypass interrupt remapping, and using the ncurses-based Terminal installer.

---

## 2. Observable Symptoms & Log Evidence
* **IOMMU Bus Crash**:
  ```text
  [ 2409.475422] DMAR: DRHD: handling fault status reg 2
  [ 2409.475825] DMAR: [INTR-REMAP] Request device [01:00.0] fault index 0x18 
                 [fault reason 0x26] Blocked an interrupt request due to source-id verification failure
Filesystem State Lockout:
    ```text

    dpkg: error: unable to create new file '/var/lib/dpkg/status-new': Read-only file system
    dpkg-deb (subprocess): decompressing archive ... failed to write (Broken pipe)

    Virtual Media Severance:
    Plaintext

    I/O error, dev loop1, sector 1107354 op 0x0: (READ)
    SQUASHFS error: Failed to read block 0x21cb349f: -5
    /sbin/unconfigured.sh: line 385: /sbin/killall5: Input/output error

    Console Keycode Desync:

        Pressing E tripped an Xorg/KMS display server child crash (kmschild).

        Pressing F switched virtual terminals into unrendered video framebuffers (gray screen).

---

## 3. Root Cause Analysis (RCA)

A. The HP Gen8 DMAR Interrupt-Remapping Mismatch

The onboard HP Smart Array B120i controller (PCI [01:00.0]), when running in AHCI mode on legacy Gen8 BIOS microcode, presents an invalid PCI Source ID during high-throughput I/O interrupts. Modern Linux kernels (Debian 12 / Proxmox 6.x/7.x kernels) enforce strict Intel VT-d interrupt remapping. When the kernel began decompressing .deb packages onto /dev/sda, the CPU detected a source-id verification failure and dropped the interrupt request. The SATA driver timed out, the drive dropped off the bus, and the kernel locked /target into read-only mode to prevent catastrophic inode corruption.

B. Live Media Disconnect

Streaming a ~1GB ISO over the iLO 4 HTML5 Virtual Media channel is fragile. When the storage controller bus hung from the DMAR fault, the shared PCI auxiliary bus reset, severing the active WebSockets connection. The loopback mount (`/dev/loop1`) disappeared mid-read, causing `killall5` and subsequent binaries to fail with fatal I/O errors.

C. Client-Side Input Desync (HTML5 Console to X11)

The lightweight Xorg display manager launched by the graphical Proxmox installer expects standard X11 keyboard scan codes. The browser-based JavaScript iLO 4 HTML5 console misinterprets specific keycodes (such as `e` and `f`) as system control shortcuts, passing terminal-switch signals that kill the graphical child process.

## 4. Remediation Steps Taken

Replaced Virtual Media with Physical Media: Flashed the PBS ISO to a USB drive using Ventoy in MBR partition mode and booted directly from a rear USB 2.0 port.

Applied Kernel Parameter Workarounds:
    Injected `intremap=off intel_iommu=off` into the bootloader to disable strict VT-d interrupt remapping on PCI [01:00.0]. Baked this permanently into `/etc/default/grub`.

Bypassed X11 via Terminal UI:
    Selected Install Proxmox Backup Server (Terminal UI) in GRUB, running the installer inside a pure ncurses text environment impervious to browser WebSockets scan code corruption.

Scorched-Earth Block Wipe:
    Cleaned legacy Synology DSM RAID partition signatures off both 4TB WD Reds using wipefs and sgdisk prior to initializing the ZFS mirror pool.

## 5. Preventative Action Items

    [x] Firmware & Bootloader: Ensure intremap=off intel_iommu=off is stamped into /etc/default/grub on all HP Gen8 MicroServer bare-metal Linux installations.

    [x] Installation Policy: Mandate physical USB installation over rear USB 2.0 and the Terminal UI for all headless server deployments over legacy BMCs.

    [x] Documentation: Archive runbook under `docs/backup/pbs-deployment.md` and commit this post-mortem.
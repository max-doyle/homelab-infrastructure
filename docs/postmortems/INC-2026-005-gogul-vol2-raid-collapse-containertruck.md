# Incident Post-Mortem: Storage Pool 2 Multi-Disk Collapse & iSCSI Flash Tier Reconstruction

| Metadata Field | Value |
| :--- | :--- |
| **Incident ID** | `INC-2026-005` |
| **Target / Impacted System** | Synology NAS (`GOGUL` Volume 2), Proxmox VE (`unit-01`), `CT 101`, `CT 102` |
| **Severity Level** | P1 (Critical Workload Outage / Storage Pool Crash) |
| **Date & Time (UTC/Local)** | 2026-11-10 13:00 EST / 2026-11-17 21:52 EST (Final Cutover) |
| **Resolution Status** | Resolved / Hardware Upgraded & Workloads Restored |
| **Author / Lead Operator** | Doyle / Lead Infrastructure Operator |

---

## 1. Executive Summary
A cascading double-drive failure across a 3-disk 250GB SSD array (`Storage Pool 2 / Volume 2`) on `GOGUL` caused `mdadm` parity loss, taking the `containertruck` iSCSI LUN offline and terminating storage I/O for `CT 101` (`nerv-dns-core`) and `CT 102` (`nerv-proxy-core`). By auditing the hypervisor block tree prior to rebooting, operators extracted a live `vzdump` backup of `CT 102` from Linux RAM pagecache before uncached sectors failed. All legacy 250GB SSDs were decommissioned, `Storage Pool 2` was rebuilt as a matched 2 × 500GB SSD RAID-1 mirror, `CT 102` was rehydrated intact from backup, and `CT 101` was deterministically reprovisioned and re-authorized for inter-VLAN DNS resolution.

---

## 2. Telemetry & Observable Symptoms
* **Data Plane / Workloads**:
  * `CT 101` (`10.0.50.10`) and `CT 102` (`10.0.50.11`) remained temporarily responsive over HTTP/TCP due to resident binaries and inodes cached in `unit-01` host RAM, despite the underlying block device (`/dev/sdc`) reporting `SIZE: 0B`.
  * Live `vzdump` extraction of `CT 101` aborted with cascading `rsync: [sender] read errors ... Input/output error (5)` on cold disk blocks (`/var/lib/unbound/root.key`, `/usr/bin/flock`).
  * Live `vzdump` extraction of `CT 102` succeeded (`2.93 GB` written), salvaging Nginx Proxy Manager databases and TLS certificates.
* **Control / Management Plane**:
  * Synology DSM reported `Storage Pool 2 / Volume 2` crashed following the second drive drop in a 1-disk fault-tolerant SHR array.
  * Subsequent to rebuilding `CT 101`, administrative workstation (`10.0.20.10`) experienced external DNS resolution timeouts while internal records resolved; `pihole-FTL` logs revealed `ignoring query from non-local network 10.0.20.10`.
* **Storage & Block Telemetry**:
  * `lsblk` on `unit-01` revealed an unused, unmounted legacy boot drive (`/dev/sda`, 500GB SSD containing volume group `pve--OLD--EB34261E`) alongside the active Proxmox boot drive (`/dev/sdb`)[cite: 3].

---

## 3. Incident Timeline
| Timestamp (Local) | Event / Action Taken | Observed Response / State |
| :--- | :--- | :--- |
| **T0 (Day 1)** | Second 250GB SSD failed in `GOGUL` Storage Pool 2 (following prior loss of Drive 6)[cite: 3]. | `mdadm` lost parity margin; Volume 2 crashed; `containertruck` iSCSI LUN halted I/O[cite: 3]. |
| **T+10m** | Audited block device state on `unit-01` via `lsblk`; halted reboot to preserve RAM pagecache[cite: 3]. | Identified `/dev/sdc` at `0B` and discovered idle internal 500GB SSD (`/dev/sda`)[cite: 3]. |
| **T+15m** | Executed emergency `vzdump 101 102 --mode snapshot` (auto-fallback to `suspend`) to local disk[cite: 3]. | `CT 101` failed with `Input/output error (5)`; `CT 102` archived cleanly (`2.93 GB` tarball)[cite: 3]. |
| **T+25m** | Deactivated `containertruck-vg`, logged out iSCSI target, and powered down `unit-01` cleanly[cite: 3]. | Avoided `systemd` 90-second D-state unmount deadlock; extracted `/dev/sda` from chassis[cite: 3]. |
| **T+40m** | Purged crashed Volume 2 in DSM; ejected surviving 250GB drive from Bay 7; installed matched 500GB SSDs in Bays 5 & 6[cite: 3]. | Provisioned ~465 GB Btrfs RAID-1 mirror (`Storage Pool 2`) and 200 GB `containertruck` LUN[cite: 3]. |
| **T+55m** | Rescanned iSCSI bus on `unit-01`, initialized `containertruck-vg`, and rehydrated `CT 102` via `pct create`[cite: 3]. | `CT 102` (`nerv-proxy-core`) booted cleanly with all proxy hosts and certificates intact[cite: 3]. |
| **T+75m** | Purged `CT 101` orphan metadata; reprovisioned Debian 12 LXC, Unbound (`127.0.0.1#5335`), and Pi-hole (`10.0.50.10`)[cite: 3]. | Cleared duplicate DNSSEC trust anchor error in `/var/lib/unbound/root.key`; confirmed `ad` flag validation[cite: 3]. |
| **T+90m (Cutover)** | Reset workstation DNS client settings to dynamic DHCP (`10.0.50.10`); diagnosed inter-VLAN query drops in Pi-hole[cite: 3]. | Toggled Pi-hole interface behavior to **Permit all origins**; full routed DNS and ad-sinkhole verified[cite: 3]. |

---

## 4. Root Cause Analysis (RCA)
* **Trigger**: Simultaneous/sequential NAND exhaustion across matched-batch legacy 250GB SSDs operating in a 3-disk single-parity (`N-1`) SHR array, compounded by elevated rebuild/read I/O after the first drive failure[cite: 3].
* **Failure Mechanism**:
  1. **Storage Layer**: Losing two drives in a 3-disk SHR/RAID-5 array exceeds parity tolerance, rendering the underlying block device unreadable (`0B` exposed over iSCSI)[cite: 3].
  2. **Restore Rollback**: Standard `pct restore --force 1` failed on raw LVM block storage due to a device-mapper activation race condition, triggering an automatic volume rollback; bypassing metadata restore via `pct create 102 <tarball> --rootfs containertruck:6` cleanly extracted the userspace payload[cite: 3].
  3. **Post-Rebuild DNS Drop**: Fresh Pi-hole deployments default to single-subnet listening (`Allow only local requests`), silently dropping routed DNS queries originating from `TRUSTED` (`10.0.20.0/24`) crossing `HASEGAWA` into `SERVICES` (`10.0.50.0/24`)[cite: 3].
* **Detection Gap**: Legacy 250GB SSDs lacked proactive wear-leveling alerts prior to batch failure, and `CT 101` had not yet completed an automated backup cycle to `unit-02` (`PBS`) prior to the second disk drop[cite: 3].

---

## 5. Remediation & Recovery Steps
1. **Clean Initiator Teardown (`unit-01`)**:
   ```bash
   vgchange -an containertruck-vg
   pvesm set containertruck --disable 1
   iscsiadm -m node --logoutall=all
   shutdown -h now
    ```
2. **Hardware & Array Reconstruction (`GOGUL`)**:
* Decommissioned all legacy 250GB SSDs across Bays 5–7.
* Installed matched 500GB SSDs in Bays 5 and 6; built RAID-1 Storage Pool 2 and thick-provisioned 200 GB `containertruck` LUN.

1. Hypervisor Volume Re-binding & Guest Recovery (`unit-01`):
    ```Bash
    iscsiadm -m session --rescan
    pvcreate /dev/disk/by-id/<persistent-scsi-id>
    vgcreate containertruck-vg /dev/disk/by-id/<persistent-scsi-id>
    pct create 102 /var/lib/vz/dump/vzdump-lxc-102-*.tar \
      --rootfs containertruck:6 --hostname nerv-proxy-core --cores 2 --memory 1024 \
      --net0 name=eth0,bridge=vmbr0,tag=50,ip=10.0.50.11/24,gw=10.0.50.1 \
      --unprivileged 1 --features nesting=1 --onboot 1
    ```
2. Split-Horizon DNS & Multi-VLAN Cutover (`CT 101`):
* Rebuilt `CT 101` (`nerv-dns-core`), purged duplicate root trust anchors via `rm -f /var/lib/unbound/root.key && unbound-anchor -a "/var/lib/unbound/root.key" -v`, and verified DNSSEC (`ad` flag) on `127.0.0.1#5335`.
* Configured Pi-hole interface settings to Permit all origins to accept inter-VLAN requests from `10.0.20.0/24`.

## 6. Preventative & Architectural Action Items
- [x] Short-Term (Hardware): Replaced aged 250GB flash array with matched 500GB SSDs in a zero-parity-overhead RAID-1 mirror.
- [x] Short-Term (Network/DNS): Enabled Permit all origins on CT 101 so routed VLANs (10, 20, 30, 50) resolve cleanly through Pi-hole + Unbound.
- [x] Medium-Term (Disaster Recovery): Enrolled CT 101 and CT 102 in the nightly encrypted vzdump backup schedule targeting unit-02-pbs (10.0.10.12).
- [x] Medium-Term (Hardware Ledger): Updated README.md to reflect GOGUL Volume 2's upgraded 2 × 500GB SSD RAID-1 configuration.
- [ ] Long-Term (High Availability): Configure HASEGAWA (10.0.20.1) or a secondary container on unit-02 as a secondary failover DNS resolver in DHCP Option 6.
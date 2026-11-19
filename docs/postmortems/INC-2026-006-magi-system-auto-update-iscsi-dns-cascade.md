# Incident Post-Mortem: Unattended Core Switch Firmware Reboot & Cascading iSCSI / DNS Outage

| Metadata Field | Value |
| :--- | :--- |
| **Incident ID** | `INC-2026-005` |
| **Target / Impacted System** | UniFi US-8-150W (`MAGI-SYSTEM`), Synology NAS (`GOGUL`), Proxmox VE (`unit-01`), `CT 101` (`nerv-dns-core`), `unit-02-pbs` |
| **Severity Level** | P2 (Degraded / External Name Resolution Outage & Storage Remount Lockout) |
| **Date & Time (UTC/Local)** | 2026-11-18 03:03 EST / 08:03 UTC |
| **Resolution Status** | Resolved / Root Cause Eliminated & Stack Hardened |
| **Author / Lead Operator** | Doyle / Lead Infrastructure Operator |

---

## 1. Executive Summary
At 03:03 EST, an unattended automatic firmware update rebooted the core Layer-2 switch (`MAGI-SYSTEM`), severing the inter-VLAN storage link (`VLAN 30`) between hypervisor `unit-01` and NAS `GOGUL` for 3 minutes and 19 seconds. The resulting iSCSI session timeout exceeded `open-iscsi`'s 120-second recovery window, forcing `ext4` on `CT 101` (`nerv-dns-core`) into a read-only (`remount-ro`) state that crashed Unbound DNSSEC recursion and severed external name resolution across the network. Operators recovered the iSCSI volume via LVM/fsck, permanently patched Unbound's systemd startup wrapper, disabled automatic switch firmware updates, increased iSCSI timeout tolerances, and resolved a secondary PBS backup pruning permission fault discovered during log triage.

---

## 2. Telemetry & Observable Symptoms
* **Data Plane / Workloads**:
  * Administrative workstation (`10.0.20.10`) retained full intranet reachability (`10.0.10.0/24`, `10.0.50.0/24`), while external web traffic failed due to recursive DNS timeouts on `10.0.50.10:53`.
  * `pihole-FTL` inside `CT 101` continued serving local `.nerv.geofront` records from RAM, while `unbound.service` (`127.0.0.1#5335`) exited with `status=1/FAILURE` due to a duplicate trust anchor in `/var/lib/unbound/root.key` combined with a read-only root filesystem (`EROFS`).
* **Control / Management Plane**:
  * `GOGUL` DSM logged a LAN link drop at ~03:03 EST; `MAGI-SYSTEM` firmware version confirmed an unscheduled overnight auto-update reboot[cite: 9].
  * Nightly `vzdump` backup task (`03:00:07 EST`) uploaded incremental snapshots for `CT 100`, `CT 101`, and `CT 102` cleanly to `unit-02-pbs`, but exited with `TASK ERROR: job errors` during retention pruning (`missing Datastore.Modify|Datastore.Prune on /datastore/backup-pool`)[cite: 9].
* **Storage & Block Telemetry (`unit-01`)**:
  * `iscsid` logged `Kernel reported iSCSI connection 1:0 error (1022 - ISCSI_ERR_NOP_TIMEDOUT)` at `03:03:25`, followed by 29 `connect to 10.0.30.2:3260 failed (No route to host)` retries until recovering at `03:06:44` (199 seconds elapsed)[cite: 9].
  * Kernel ring buffer logged `Buffer I/O error on dev dm-7, logical block 8999` at `03:06:02` and `EXT4-fs (dm-7): Remounting filesystem read-only` at `03:07:16`[cite: 9].

---

## 3. Incident Timeline
| Timestamp (EST) | Event / Action Taken | Observed Response / State |
| :--- | :--- | :--- |
| **T-3m (03:00)** | Nightly `vzdump` backup job ran on `unit-01` targeting `unit-02-pbs`[cite: 9]. | All guest backups uploaded; prune phase failed due to RBAC role scope (`DatastoreBackup`)[cite: 9]. |
| **T0 (03:03)** | `MAGI-SYSTEM` initiated an automatic firmware update reboot[cite: 9]. | L2 switching fabric offline; `GOGUL` LAN dropped; `unit-01` iSCSI NOP timed out (`03:03:25`)[cite: 9]. |
| **T+2m (03:05)** | `open-iscsi` exceeded its 120s `replacement_timeout` threshold while switch was still rebooting[cite: 9]. | Kernel injected SCSI write I/O errors into `dm-7` (`CT 101` rootfs)[cite: 9]. |
| **T+4m (03:07)** | Switch finished booting (`03:06:44`), but `ext4` aborted `dm-7`'s journal (`03:07:16`)[cite: 9]. | `CT 101` remounted read-only; `unbound.service` crashed, halting external DNS resolution[cite: 9]. |
| **T+5h (08:00)** | Triage initiated: isolated Layer-3 WAN egress (`ping 1.1.1.1`) vs. DNS (`10.0.50.10`)[cite: 9]. | Identified `unbound.service` failure and `Read-only file system` lock inside `CT 101`[cite: 9]. |
| **T+5h 25m** | Stopped `CT 101` on `unit-01`, activated LVM slice (`lvchange -ay`), and ran `e2fsck -fy`[cite: 9]. | Dirty `ext4` journal replayed cleanly; `CT 101` remounted read-write[cite: 9]. |
| **T+5h 45m** | Purged duplicate `root.key`, created systemd drop-in override for `unbound-helper`, and restarted Unbound[cite: 9]. | `unbound.service` active; DNSSEC (`ad` flag) and external web access restored[cite: 9]. |
| **T+6h** | Audited `GOGUL` and UniFi logs; disabled switch auto-updates, raised iSCSI timeout to `300s`, and updated PBS ACLs[cite: 9]. | Root cause eliminated; storage fault tolerance and backup pruning hardened[cite: 9]. |

---

## 4. Root Cause Analysis (RCA)
* **Trigger**: Automatic device firmware updates were enabled in the UniFi Network Application, causing core switch `MAGI-SYSTEM` to reboot unattended at 03:03 EST across live production and storage VLANs[cite: 9].
* **Failure Mechanism**:
  1. **Storage Timeout Race Condition**: `MAGI-SYSTEM`'s firmware reboot and Spanning Tree transition took ~199 seconds, exceeding Debian's default `open-iscsi` `node.session.timeo.replacement_timeout` of `120` seconds[cite: 9]. Rather than continuing to queue writes in RAM, the SCSI layer surfaced I/O errors to `ext4` on `dm-7`, triggering `errors=remount-ro`[cite: 9].
  2. **Unbound Startup Fragility**: On restart, Debian's `/usr/libexec/unbound-helper root_trust_anchor_update` appended a duplicate root trust anchor line into `/var/lib/unbound/root.key`, causing Unbound's DNSSEC validator module to exit with `status=1/FAILURE` on a read-only filesystem[cite: 9].
  3. **Secondary Backup RBAC Mismatch**: `pve-backup@pbs` was scoped to `DatastoreBackup` (append-only), which permitted snapshot uploads but lacked the `Datastore.Modify|Datastore.Prune` bitflags included in `DatastorePowerUser`[cite: 9].
* **Detection Gap**: Core infrastructure lacked change-window enforcement on firmware updates, and `open-iscsi` timeouts were not tuned to exceed the boot cycle duration of the upstream storage switch/NAS[cite: 9].

---

## 5. Remediation & Recovery Steps
1. **Filesystem & Block Slice Recovery (`unit-01`)**:
   ```bash
   pct stop 101
   lvchange -ay containertruck-vg/vm-101-disk-0
   e2fsck -fy /dev/containertruck-vg/vm-101-disk-0
   pct start 101

2. **Permanent Unbound Trust-Anchor Hardening** (`CT 101`):
    ```bash
    rm -f /var/lib/unbound/root.key /var/lib/unbound/var/lib/unbound/root.key
    unbound-anchor -a "/var/lib/unbound/root.key" -v
    chown -R unbound:unbound /var/lib/unbound/ && chmod 644 /var/lib/unbound/root.key

    mkdir -p /etc/systemd/system/unbound.service.d
    cat << 'EOF' > /etc/systemd/system/unbound.service.d/override.conf
    [Service]
    ExecStartPre=
    ExecStartPre=/usr/libexec/unbound-helper chroot_setup
    EOF

    systemctl daemon-reload
    systemctl reset-failed unbound.service
    systemctl start unbound

    iSCSI Fault-Tolerance & PBS RBAC Tuning (unit-01 & unit-02):
    Bash

    # On unit-01: Increase iSCSI replacement timeout from 120s to 300s
    sed -i 's/^node.session.timeo.replacement_timeout = .*/node.session.timeo.replacement_timeout = 300/' /etc/iscsi/iscsid.conf
    iscsiadm -m node -T <target-iqn> -p 10.0.30.2:3260 -o update -n node.session.timeo.replacement_timeout -v 300
    systemctl restart iscsid

    # On unit-02: Grant DatastorePowerUser role for automated GFS retention pruning
    proxmox-backup-manager acl update /datastore/backup-pool DatastorePowerUser --auth-id pve-backup@pbs
    ```

6. Preventative & Architectural Action Items

    [x] Short-Term (Change Control): Disabled Automatic Device Firmware Updates in UniFi (https://10.0.10.24:8443) so MAGI-SYSTEM only reboots during planned maintenance windows[cite: 9].

    [x] Short-Term (Storage Resilience): Raised open-iscsi replacement_timeout to 300s on unit-01 to absorb transient switch/NAS reboots in memory without triggering ext4 remount-ro[cite: 9].

    [x] Short-Term (DNS & Backup Hygiene): Locked in systemd override.conf for unbound.service on CT 101 and upgraded pve-backup@pbs to DatastorePowerUser on unit-02[cite: 9].

    [x] Medium-Term (Documentation / Git): Committed INC-2026-005 to homelab-infrastructure/docs/postmortems/[cite: 9].

    [ ] Long-Term (High Availability): Configure HASEGAWA (10.0.20.1) or a secondary container on unit-02 as a secondary failover resolver in DHCP Option 6[cite: 9].
    '@ | Out-File -FilePath "docs\postmortems\005-magi-system-auto-update-iscsi-dns-cascade.md" -Encoding utf8
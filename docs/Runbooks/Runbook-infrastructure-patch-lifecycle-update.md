# Runbook: Infrastructure Patch & Update Lifecycle

- **Scope**: `nerv.geofront` Production Stack (LXC Workloads, Hypervisors, Storage Arrays, Core Network Fabric, Out-of-Band BMCs)
- **Objective**: Enforce deterministic, tiered update execution with zero unscheduled downtime, verified rollback points, and strict storage quiescing.
- **Prerequisites**:
  - `Runbook-containertruck-maintenance-lifecycle-control.md` (Required before rebooting `GOGUL` or `MAGI-SYSTEM`)
  - Administrative SSH/WebGUI access from `Hulle-Granz-Cathedral` (`10.0.20.10`)

---

## 1. Architectural Doctrine & Patching Order

Updates across `nerv.geofront` execute strictly from **highest-level workloads down to core transport fabric** to minimize blast radius and prevent storage corruption:

1. **Tier 1 — Guest Workloads (LXC / VM)**: Automated nightly OS security patches; monthly manual application updates.
2. **Tier 2 — Backup & Compute Nodes (`unit-02` $\to$ `unit-01`)**: Staggered by 48–72 hours so a kernel or driver regression never compromises compute and backup planes simultaneously.
3. **Tier 3 — Storage Fabric (`GOGUL`, `TentacleShop`)**: Quarterly or critical security advisories; requires prior iSCSI teardown.
4. **Tier 4 — Core Network & Perimeter (`MAGI-SYSTEM`, `HASEGAWA`)**: Strictly manual; automatic device firmware updates disabled.
5. **Tier 5 — Out-of-Band BMC (`unit-02-ilo`)**: Semi-annual or critical CVE; cryptographic SHA-256 verification mandatory before flashing.

---

## 2. Phase 1: Pre-Flight Audit & Backup Verification

Before applying manual updates to hypervisors, storage, or network hardware:

1. **Verify Backup Integrity (`unit-02-pbs` — `10.0.10.12:8007`)**:
   - Confirm the prior night's `vzdump` backup task for `CT 100`, `CT 101`, and `CT 102` completed with `TASK OK`.
2. **Export Network Configuration Snapshots**:
   - **`HASEGAWA` (`10.0.10.1`)**: Navigate to **Diagnostics $\to$ Backup & Restore** and download the encrypted `config.xml`.
   - **`CT 100` UniFi Controller (`10.0.10.24:8443`)**: Navigate to **Settings $\to$ System $\to$ Backups** and download a fresh `.unf` backup archive.
3. **Check Pending Advisories**:
   - **Proxmox Nodes (`unit-01`, `unit-02`)**: Inspect **Node $\to$ Updates** in the WebGUI or run `apt update && apt list --upgradable`.
   - **Synology `GOGUL` (`10.0.30.2:5001`)**: Check **Control Panel $\to$ Update & Restore** (Auto-download enabled, auto-install disabled).
   - **pfSense `HASEGAWA`**: Check dashboard version banner against Netgate release notes.

---

## 3. Phase 2: Tiered Execution Procedures

### Tier 1: Guest LXC Containers (`CT 100`, `CT 101`, `CT 102`)
- **Automated Security Hygiene**: `unattended-upgrades` runs daily at `04:00` pulling exclusively from `bookworm-security`.
- **Manual Application Updates (Monthly)**:
  ```bash
  # Enter target container from unit-01
  pct enter <CTID>
  apt update && apt upgrade -y
  ```
- **Post-Update Check**:
  - `CT 101`: Verify Unbound DNSSEC resolution (`dig @127.0.0.1 -p 5335 cloudflare.com +dnssec` returns `ad` flag).
  - `CT 102`: Confirm `https://proxmox.nerv.geofront` and wildcard TLS endpoints respond with HTTP 200/302.

### Tier 2: Backup & Hypervisor Nodes (`unit-02` First, `unit-01` 48h Later)
*Never use plain `apt upgrade` on Proxmox systems; always use `apt dist-upgrade` so kernel and ZFS/QEMU dependency shifts resolve cleanly.*

```bash
# 1. Refresh indexes and apply full distribution upgrade
apt update && apt dist-upgrade -y

# 2. Check if a new kernel was staged requiring a reboot
uname -r
ls -lh /boot/vmlinuz-*
```

- **If Reboot Required on `unit-02` (`10.0.10.12`)**:
  - Verify no active backup, verify, or garbage collection jobs are running on `backup-pool`.
  - Execute `reboot` and verify `zpool status backup-pool` returns `ONLINE` with zero errors.
- **If Reboot Required on `unit-01` (`10.0.10.10`)**:
  - Follow `Runbook-containertruck-maintenance-lifecycle-control.md` Part 1 to cleanly stop `CT 101`/`CT 102` and log out of iSCSI before rebooting `unit-01`.
  - After reboot, execute Part 2 to re-mount `containertruck` and start guests, then verify `pveversion -v` and `uname -r`.

### Tier 3 & Tier 4: Storage (`GOGUL`) & Core Switch (`MAGI-SYSTEM`)
*Mandatory Storage Quiesce: Never reboot `GOGUL` or `MAGI-SYSTEM` while `unit-01` holds active iSCSI sessions.*

1. Execute **Part 1 (Graceful Teardown)** of `Runbook-containertruck-maintenance-lifecycle-control.md` on `unit-01`. Confirm `iscsiadm -m session` reports `No active sessions`.
2. Apply DSM update on `GOGUL` (`10.0.30.2:5001`) or trigger manual firmware update on `MAGI-SYSTEM` (`10.0.10.24:8443`).
3. Wait for target device to return online and pass ICMP/port checks (`nc -zvw3 10.0.30.2 3260`).
4. Execute **Part 2 (Cold Re-Activation)** of `Runbook-containertruck-maintenance-lifecycle-control.md` on `unit-01` to re-login iSCSI, activate `containertruck-vg`, and start `CT 101` and `CT 102`.

### Tier 5: Out-of-Band BMC (`unit-02-ilo` — `10.0.10.11`)
1. Download the official HPE iLO 4 firmware `.bin` package on `Hulle-Granz-Cathedral`.
2. Verify cryptographic integrity before upload:
   ```powershell
   Get-FileHash .\ilo4_282.bin -Algorithm SHA256
   ```
3. Upload via iLO 4 WebGUI (**Administration $\to$ Firmware**). Note: iLO 4 resets its management processor independently without powering off `unit-02`.

---

## 4. Post-Maintenance Verification Checklist
- [ ] `pvesm status` on `unit-01` shows `local`, `local-lvm`, `containertruck`, and `unit-02-pbs` as `active`.
- [ ] `pct list` confirms `CT 100`, `CT 101`, and `CT 102` are `running`.
- [ ] `iscsiadm -m session -P 1` confirms active session to `10.0.30.2:3260` with `replacement_timeout = 300`.
- [ ] Internal DNS (`pi-hole.nerv.geofront`) and TLS reverse proxy (`nginx.nerv.geofront`) resolve and load cleanly from `10.0.20.10`.

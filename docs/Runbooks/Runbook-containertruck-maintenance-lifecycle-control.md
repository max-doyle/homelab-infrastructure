# Runbook: containertruck Maintenance & Lifecycle Control
## Part 1: Graceful Teardown (Turning It OFF for Maintenance)

Execute this sequence on unit-01 (SSH or Web Shell as root) before taking GOGUL down or performing maintenance:
# 1. Stop all containers/VMs residing on containertruck
```bash
# (Check running guests with 'pct list' or 'qm list')
pct stop 101 2>/dev/null || true

# 2. Deactivate the LVM Volume Group
# This flushes in-memory transaction logs and revokes active logical volume mappings
vgchange -an containertruck-vg

# 3. Disable Proxmox storage checkers so pvestatd doesn't poll a dead socket
pvesm set containertruck --disable 1
pvesm set gogul-iscsi --disable 1

# 4. Cleanly log out of the iSCSI session to GOGUL
iscsiadm -m node -T iqn.2000-01.com.synology:GOGUL.containertruck -u
# (Or cleanly log out of all active target sessions):
# iscsiadm -m node --logoutall=all

# 5. Verify the block session is dead
iscsiadm -m session
```

**Verification:** iscsiadm -m session must report iscsiadm: No active sessions. Now GOGUL can be cleanly rebooted, serviced, or powered down without hanging unit-01.

## Part 2: Cold Re-Activation (Turning It Back ON)

Execute this sequence on unit-01 right now to resurrect containertruck:
### Step 1: Verify the Target & Layer 4 Pinhole

First, prove GOGUL's SAN daemon is listening on VLAN 30 and HASEGAWA isn't dropping the packets:
```bash
nc -zvw3 10.0.30.2 3260
```
**Verification:** Must return open or succeeded.
### Step 2: Establish the iSCSI Session

Re-discover and log into the containertruck target on GOGUL:
```bash
# Discover targets on GOGUL
iscsiadm -m discovery -t st -p 10.0.30.2

# Log into the target session
iscsiadm -m node -p 10.0.30.2 --login

# Verify the kernel mapped the session
iscsiadm -m session -P 1
```
**Verification:** iscsiadm -m session will output an active session to 10.0.30.2:3260.

### Step 3: Rescan the SCSI Bus & Activate the LVM Pool

Ensure the Linux kernel sees the raw disk node and wakes up the volume group:
```bash
# Rescan the SCSI host bus
rescan-scsi-bus.sh 2>/dev/null || udevadm trigger

# Re-scan for the Volume Group and activate it
vgscan
vgchange -ay containertruck-vg

# Verify the block device and VG state
lvs
vgs
```
**Verification:** vgs will display containertruck-vg with ~200GB capacity and active attributes

### Step 4: Re-Enable Storage in Proxmox

Flip the storage pools back to active inside Proxmox's storage engine and kick pvestatd:
```bash
# Enable the iSCSI target and LVM backend in Proxmox
pvesm set gogul-iscsi --disable 0
pvesm set containertruck --disable 0

# Restart the status polling daemon to force a sync
pvestatd restart
pvesm status
```


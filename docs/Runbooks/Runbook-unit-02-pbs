# Runbook: Commissioning unit-02 as Proxmox Backup Server (PBS)

- **Target Node**: `unit-02` (HP ProLiant MicroServer Gen8)
- **Role**: Dedicated Bare-Metal Backup Appliance & ZFS Datastore Target
- **Management Interface**: `10.0.10.12/24` (VLAN 10 - MGMT) | Gateway: `10.0.10.1` | Port: 8007
- **Out-of-Band (iLO 4)**: `10.0.10.11` (Dedicated Port, VLAN 10)
- **Storage Substrate**:
  - Boot/OS: 1 × Samsung 840 120GB SATA SSD (Bay 1)
  - Backup Pool: 2 × Western Digital Red 4TB HDD (Bays 2 & 3, ZFS Mirror)

---

## 1. Hardware & Out-of-Band Staging
1. Verify onboard HP Dynamic Smart Array B120i is operating in **Legacy / SATA AHCI Mode** in BIOS.
2. Upgrade HP iLO 4 firmware to **v2.82** to enable the native HTML5 Remote Console.
3. Bind iLO 4 dedicated NIC to static DHCP mapping on `VLAN 10` (`10.0.10.11`).

## 2. Boot Media & Installation (Text-Only Mode)
1. Format a USB drive using **Ventoy in MBR Partition Style** (avoid GPT hybrid schemes on Gen8 BIOS).
2. Load the official Proxmox Backup Server ISO onto the data partition.
3. Insert into a **rear USB 2.0 port** (black ports) to avoid legacy xHCI USB 3.0 controller initialization drops.
4. Power on via iLO 4 HTML5 console, press **F11** at POST, and select the USB drive.
5. In the Proxmox bootloader menu:
   - Highlight **Install Proxmox Backup Server (Terminal UI)**.
   - Press `e` to edit the GRUB boot command line.
   - Append to the `linux` stanza: `intremap=off intel_iommu=off`
   - Press `Ctrl + X` (or `F10`) to boot.
6. Target the 120GB Samsung SSD (`/dev/sda`), assign hostname `unit-02.nerv.geofront`, IP `10.0.10.12/24`, Gateway `10.0.10.1`, DNS `10.0.10.1`.

## 3. Post-Install OS Hardening
1. **Permanent GRUB Mitigation**:
   ```bash
   # Add IOMMU parameter workaround to default kernel arguments
   sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="quiet intremap=off intel_iommu=off"/' /etc/default/grub
   update-grub
2. **Switch to No-Subscription Repository:**
    ```bash
    # Disable enterprise repo
    sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pbs-enterprise.list 2>/dev/null || true
    rm -f /etc/apt/sources.list.d/pbs-enterprise.sources 2>/dev/null || true

    # Add community no-subscription repo
    echo "deb [http://download.proxmox.com/debian/pbs](http://download.proxmox.com/debian/pbs) bookworm pbs-no-subscription" > /etc/apt/sources.list.d/pbs-no-subscription.list

    # Refresh and upgrade
    apt update && apt dist-upgrade -y
    ```
## 4. ZFS Datastore Provisioning

1. **Scorched-Earth Partition Wipe on 4TB Platters:**
    ```Bash
    wipefs -a /dev/sdb /dev/sdc
    sgdisk --zap-all /dev/sdb
    sgdisk --zap-all /dev/sdc
    partprobe /dev/sdb /dev/sdc
    ```
    

1. **Create Mirrored Zpool & Datastore:**
    ```Bash

    # Create redundant ZFS pool
    proxmox-backup-manager disk zpool create backup-pool --devices sdb,sdc --raidlevel mirror

    # Initialize PBS chunk metadata tree and datastore mapping
    proxmox-backup-manager datastore create backup-pool /mnt/datastore/backup-pool
    ```

## 5. Hypervisor Integration (`unit-01` -> `unit-02`)

1. On unit-02 (PBS):

    - Retrieve SHA-256 fingerprint from Dashboard -> Show Fingerprint.

    - Create user pve-backup@pbs under Access Control -> User Management.

    - Grant role Datastore.Backup on /datastore/`backup-pool`.

2. On unit-01 (PVE):

    - Navigate to Datacenter -> Storage -> Add -> Proxmox Backup Server.

    - Configure ID `unit-02-pbs`, Server `10.0.10.12`, User `pve-backup@pbs`, Datastore `backup-pool`.

    - Enter TLS Fingerprint and generate a client-side Encryption Key (AES-256-GCM).

    - Export `.enc` key file to offline cold vault and password manager.

3. Automated Maintenance Schedules:

    - `unit-01`: Scheduled backup daily at `02:00` (Snapshot mode, ZSTD).

    - `unit-02`: Garbage Collection weekly on Saturday `04:00`.

    - `unit-02`: Datastore Verify Job weekly on Sunday `02:00`.
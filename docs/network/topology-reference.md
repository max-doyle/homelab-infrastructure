**Physical Topology**
```text
+---------------------------------------+
                     |        Upstream AmpliFi Router        |
                     +-------------------+-------------------+
                                         |
                                         | Cat6 Patch
                                         v
                     +---------------------------------------+
                     |  Netgate SG-3100 Gateway [HASEGAWA]   |
                     |  WAN: Port 0 (mvneta0)                |
                     |  LAN 1: Port 1 (Marvell Switch Port 1)|
                     |  CPU Bus: Port 5 (mvneta1 2.5Gbps KX) |
                     +-------------------+-------------------+
                                         |
                                         | RJ-45 Trunk Uplink
                                         | (802.1Q Multiplexed)
                                         v
+---------------------------------------------------------------------------------+
|                       UniFi US-8-150W Switch [MAGI-SYSTEM]                     |
+----+--------+--------+--------+--------+--------+--------+----------------------+
| P1 |   P2   |   P3   |   P4   |   P5   |   P6   |   P7   |          P8          |
+----+--------+--------+--------+--------+--------+--------+----------------------+
  |      |        |        |        |        |        |               |
  |      |        |        |        |        |        |               +-- [Unused / Dark]
  |      |        |        |        |        |        |                   (PoE Disabled)
  |      |        |        |        |        |        |
  |      |        |        |        |        |        +-- Synology DS1815+ [GOGUL]
  |      |        |        |        |        |            LAN 1 (Storage Array)
  |      |        |        |        |        |
  |      |        |        |        |        +-- Daily Driver Station
  |      |        |        |        |            [Hulle-Granz-Cathedral]
  |      |        |        |        |
  |      |        |        |        +-- HP MicroServer Gen8 [Server B]
  |      |        |        |            LAN 1 (PBS Compute Node)
  |      |        |        |
  |      |        |        +-- HP MicroServer Gen8 [unit-02-ilo]
  |      |        |            Dedicated iLO 4 Port (BMC OOB)
  |      |        |
  |      |        +-- Mobile Admin Drop / Bench
  |      |            [AAA Wunder]
  |      |
  |      +-- Proxmox VE Compute Host [unit-01]
  |          ASRock Z97 Extreme6 -> Physical NIC 1 (nic1)
  |
  +-- Uplink Trunk from HASEGAWA (LAN 1)
```

**Logical Topology**
```text
  [WAN: Public / Double-NAT upstream]
                                                  |
                                                  v
                               +-------------------------------------+
                               |         Netgate SG-3100             |
                               |           [HASEGAWA]                |
                               |    Stateful Zero-Trust Firewall     |
                               +------------------+------------------+
                                                  |
                                                  | mvneta1 (Trunk: 10, 20, 30, 40, 50, 66)
                                                  | Switch Table: 1t, 5t
                                                  v
                               +-------------------------------------+
                               |        UniFi US-8-150W Switch       |
                               |            [MAGI-SYSTEM]            |
                               |      Layer-2 802.1Q Switching       |
                               +--+-------+--------+--------+-----+--+
                                  |       |        |        |     |
            +---------------------+       |        |        |     +--------------------+
            |                             |        |        |                          |
            v                             v        v        v                          v
+-------------------------+         +---------+ +----+ +---------+         +-----------------------+
|  VLAN 10: MGMT          |         | VLAN 20 | |VLAN| | VLAN 50 |         |  VLAN 30: STORAGE     |
|  10.0.10.0/24           |         | TRUSTED | | 66 | | SERVICES|         |  10.0.30.0/24         |
+-------------------------+         |10.0.20.0| |LAB | |10.0.50.0|         +-----------------------+
| • HASEGAWA  : 10.0.10.1 |         |   /24   | | /24| |   /24   |         | • HASEGAWA : 10.0.30.1|
| • MAGI-SYS  : 10.0.10.2 |         +----+----+ +--+-+ +----+----+         | • GOGUL    : 10.0.30.2|
| • unit-01   : 10.0.10.10|              |         |        |              +-----------------------+
| • iLO 4     : 10.0.10.11|              |         |        |                          ^
| • CT 100    : 10.0.10.24|              |         |        |                          |
+-------------------------+              |         |        |                          | iSCSI / NFS
            ^                            |         |        |                          | (TCP 3260)
            |                            v         |        |                          |
            |                 +---------------+    |        |                          |
            | Pinhole Rules:  |   Workstation |    |        |                          |
            | TCP 8006, 8443, |      [HGC]    |    |        |                          |
            | 443, 22         |  10.0.20.10   |    |        |                          |
            +-----------------+---------------+    |        |                          |
                                                   |        |                          |
                                                   v        v                          |
                        +----------------------------------------------------+         |
                        |              unit-01 Hypervisor Compute            |         |
                        |      Linux VLAN-Aware Bridge (vmbr0: VIDs 2-4094)  +---------+
                        |                                                    |
                        |  • vmbr0.10   -> Hypervisor Host OS (10.0.10.10)   |
                        |  • veth100i0  -> CT 100 UniFi Controller (Tag 10)  |
                        |  • [Future VM]-> SERVICES Engine (Tag 50)          |
                        |  • [Future VM]-> Kali Sandbox Node (Tag 66)        |
                        +----------------------------------------------------+
```
# Incident Post-Mortem: Layer-3 Loss of Switch Management Plane

| Incident ID | Target Component | Severity | Resolution Status |
| :--- | :--- | :--- | :--- |
| `INC-2026-001` | UniFi US-8-150W (`MAGI-SYSTEM`) | P2 (Loss of Management Plane) | Root Cause Identified / Mitigation Staged |

---

## 1. Summary
Following a network reconfiguration shifting the management plane to dedicated `VLAN 10 (MGMT)`, the core switch (`MAGI-SYSTEM`) stopped responding across all Layer-3 interfaces (HTTP/SSH) and failed to obtain a DHCP lease or bind a static ARP entry at the gateway. The hardware Layer-2 data plane continued forwarding tagged 802.1Q traffic unaffected.

## 2. Telemetry & Symptoms
* **Data Plane**: Normal. Tagged traffic across `VLAN 10` (Proxmox hypervisor to Netgate gateway) and `VLAN 20` (workstation endpoints) continued flowing through the switch without packet loss.
* **Control Plane**: Unreachable. Switch MAC address completely disappeared from gateway ARP tables across all interfaces (`mvneta1`, `mvneta1.10`, `mvneta1.20`).
* **Fallback State**: Switch failed to self-assign standard fallback IP `192.168.1.20` on legacy subnets.

## 3. Root Cause Analysis
During the trunk port profile shift, the switch's internal switching chip (Layer 2 ASIC) updated successfully to pass tagged trunks. However, the embedded Linux network management stack running on the switch failed to release its legacy DHCP binding and did not properly re-bind its internal management interface (`br0` / Layer 3 IP stack) to the designated PVID on the uplink trunk port[cite: 2]. Because Layer-3 management access dropped while the data-plane trunk remained intact, remote GUI/SSH intervention was locked out[cite: 2].

## 4. Remediation & Operational Mitigations
1. **Immediate Remediation**: Out-of-band console access via an FTDI USB-to-RJ45 serial rollover cable attached directly to the switch's RS-232 serial console (115200 8-N-1).
2. **Permanent Architecture Addition**: Design of an independent Out-of-Band (OOB) serial terminal server using `ser2net` running on a low-power management controller. This ensures RS-232/UART access to edge switches and the Netgate gateway is available over a private console network, preventing physical intervention during future trunking modifications.

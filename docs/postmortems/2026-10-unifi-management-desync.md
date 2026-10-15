Incident Summary:

    Service Impact: UniFi Network Application (CT 100) administrative authentication lockout and MAGI-SYSTEM (US-8-150W) management control plane loss during VLAN 10 migration.

    Data Plane Status: Zero packet loss across transit VLANs (VLAN 10 Proxmox trunking, VLAN 20 workstation access remained fully operational).

Root Cause Analysis (RCA):

    Headless Debian container lacked mongo/mongosh client binaries in upstream package manifests while mongod was actively listening on loopback port 27117.

    Interrupted Layer-3 provision caused UniFi daemon to drop native bridge binding prior to establishing gateway route verification on tagged management sub-interface.

Resolution & Recovery Steps:

    Injected python3-pymongo to query the ace.admin collection directly over loopback socket, extracting administrative username and associated identity.

    Restored control plane access without blowing away operational transit profiles.

Action Items & Preventative Controls:

    [ ] Standardize local credential backups in password vault alongside container environment declarations.

    [ ] Maintain python3-pymongo in baseline container template provisioning scripts.

    [ ] Require dedicated out-of-band serial or dual-homed management interfaces prior to core switch re-IP operations.

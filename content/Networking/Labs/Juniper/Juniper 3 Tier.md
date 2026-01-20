# Juniper 3-Tier Lab: Final, Detailed JNCIA-Junos Study Guide

  

Welcome to your 3-tier Juniper lab! This document provides a complete, step-by-step guide for configuring this network from the ground up. Each task is mapped to JNCIA-Junos exam objectives and includes specific goals, commands, and verification steps.

---

## Lab Topology & IP Plan

Before starting, familiarize yourself with the lab's layout and the IP addressing scheme we will implement.

- **Core:** `core1`, `core2` (vJunos-Evolved)

- **Distribution:** `dist1`, `dist2` (vJunos-Router)

- **Access:** `access1`, `access2` (vJunos-Switch)

- **Clients:** `client1`, `client2` (Linux)

  
| Link                            | Device      | Interface  | IP Address          |
| :------------------------------ | :---------- | :--------- | :------------------ |
| **Loopbacks**                   | `core1`     | `lo0.0`    | `1.1.1.1/32`        |
|                                 | `core2`     | `lo0.0`    | `2.2.2.2/32`        |
|                                 | `dist1`     | `lo0.0`    | `3.3.3.3/32`        |
|                                 | `dist2`     | `lo0.0`    | `4.4.4.4/32`        |
| **Core Interconnect**           | `core1`     | `ge-0/0/0` | `10.0.0.1/30`       |
|                                 | `core2`     | `ge-0/0/0` | `10.0.0.2/30`       |
| **Dist1 <> Core**               | `dist1`     | `ge-0/0/0` | `10.1.1.1/30`       |
|                                 | `core1`     | `ge-0/0/1` | `10.1.1.2/30`       |
|                                 | `dist1`     | `ge-0/0/1` | `10.1.2.1/30`       |
|                                 | `core2`     | `ge-0/0/1` | `10.1.2.2/30`       |
| **Dist2 <> Core**               | `dist2`     | `ge-0/0/0` | `10.2.1.1/30`       |
|                                 | `core1`     | `ge-0/0/2` | `10.2.1.2/30`       |
|                                 | `dist2`     | `ge-0/0/1` | `10.2.2.1/30`       |
|                                 | `core2`     | `ge-0/0/2` | `10.2.2.2/30`       |
| **VLAN 10 Gateway (Clients A)** | `dist1`     | `irb.10`   | `192.168.10.253/24` |
|                                 | `dist2`     | `irb.10`   | `192.168.10.252/24` |
|                                 | **VRRP IP** | `vrrp`     | `192.168.10.254/24` |
| **VLAN 20 Gateway (Clients B)** | `dist1`     | `irb.20`   | `192.168.20.253/24` |
|                                 | `dist2`     | `irb.20`   | `192.168.20.252/24` |
| **Clients**                     | `client1`   | `eth1`     | `192.168.10.1/24`   |
|                                 | `client2`   | `eth1`     | `192.168.20.1/24`   |

---

## Module 1: Initial Device Setup

  
**Goal:** Perform basic "day one" configuration on all Juniper devices to establish secure management access and unique identities.

### 1.1: Set Hostname and Root Password
  
- **Applies to:** All 6 Juniper devices (`core1`, `core2`, `dist1`, `dist2`, `access1`, `access2`)

- **Steps:**

```bash

# [On Each Device]

set system host-name <device-name>

set system root-authentication plain-text-password

# Enter and confirm your password

commit

```

- **Verification:** The CLI prompt will change to `root@<device-name>>`.

> [!NOTE]
> If you're using Containerlab, this is already done for you!
> 

  

### 1.2: Create a Non-Root Admin User

- **Goal:** Create a user for daily administration, following the best practice of not using the `root` account for routine tasks.

- **Applies to:** All 6 Juniper devices

- **Steps:**

```bash

# [On Each Device]

set system login user labadmin class super-user authentication plain-text-password

# Enter and confirm your password

commit

```

- **Verification:** Log out and log back in as the `labadmin` user. You should have full administrative rights.

  

### 1.3: Configure System Services using Groups

  

- **Goal:** Configure NTP and Syslog efficiently using a configuration group, then apply that group.

- **Applies to:** All 6 Juniper devices

- **Steps:**

```bash

# [On Each Device]

set groups global-config system syslog file messages any any

set groups global-config system name-server 8.8.8.8 routing-instance mgmt_junos

set groups global-config system name-server 8.8.4.4 routing-instance mgmt_junos

set groups global-config system ntp server time.google.com

set apply-groups global-config

commit

```

- **Verification:**

```junos

# [On Each Device]

show ntp status # Should show the server and a high stratum until it syncs

```

  

### 1.4: Configure Loopback Interfaces

  

- **Goal:** Configure stable loopback addresses for management and for use as a router-ID in routing protocols.

- **Applies to:** `core1`, `core2`, `dist1`, `dist2`

- **Steps:**

  

```junos

# [On core1]

set interfaces lo0 unit 0 family inet address 1.1.1.1/32

commit

  

# --- Repeat for other devices with their respective IPs from the plan ---

```

  

- **Verification:**

```junos

# [On core1]

show interfaces lo0.0 terse

# Expected output shows lo0.0 is up and has the correct IP

```

  

---

  

## Module 2: Layer 2 - Switching & VLANs

  

**Goal:** Configure the access and distribution layers to segment traffic into two separate broadcast domains using VLANs.

  

### 2.1: Create VLANs

  

- **Applies to:** `access1`, `access2`, `dist1`, `dist2`

- **Steps:**

```junos

# [On all four devices]

set vlans CLIENTS-A vlan-id 10

set vlans CLIENTS-B vlan-id 20

commit

```

- **Verification:**

```junos

# [On any of the four devices]

show vlans

# You should see CLIENTS-A and CLIENTS-B listed with the correct VLAN IDs

```

  

### 2.2: Configure Access Ports

  

- **Goal:** Assign client-facing ports on the access switches to their respective VLANs.

- **Applies to:** `access1`, `access2`

- **Steps:**

  

```junos

# [On access1]

set interfaces ge-0/0/10 unit 0 family ethernet-switching vlan members CLIENTS-A

commit

  

# [On access2]

set interfaces ge-0/0/10 unit 0 family ethernet-switching vlan members CLIENTS-B

commit

```

  

- **Verification:**

```junos

# [On access1]

show vlans CLIENTS-A

# The output should list ge-0/0/10.0 as a member of the VLAN

```

  

### 2.3: Configure Trunk Ports

  

- **Goal:** Configure the links between the access and distribution layers to carry traffic for all VLANs.

- **Applies to:** `access1`, `access2`, `dist1`, `dist2`

- **Steps:**

  

```junos

# [On access1]

set interfaces ge-0/0/0 unit 0 family ethernet-switching interface-mode trunk

set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members all

set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode trunk

set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members all

commit

  

# --- Repeat this pattern for the corresponding ports on access2, dist1, and dist2 ---

```

  

- **Verification:**

```junos

# [On access1]

show interfaces ge-0/0/0

# The output should show "VLAN-Tagging" is enabled

```

  

---

  

## Module 3: Layer 3 - Routing

  

**Goal:** Establish IP connectivity throughout the network, enabling communication between all subnets using OSPF.

  

### 3.1: Configure Point-to-Point Interfaces

  

- **Applies to:** `core1`, `core2`, `dist1`, `dist2`

- **Steps:**

  

```junos

# [On core1]

set interfaces ge-0/0/0 unit 0 family inet address 10.0.0.1/30

set interfaces ge-0/0/1 unit 0 family inet address 10.1.1.2/30

set interfaces ge-0/0/2 unit 0 family inet address 10.2.1.2/30

commit

  

# --- Repeat for all other core and distribution routers according to the IP Plan ---

```

  

- **Verification:**

```junos

# [On core1]

show interfaces terse | match "ge-"

# All configured interfaces should be up/up with the correct IP address

```

  

### 3.2: Configure Inter-VLAN Routing (Gateways)

  

- **Goal:** Create Layer 3 gateway interfaces on the distribution routers for each VLAN.

- **Applies to:** `dist1`, `dist2`

- **Steps (vJunos-Router uses bridge-domains):**

  

```junos

# [On dist1]

set interfaces irb unit 10 family inet address 192.168.10.253/24

set interfaces irb unit 20 family inet address 192.168.20.253/24

set vlans CLIENTS-A l3-interface irb.10

set vlans CLIENTS-B l3-interface irb.20

commit

  

# --- Repeat on dist2 with its respective IP addresses ---

```

  

- **Verification:**

```junos

# [On dist1]

show interfaces irb.10

# The interface should be up and have the correct IP address

```

  

### 3.3: Configure OSPF

  

- **Goal:** Enable OSPF on all routed interfaces to dynamically build the routing table.

- **Applies to:** `core1`, `core2`, `dist1`, `dist2`

- **Steps:**

```junos

# [On all four devices]

set protocols ospf area 0.0.0.0 interface all

set protocols ospf area 0.0.0.0 interface lo0.0 passive

commit

```

- **Verification:**

```junos

# [On core1]

show ospf neighbor

# You should see neighbors for dist1, dist2, and core2 in state "Full"

show route protocol ospf

# The routing table should be populated with routes from all other routers

```

  

---

  

## Module 4: Routing Policy

  

**Goal:** Manipulate the routing table by preventing a specific route from being advertised, demonstrating export policies.

  

### 4.1: Block an OSPF Advertisement

  

- **Applies to:** `dist1` (policy) and `core1` (verification)

- **Steps:**

```junos

# [On dist1]

set policy-options prefix-list REJECT-CLIENT-A 192.168.10.0/24

set policy-options policy-statement NO-ADV-CLIENT-A term 1 from prefix-list REJECT-CLIENT-A

set policy-options policy-statement NO-ADV-CLIENT-A term 1 then reject

set policy-options policy-statement NO-ADV-CLIENT-A term 2 then accept

set protocols ospf export NO-ADV-CLIENT-A

commit

```

- **Verification:**

```junos

# [On core1]

show route 192.168.10.0/24

# The route via dist1 (next-hop 10.1.1.1) should be gone. You may still see a route via dist2.

```

  

---

  

## Module 5: Firewall Filters

  

**Goal:** Secure the network by creating and applying a stateless firewall filter to block specific traffic.

  

### 5.1: Block and Count ICMP Traffic

  

- **Applies to:** `dist1`

- **Steps:**

```junos

# [On dist1]

set firewall family inet filter BLOCK-PING term 1 from source-address 192.168.10.0/24

set firewall family inet filter BLOCK-PING term 1 from protocol icmp

set firewall family inet filter BLOCK-PING term 1 then count PING-ATTEMPTS

set firewall family inet filter BLOCK-PING term 1 then reject

set firewall family inet filter BLOCK-PING term 2 then accept

set interfaces irb unit 10 family inet filter input BLOCK-PING

commit

```

- **Verification:**

1. Configure `client1` (`ip addr add 192.168.10.1/24 dev eth1; ip route add default via 192.168.10.253`) and `client2` (`...192.168.20.1/24 ... 192.168.20.252`).

2. From `client1`, `ping 192.168.20.1`. It should fail with a "Destination Port Unreachable" message.

3. On `dist1`, run `show firewall filter BLOCK-PING`. The `PING-ATTEMPTS` counter should be greater than zero.

  

---

  

## Module 7: Advanced Monitoring & Maintenance

  

**Goal:** Configure services that allow for external monitoring and robust device management.

  

### 7.1: Configure SNMP for Monitoring

  

- **Applies to:** Any single Juniper device (e.g., `dist1`)

- **Steps:**

```junos

# [On dist1]

set snmp location "3-Tier-Lab"

set snmp contact "labadmin"

set snmp community public authorization read-only

commit

```

- **Verification:** Requires an external tool. From a host that can reach `dist1`, you would run `snmpwalk -v2c -c public <dist1_ip> system.sysDescr.0`.

  

---

  

## Module 9: High Availability with VRRP

  

**Goal:** Provide a single, redundant default gateway for clients in VLAN 10 using VRRP.

  

### 9.1: Configure VRRP

  

- **Applies to:** `dist1`, `dist2`

- **Steps:**

  

```junos

# [On dist1 - Master]

set interfaces irb unit 10 family inet address 192.168.10.253/24 vrrp-group 10 virtual-address 192.168.10.254

set interfaces irb unit 10 family inet address 192.168.10.253/24 vrrp-group 10 priority 150

commit

  

# [On dist2 - Backup]

set interfaces irb unit 10 family inet address 192.168.10.252/24 vrrp-group 10 virtual-address 192.168.10.254

commit

```

  

- **Verification:**

1. Update `client1`'s default gateway to the virtual IP: `ip route add default via 192.168.10.254`.

2. On `dist1` and `dist2`, run `show vrrp summary`. `dist1` should be `master`, `dist2` should be `backup`.

3. From `client1`, ping an external address like `core1`'s loopback (`1.1.1.1`).

4. On `dist1`, deactivate the IRB interface (`deactivate interfaces irb unit 10`). Commit.

5. Run `show vrrp summary` on `dist2`. It should now be `master`. The ping from `client1` should continue with minimal interruption.

[^1]: 

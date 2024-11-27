## Topology

![[Pasted image 20241123094119.png]]

## Addressing Table

| Device | Interface | IPv4 Address  | IPv6 Address              | IPv6 Link-Local |
| ------ | --------- | ------------- | ------------------------- | --------------- |
| R1     | G0/0/1    | 10.1.13.1/24  | 2001:db8:acad:10d1::1/64  | fe80::1:1       |
| R1     | S0/1/1    | 10.1.3.1/24   | 2001:db8:acad:1013::1/64  | fe80::1:2       |
| D1     | G1/0/11   | 10.1.13.13/24 | 2001:db8:acad:10d1::d1/64 | fe80::d1:1      |
| D1     | VLAN50    | 10.2.50.1/24  | 2001:db8:acad:1050::d1/64 | fe80::d1:2      |
| D1     | VLAN60    | 10.2.60.1/24  | 2001:db8:acad:1060::d1/64 | fe80::d1:3      |
| R3     | S0/1/1    | 10.1.3.3/24   | 2001:db8:acad:1013::3/64  | fe80::3:1       |
| R3     | G0/0/1.75 | 10.3.75.1/24  | 2001:db8:acad:3075::1/64  | fe80::3:2       |
| R3     | G0/0/1.85 | 10.3.85.1/24  | 2001:db8:acad:3085::1/64  | fe80::3:3       |
| D2     | VLAN75    | 10.3.75.14/24 | 2001:db8:acad:3075::d2/64 | fe80::d2:1      |
| PC1    | NIC       | 10.2.50.50/24 | 2001:db8:acad:1050::50/64 | EUI-64          |
| PC2    | NIC       | 10.2.60.50/24 | 2001:db8:acad:1060::50/64 | EUI-64          |
| PC3    | NIC       | 10.3.75.50/24 | 2001:db8:acad:3075::50/64 | EUI-64          |
| PC4    | NIC       | 10.3.85.50/24 | 2001:db8:acad:3085::50/64 | EUI-64          |

## Objectives
Part 1: Build the Network and Configure Basic Device Settings

Part 2: Configure and Verify Inter-VLAN Routing on a Layer 3 Switch

Part 3: Configure and Verify Router-based Inter-VLAN Routing

Part 4: Examine CAM and CEF Details

# Background / Scenario

The methods used to move packets and frames from one interface to the next has changed over the years. In this lab you will configure Inter-VLAN Routing in its various forms and then examine the different tables used in making forwarding decisions.

**Note**: This lab is an exercise in configuring and verifying various methods of Inter-VLAN routing and does not reflect networking best practices.

**Note**: The routers and switches used with CCNP hands-on labs are Cisco 4221 and Cisco 3650, both with Cisco IOS XE Release 16.9.4 (universalk9 image). Other routers and Cisco IOS versions can be used. Depending on the model and Cisco IOS version, the commands available and the output produced might vary from what is shown in the labs.

**Note**: Ensure that the routers and switches have been erased and have no startup configurations. If you are unsure contact your instructor.

# Instructions

## Part 1: Build the Network and Configure Basic Device Settings

In Part 1, you will set up the network topology and configure basic settings.

### Step 1: Cable the network as shown in the topology.

Attach the devices as shown in the topology diagram, and cable as necessary.
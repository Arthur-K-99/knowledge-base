---
title: "Chapter 3: Advanced STP Tuning"
---
# Content

**This chapter covers the following content:**

- **STP Topology Tuning -** This section explains some of the options for modifying the root bridge location or moving blocking ports to designated ports.

- **Additional STP Protection Mechanisms -** This section examines protection mechanisms such as root guard, BPDU guard, and STP loop guard.

## STP Topology Tuning

- In a properly designed network a switch is deliberately selected to become the root bridge and the designated and alternate ports are modified.

- Network design considerations factor in hardware platform, resiliency, and network topology.

### Root Bridge Placement

To ensure root bridge placement set the system priority on:

- The root bridge to the lowest value

- The secondary root bridge to a value slightly higher than that of the root bridge

- All other switches to a value higher than the secondary root bridge

| **Command**                                                                                 | **Description**                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **spanning-tree vlan** _vlan-id_ **priority** _priority_                                    | The priority is a value between 0 and 61,440, in increments of 4,096.                                                                                                                                                                             |
| **spanning-tree vlan** _vlan-id_ **root {primary \| secondary} \[diameter** _diameter_**]** | The **primary** keyword sets the priority to 24,576, and the **secondary** keyword sets the priority to 28,672. The optional **diameter** command makes it possible to tune the Spanning Tree Protocol (STP) convergence and modifies the timers. |

### Configuring the Root Bridge

In the example:

- The initial priority for VLAN 1 on SW1 is verified, 32,769. 

- SW1 is configured to be the primary root for VLAN 1

- The priority is verified again to ensure the change took place.

![[Pasted image 20250225194719.png]]

### Configuring the Backup Root Bridge

In the example:

- The initial priority for VLAN 1 on SW2 is verified, 32,769. 

- SW2 is configured to be the secondary root for VLAN 1

- The priority is verified again to ensure the change took place.

![[Pasted image 20250225194739.png]]

### Modifying STP Root Port & Blocked Switch Port Locations

Calculating total path cost to the root bridge:

- SW1 sends a BPDU to SW3 with the path cost of 0. 

- SW3 receives the BPDU and adds its root port cost (4) to cost from the BPDU (0), resulting in the cost of 4.

- SW3 sends a BPDU to SW5 with the path cost of 4.  

- SW5 receives the BPDU and adds its root port cost (4) to the cost from the BPDU (4), resulting in the cost of 8 for SW5 to reach the root bridge.

![[Pasted image 20250225194807.png]]

### Verifying the Total Path Cost

The example highlights the total path cost to the root bridge from SW3 and SW5.

![[Pasted image 20250225194848.png]]

![[Pasted image 20250225194852.png]]
**Note**: There is not a total path cost in SW1’s output

### Modifying STP Port Cost

•The **spanning tree** \[**vlan** _vlan-id]_ **cost** _cost_ command can be used to modify the STP forwarding path. 

•Using the spanning tree command will modify the cost for all VLANs unless the optional **vlan** keyword is used.

![[Pasted image 20250225194926.png]]

### Modifying STP Port Priority

STP port priority influences which port becomes the alternate port when multiple links are used between switches. Use the command **spanning-tree** \[**vlan** _vlan-id_] **port-priority** _priority_ to change the STP port priority on a switch’s interface.

![[Pasted image 20250225194952.png]]
![[Pasted image 20250225194955.png]]
![[Pasted image 20250225194958.png]]
### Additional STP Protection Mechanisms

- A network forwarding loop occurs when there are multiple active paths between two devices. Broadcast and multicast traffic are forwarded out every switch port continuing the forwarding loop. 

- The network’s throughput is drastically effected as the switches are processing numerous frames. The switches CPU utilization will be high and memory space will be consumed. The switches might crash and users will likely notice the impact on the network.

Common issues for Layer 2 forwarding loops:

- STP is disabled on a switch.

- A load balancer is misconfigured and sends traffic out multiple ports with the same MAC address.

- A virtual switch that bridges two physical ports.

- End users using an unmanaged switch or hub.

### Root Guard

Root guard is an STP feature that prevents a configured port from becoming a root port.

- It does this by placing the port in an ErrDisabled state if a superior BDPU is received on that port. 

- Root guard is placed on designated ports towards other switches that should never become root bridges. 

- Root guard is enabled on a port-by-port basis.

Use the interface command **spanning-tree guard root** to enable root guard.

### STP Portfast

STP portfast disables the topology notification notification (TCN) generation and causes access ports that come up to bypass the learning and listening states and enter the forwarding state immediately. If a BPDU is received on a portfast-enabled port, the portfast functionality is removed from that port.

| **Command**                        | **Description**                                                                                                                 |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **spanning-tree portfast**         | Interface command to enable portfast on a specific access port                                                                  |
| **spanning-tree portfast default** | Global command to enable portfast on all access ports                                                                           |
| **spanning-tree portfast disable** | Disable portfast on a port                                                                                                      |
| **spanning-tree portfast trunk**   | Command used on trunk links to enable portfast<br><br>* This command should only be used with ports connected to a single host. |

### STP Portfast Examples

The following shows how to enable STP portfast globally and on a specific interface.

![[Pasted image 20250225195211.png]]
![[Pasted image 20250225195217.png]]

### BPDU Guard

BPDU guard is a safety mechanism that shuts down ports configured with STP portfast upon receiving a BPDU.

| **Command**                                                              | **Description**                                                            |
| ------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| **spanning-tree portfast bpduguard default**                             | Global command to enable BPDU guard on all STP portfast ports              |
| **spanning-tree portfast bpduguard default** {**enable** \| **disable**} | Interface command to enables or disable BPDU guard on a specific interface |
| **show spanning-tree interface** _interface-id_ **detail**               | Displays whether BPDU guard is enabled for the specified interface         |

**Note**: BPDU Guard is typically configured with all host-facing ports that are enabled with portfast.

### BPDU Guard Examples

The following shows how to configure BPDU guard and a BPDU guard-enabled port detecting a BPDU.

![[Pasted image 20250225195303.png]]

![[Pasted image 20250225195306.png]]
The **show interfaces status**command shows the err-disabled status of the port that received the BPDU.

### BPDU Guard Error Recovery

The Error Recovery service can be used to reactivate ports that are shut down. Ports that are put into the ErrDisabled mode due to BPDU guard do not automatically restore themselves. Use the following commands to recover ports that were shutdown from BPDU guard:

| **Command**                                     | **Description**                                 |
| ----------------------------------------------- | ----------------------------------------------- |
| **errdisable recovery cause bpduguard**         | Recovers ports shutdown by BPDU guard           |
| **errdisable recovery interval** _time-seconds_ | The period that Error Recovery checks for ports |

### BPDU Guard Error Recovery Example

The following example shows how to configure the Error Recovery service.

![[Pasted image 20250225195351.png]]
![[Pasted image 20250225195354.png]]
Note: The Error Recovery service operates every 300 seconds (5 minutes). This can be changed from 5 to 86,000 seconds with the global command **errdisable recovery interval** _time_

### BPDU Filter

BPDU filter blocks BPDUs from being transmitted out of a port. It can be enabled globally or on a specific interface. 

Global BPDU filter command:

**spanning-tree portfast bpdufilter default**

With the global BPDU configuration the port sends a series of 10 –12 BPDUs. If the switch receives any BPDUs, it checks to identify which switch is more preferred.

- The preferred switch doesn’t process any BPDUs but still passes them along to inferior switches. 

- A non-preferred switch processes the BPDUs that are received but doesn’t transmit any BPDUs to superior switches.

Interface-specific BPDU filter command:

**Spanning-tree bpdufilter enable**

With the interface-specific BPDU configuration the port does not send any BPDUs on an ongoing basis. If the remote port has BPDU guard, that generally shuts down the port as a loop prevention mechanism.

### Verifying a BPDU Filter

The following shows using the **show spanning-tree interface** _interface-id_ **detail** command to verify that BPDU filter is enabled.

![[Pasted image 20250225195453.png]]

### Problems with Unidirectional Links

Network devices that utilize fiber-optic cables for connectivity can encounter unidirectional traffic flows if one strand is broken. BPDUs will not able to be transmitted causing other switches on the network to eventually time out the existing root port and change root ports resulting in a forwarding loop.

Two solutions to problems with unidirectional links:

- STP Guard

- Unidirectional Link Detection

### STP Loop Guard

STP Loop guard prevents any alternative or root ports from becoming designated ports due to loss of BPDUs on the root port.  Loop guard places the original port into an ErrDisabled state while BPDUs are not being received and transitions back through the STP states when it begins receiving BPDUs again.

| **Command**                               | **Description**                                                           |
| ----------------------------------------- | ------------------------------------------------------------------------- |
| **spanning-tree loopguard default**       | Global command to enable loop guard                                       |
| **spanning-tree guard loop**              | Interface command to enable loop guard                                    |
| **show spanning-tree inconsistent-ports** | Shows ports in the inconsistent state due to the port not receiving BPDUs |
**Note**: Loop guard shouldn’t be enabled on portfast-enabled ports because it directly conflicts with root/alternate port logic

### STP Loop Guard Examples

The following examples show configuring loop guard, triggering loop guard by blocking BPDUs and the port in an inconsistent state.

![[Pasted image 20250225195552.png]]

![[Pasted image 20250225195555.png]]
### Unidirectional Link Detection

Unidirectional Link Detection (UDLD) allows for the bidirectional monitoring of fiber-optic cables.

UDLD operates in two modes:

- **Normal** – If a frame is not acknowledged, the link is considered undetermined and the port remains active.

- **Aggressive** – If a frame is not acknowledged, the switch sends another 8 packets in 1 second intervals. If those packets aren’t acknowledged, the port is placed into an error state.

### UDLD Commands

The following are commands for configuring and verifying UDLD:

| **Command**                             | **Description**                                                                            |
| --------------------------------------- | ------------------------------------------------------------------------------------------ |
| **udld enable** [**aggressive**]        | Global command to enable UDLD. *Optional aggressive keyword sets the mode to aggressive.   |
| **udld port** [**aggressive**]          | Interface command to enable UDLD *Optional aggressive keyword sets the mode to aggressive. |
| **udld port disable**                   | Disable UDLD on a specific interface                                                       |
| **udld recovery** [**interval** _time_] | Enables UDLD recovery. The _time_default value is 5 minutes.                               |
| **show udld neighbors**                 | Displays the status of UDLD neighborship                                                   |
| **show udld** _interface-id_            | Displays detailed information about UDLD                                                   |
### Configuring & Verifying UDLD Examples

The following are examples for configuring and verifying UDLD:

![[Pasted image 20250225195923.png]]
![[Pasted image 20250225195934.png]]
![[Pasted image 20250225195927.png]]
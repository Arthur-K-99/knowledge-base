EVPN (Ethernet VPN) is a **networking technology** designed to provide highly scalable and flexible Layer 2 and Layer 3 virtual private network (VPN) services over IP/MPLS networks. It is defined in [RFC 7432](https://www.rfc-editor.org/rfc/rfc7432) and is commonly used in data centers, service provider networks, and enterprise WANs.

---

### Key Concepts of EVPN:
1. **Layer 2 and Layer 3 Support**:
   - EVPN extends Ethernet connectivity across geographically dispersed locations while also supporting Layer 3 IP routing.

2. **Control Plane**:
   - EVPN uses **MP-BGP (Multiprotocol Border Gateway Protocol)** to distribute Layer 2 and Layer 3 reachability information. This is more efficient than traditional flooding methods like MAC learning in typical Layer 2 networks.

3. **Overlay Networking**:
   - EVPN can act as an overlay solution, running over an IP/MPLS network. It uses VXLAN, MPLS, or other tunneling protocols to encapsulate traffic.

4. **MAC Address Learning**:
   - Instead of relying on flooding to learn MAC addresses, EVPN uses the control plane (BGP) for efficient MAC distribution, improving scalability and reducing unnecessary broadcast traffic.

5. **Multi-Homing and Redundancy**:
   - EVPN supports **active-active** or **active-standby multi-homing** for resiliency and load balancing when connecting to multiple endpoints or devices.

6. **EVPN-VXLAN**:
   - A common implementation involves EVPN working with **VXLAN (Virtual Extensible LAN)** to extend Layer 2 domains over Layer 3 networks, especially in data center interconnects.

---

### Benefits of EVPN:
- **Scalability**: Efficient handling of large networks with many devices and locations.
- **Flexibility**: Support for both Layer 2 and Layer 3 VPN services.
- **Improved Resiliency**: Built-in multi-homing capabilities for redundancy and fault tolerance.
- **Operational Efficiency**: Reduces broadcast traffic by using BGP for control-plane learning.
- **Seamless Data Center Interconnect**: Extends VLANs across data centers with ease.
- **Interoperability**: Works with existing MPLS, IP, and VXLAN-based networks.

---

### Common Use Cases:
1. **Data Center Interconnect (DCI)**:
   - Connects multiple data centers and provides consistent Layer 2/3 connectivity.
2. **Service Provider Networks**:
   - Supports Ethernet-based VPN services for enterprise customers.
3. **Campus Networks**:
   - Extends VLANs across campus environments.
4. **Enterprise WANs**:
   - Replaces traditional MPLS L2/L3 VPNs for branch office connectivity.

## Vendor Specific
* [[Arista BGP EVPN]]
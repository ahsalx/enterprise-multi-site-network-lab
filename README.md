# Multi-Site Enterprise Network Lab PNETLab

A multi-site enterprise network built in **PNETLab** using Cisco IOS and FortiGate. The lab connects an HQ and two branches through a routed provider network, with separate networks for Staff, CCTV, Printers, and Management.

The design was inspired by a real enterprise topology that an experienced network engineer in Lisbon walked me through and reviewed with me while I was building the lab. I adapted the idea to the technologies I had already studied during my CCNA and could configure and troubleshoot myself.

**Core technologies:** VLANs · Layer 3 switching · OSPF · FortiGate HA · Firewall Policies · Extended ACLs · LACP/EtherChannel · DHCP · SSH · Port Security

---

## 1. Architecture

![Final PNETLab topology](screenshots/01-topology-overview.png)

The network is split into three sites: **HQ, Branch 1, and Branch 2**. A router named `PE_MPLS` represents the provider network and connects the three sites.

Both branches follow the same basic design: one Layer 3 switch handles the local VLAN gateways and inter-VLAN routing, while two routers provide separate paths toward the provider. I used two WAN routers so the branch can keep an alternate path if one router or one provider-facing link fails.

HQ is designed differently because it is the main security and management site. It has two switches connected with an LACP EtherChannel and two FortiGates running as an active/passive HA pair.

For the provider-facing side of HQ, I chose to place a shared transit switch between the FortiGate pair and `PE_MPLS`. The alternative would have been to give each FortiGate its own dedicated provider-facing connection. Using the shared switch keeps the design simpler and reduces the number of links and interfaces needed. The trade-off is that the shared transit switch remains a single point of failure.

Security follows the actual traffic path. Traffic going to or from HQ crosses the FortiGate, so the firewall is the main policy point for HQ. Branch-to-branch traffic can stay inside the provider path and never reach the FortiGate, so the branch Layer 3 switches also use ACLs to protect sensitive networks such as CCTV.

For example, traffic from a Branch 1 Staff PC to an HQ Staff host follows this path:

```text
Branch endpoint → Branch L3 switch → R1 or R2 → PE_MPLS → HQ transit switch → FortiGate HA pair → HQ VLAN
```

Internet access is not yet configured in the current lab. The intended design is for Internet-bound branch traffic to pass through HQ and the FortiGate, and this can be added later.

---

## 2. VLAN and IP Plan

I kept VLANs 10, 20, and 30 consistent across all sites. VLAN 99 is used at HQ for Administration/Management:

| VLAN | Role |
|---:|---|
| 10 | Staff |
| 20 | CCTV |
| 30 | Printers |
| 99 | Administration / Management |

Each site uses different subnets, but the addressing follows a pattern:

```text
Branch 1: 192.168.10.0 / 20.0 / 30.0
Branch 2: 192.168.110.0 / 120.0 / 130.0
HQ:       192.168.210.0 / 220.0 / 230.0
```

That makes it easier to identify both the site and the device role from the IP address. For example, `192.168.220.x` belongs to the HQ CCTV network.

At HQ, the VLAN gateways are FortiGate subinterfaces. The screenshot below shows VLAN 10 for HQ Staff on `port1` with gateway `192.168.210.1/24`.

![FortiGate VLAN 10](screenshots/02-vlan10-fortigate-interface.png)

### Branch 1

![Branch 1 VLANs and routed links](screenshots/branch1-addressing.png)

### Branch 2

![Branch 2 VLANs and routed links](screenshots/branch2-addressing.png)

### HQ

![HQ VLANs and provider transit](screenshots/hq-addressing.png)

The point-to-point routed links use `/30` networks because only two usable addresses are needed on each link.

---

## 3. Routing Design

`PE_MPLS` acts as the central provider/transit router between HQ and both branches.

The name represents the provider role in the topology; this lab does **not** implement real MPLS label switching, LDP, or MPLS VPNs. The provider side is IP routing with OSPF.

| Link | PE Interface | Address |
|---|---|---|
| Branch 1 — R1 | Ethernet0/0 | `10.0.1.1/30` |
| Branch 1 — R2 | Ethernet0/2 | `10.0.1.5/30` |
| Branch 2 — R3 | Ethernet0/3 | `10.0.2.1/30` |
| Branch 2 — R4 | Ethernet0/1 | `10.0.2.5/30` |
| HQ — FortiGate | Ethernet1/0 | `10.0.3.2/30` |

I used **OSPF Area 0** for dynamic routing. OSPF was the routing protocol I knew from my CCNA studies, and it fit what I needed here.

![Branch 2 OSPF configuration](screenshots/16-branch2-ospf-configuration.png)

On the HQ side, the FortiGate forms an OSPF adjacency with `10.0.3.2` and advertises the HQ networks.

![FortiGate OSPF](screenshots/04-fortigate-ospf-neighbor.png)

I also verified basic Layer 3 connectivity between the provider router and the FortiGate before working on firewall policy.

![PE to FortiGate ping](screenshots/05-provider-fortigate-ping.png)

---

## 4. FortiGate HA and Security Policy

The HQ uses two FortiGate VMs in an **active/passive HA pair**. `port4` is used as the heartbeat link between them.

![FortiGate HA cluster](screenshots/03-fortigate-ha-cluster.png)

The cluster reached a synchronized Primary/Secondary state. HA removes the individual firewall as a single point of failure.

### Address objects and groups

I created address objects for the networks instead of building policies around raw IP addresses.

![FortiGate address objects](screenshots/06-fortigate-address-objects.png)

Examples:

```text
BR1-STAFF     192.168.10.0/24
BR1-CCTV      192.168.20.0/24
BR1-PRINTER   192.168.30.0/24
BR2-STAFF     192.168.110.0/24
BR2-CCTV      192.168.120.0/24
BR2-PRINTER   192.168.130.0/24
HQ-STAFF      192.168.210.0/24
HQ-CCTV       192.168.220.0/24
HQ-PRINTER    192.168.230.0/24
ADMIN-NET     192.168.99.0/28
```

Objects with the same role are grouped together, for example `ALL-STAFF`.

![ALL-STAFF address group](screenshots/07-fortigate-all-staff-group.png)

That keeps the firewall rules readable.

### Firewall policy

The policy is based on allowing the traffic that is actually required and leaving the rest to FortiGate's implicit deny.

![FortiGate firewall policies](screenshots/08-fortigate-firewall-policies.png)

| Source | Destination | Result |
|---|---|---|
| Branch Staff | HQ Staff | Allow |
| HQ Staff | Branch Staff / Printers | Allow |
| HQ Security / CCTV | Branch CCTV | Allow |
| Admin | Branch networks | Allow |
| Admin | HQ Staff / CCTV / Printer VLANs | Allow |
| Other unmatched traffic | Protected networks | Implicit deny |

FortiGate is stateful, so return traffic for an established allowed session does not need a second reverse rule.

---

## 5. Branch ACL Security

The FortiGate is not in the path for every packet in the network. Branch-to-branch traffic can cross the provider directly, so I could not rely on the HQ firewall to protect the branch CCTV networks.

For that reason, I used extended ACLs on the branch Layer 3 switches to control traffic going toward CCTV.

![Extended ACL configuration](screenshots/10-extended-acl-icmp.png)

An extended ACL was used to match both the **source** and the **destination**.

The ACL was tested from a VPCS host:

![ACL block verification](screenshots/11-acl-block-test.png)

The `administratively prohibited` response was proof that the packet reached the routing device and was intentionally rejected by the ACL.

At HQ, I did not duplicate the same ACL design because the FortiGate already sits between the relevant networks and handles it with its implicit deny firewall policy.

---

## 6. HQ Switching and EtherChannel

`HqSwitch4` and `HqSwitch5` are connected using a two-link **LACP EtherChannel**.

![HQ EtherChannel links](screenshots/15-hq-etherchannel-links.png)

The Port-Channel carries VLANs `10,20,30,99`:

```cisco
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20,30,99
 switchport mode trunk
```

The physical member interfaces use matching trunk settings and join `channel-group 1`.

I used EtherChannel for two reasons: it gives the switches more than one physical link between them without creating two independent Layer 2 paths, and one member link can fail while the Port-Channel stays up on the remaining link.

---

## 7. Management and SSH

The infrastructure is managed through dedicated management addresses.

Branch routers and switches use loopback addresses as stable SSH targets. At HQ, management is separated through VLAN 99.

The Cisco SSH configuration follows this pattern:

```cisco
username admin secret \<REDACTED>
ip domain-name lab.lab
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
```

The Linux admin workstation was then able to SSH to the branch management addresses.

![SSH management](screenshots/09-ssh-management.png)

---

## 8. DHCP

Normal endpoints use DHCP, while infrastructure devices keep predictable static addresses.

At HQ, DHCP is provided by the FortiGate for the Staff, CCTV, and Printer VLANs:

![FortiGate DHCP leases](screenshots/13-dhcp-leases-hq.png)

At the branches, the Layer 3 switches provide DHCP locally for their VLANs.

![Branch 1 DHCP server and binding](screenshots/17-branch1-dhcp-server.png)

A VPCS Staff client successfully received an IP address and default gateway through DHCP:

![VPCS DHCP DORA](screenshots/14-dhcp-dora-vpcs.png)

---

## 9. Layer-2 Access Security

Port Security is enabled on endpoint-facing switchports, preventing someone from disconnecting a known endpoint and connecting another device to the same access port.

```cisco
switchport port-security
switchport port-security mac-address sticky
switchport port-security violation shutdown
```

![Switchport Port Security](screenshots/12-switchport-port-security.png)

---

## 10. Troubleshooting Highlights

### EtherChannel would not bundle

**Symptom:** IOS generated `%EC-5-CANNOT_BUNDLE2`.

**Cause:** I had configured the trunk on the Port-Channel, but the physical member interfaces did not have matching trunk parameters.

**Fix:** I configured the member interfaces with the same `dot1q` trunk settings before adding them to the channel group.

### Modern Linux SSH client rejected the Cisco connection

**Symptom:** `no matching cipher found`.

**Cause:** The older IOS image only offered SSH algorithms that modern OpenSSH disables by default.

**Fix:** I kept SSHv2 enabled on Cisco and allowed the required legacy algorithms only for the lab management ranges on the Linux client.

### FortiGate replied to ping but the GUI did not open over HTTPS

The network path was working because the firewall replied to ICMP, but the HTTPS management service wasn't working correctly. With this FortiGate image, I used HTTP management as a workaround.

# Secure Enterprise Campus Network (Cisco Packet Tracer)

I built this lab to practice the core topics from the CCNA syllabus in one connected network, instead of studying each topic on its own. The network has five VLANs, one router doing the inter-VLAN routing, a central DHCP server, internet access through PAT, ACLs to keep guests away from internal systems, and security features on the access switches.

Everything was built and tested in Cisco Packet Tracer. I also broke the network on purpose six times to practice troubleshooting. Those notes are in [troubleshooting.md](troubleshooting.md).

![Topology](screenshots/topology.png)

## Devices used

- 1 Cisco 2811 router (R1), which does the inter-VLAN routing and NAT
- 1 router acting as the ISP
- 4 Cisco 2960-24TT switches (SW-Core, SW1, SW2, SW3)
- 1 server for DHCP, DNS and HTTP, plus 1 server acting as the internet (8.8.8.8)
- PCs for Sales, HR and IT
- 1 access point and 1 laptop for the Guest Wi-Fi

## VLAN and IP plan

| VLAN | Name | Subnet | Gateway (R1 subinterface) |
|---|---|---|---|
| 10 | Sales | 192.168.10.0/24 | 192.168.10.1 |
| 20 | HR | 192.168.20.0/24 | 192.168.20.1 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 |
| 40 | Servers | 192.168.40.0/24 | 192.168.40.1 |
| 50 | Guest | 192.168.50.0/24 | 192.168.50.1 |
| 99 | Management and native | 192.168.99.0/24 | 192.168.99.1 |

Other addresses:

| Item | Address |
|---|---|
| DHCP/DNS server | 192.168.40.10 |
| Switch management IPs | 192.168.99.2 (SW-Core), .3 (SW1), .4 (SW2), .5 (SW3) |
| R1 to ISP link | 203.0.113.0/30 (R1 is .2, ISP is .1) |
| Simulated internet server | 8.8.8.8 on 8.8.8.0/24 |

## What I set up

**Switching**
- VLANs 10, 20, 30, 40, 50 and 99 on all four switches
- 802.1Q trunks between the switches and to the router, with VLAN 99 as the native VLAN instead of the default VLAN 1
- Access ports assigned to the right VLAN for each device

**Routing**
- Router-on-a-stick: one physical link from R1 to SW-Core, split into subinterfaces, one per VLAN
- Management IPs on the switches in VLAN 99, with a default gateway so they can be reached from other subnets

**Services**
- One central DHCP server in VLAN 40 serving the Sales, HR, IT and Guest VLANs
- `ip helper-address` on the router subinterfaces so DHCP requests are relayed across VLANs
- PAT (NAT overload) on R1, so all internal VLANs share one public address to reach the simulated internet

**Access control**
- A Guest ACL that lets guests get an address, use DNS and reach the internet, but blocks them from every internal network
- An ACL that stops HR from reaching the IT VLAN

**Switch security**
- DHCP snooping with trusted uplinks and server port
- Port security with sticky MAC addresses and shutdown on violation
- PortFast with BPDU guard on access ports
- Unused ports shut down

**Wireless**
- A Guest SSID on an access point connected to an access port in VLAN 50, with a laptop as the guest client

## Testing

| Test | Result |
|---|---|
| Ping between PCs in different VLANs | Works |
| Ping from Sales, HR and IT to the server | Works |
| Every VLAN gets an address by DHCP | Works |
| Ping from internal PCs to 8.8.8.8 | Works |
| `show ip nat translations` shows PAT entries | Works |
| Guest laptop gets a 192.168.50.x address over Wi-Fi | Works |
| Guest to HR, IT and the server | Blocked by the ACL |
| Guest to 8.8.8.8 | Works |
| HR to IT | Blocked by the ACL |
| Unknown PC plugged into a secured port | Port goes err-disabled |
| Switch plugged into a BPDU guard port | Port goes err-disabled |
| Rogue DHCP server on an access port | Tested, then removed from the lab |

Screenshots for these are in the `screenshots` folder.

## Troubleshooting practice

After the network was working, I introduced six faults one at a time and wrote down the symptom, how I found the cause, and what was wrong. I fixed each one and re-tested before moving to the next.

1. Native VLAN mismatch on a trunk
2. Missing DHCP relay on a subinterface
3. PC port placed in the wrong VLAN
4. VLAN removed from a trunk's allowed list
5. Wrong encapsulation number on a subinterface
6. Missing default route on R1

Full write-up: [troubleshooting.md](troubleshooting.md)

## What I learned

- A subinterface that shows up/up is not proof that it works. The encapsulation number has to match the VLAN, and I only found that by reading the config.
- ACL order matters, and the direction you apply it in decides whether it ever matches anything.
- Most of my problems came from small mismatches between two ends of a link, like the native VLAN on a trunk. Comparing both ends with `show interfaces trunk` solved it quickly.
- Turning on a security feature can quietly break something that worked before, so I re-ran the full set of tests after each one.

## Limits and things I would add next

- Router-on-a-stick sends all inter-VLAN traffic over one link. A Layer 3 switch would remove that bottleneck.
- The native VLAN and the management VLAN are the same (VLAN 99). Best practice is to keep them separate.
- There is only one core switch and one router, so there is no redundancy yet. HSRP and EtherChannel would be the next step.
- A second site with OSPF would turn this into a multi-branch design.

## Files in this repo

- `campus-network.pkt`: the Packet Tracer file
- `configs/`: running configs of R1, ISP and the four switches
- `screenshots/`: topology, test results and troubleshooting screenshots
- `troubleshooting.md`: the six faults I created and diagnosed

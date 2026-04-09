# Playbook Review

This document reviews [playbook.yml](playbook.yml) as an Arista EOS VXLAN EVPN fabric build that uses:

- eBGP for the routed underlay
- eBGP for the EVPN overlay
- leafs as VTEPs and distributed anycast gateways
- spines as the routed core and EVPN control-plane transit nodes

## Review Summary

The current playbook is structured like a straightforward non-MLAG leaf/spine EVPN fabric. It follows the normal EOS pattern where leafs own tenant VRFs, VLANs, SVIs, VXLAN VNIs, and VTEP functions, while spines provide IP underlay reachability and relay EVPN routes between leafs over loopback-based BGP sessions.

## Current Fabric Variables

- `loopback_base`: `192.168.1`
- `links`: routed `/31` leaf-to-spine underlay map from `group_vars/all.yml`
- `tenants`: `TENANT-1` and `TENANT-2`
- `vlans`: VLANs `10`, `20`, and `30`

## Source Key

- `OV`: Arista EOS EVPN Overview, [EOS EVPN Overview](https://www.arista.com/en/um-eos/eos-evpn-overview?searchword=eos+multi)
- `SC`: Arista EOS Sample Configurations, [EOS Sample Configurations](https://www.arista.com/en/um-eos/eos-sample-configurations?searchword=eos+section+21+4+evpn+type+5+routes+ip+prefix+advertisement)
- `ANS`: Ansible docs, [arista.eos.eos_config module](https://docs.ansible.com/ansible/latest/collections/arista/eos/eos_config_module.html)

## Leaf Play

| Line | Play or Task | Reason | Source |
| --- | --- | --- | --- |
| 5 | Configure leafs (VXLAN EVPN) | Applies all tenant-facing VTEP, gateway, underlay, and EVPN control-plane configuration to the leaf switches. | `OV`, `SC` |
| 16 | Enable routing | Enables EOS routing services, IP forwarding, local exec authorization, and the shared anycast gateway MAC used by distributed SVIs. | `SC` |
| 24 | Configure local cg user | Creates a local administrative user and SSH key so automation or operators can access the switch without external AAA. | Local playbook |
| 30 | Configure Loopback | Creates Loopback0 as the stable router-id and EVPN peering source address. | `SC` |
| 36 | Configure VTEP loopback | Creates Loopback1 as the VXLAN source address for encapsulation and decapsulation. | `OV`, `SC` |
| 42 | Configure VRFs | Creates tenant VRF instances so each tenant gets an isolated routing domain. | `OV`, `SC` |
| 48 | Enable VRF routing | Turns on IPv4 forwarding inside each tenant VRF so routed tenant interfaces can operate. | `SC` |
| 57 | Configure leaf underlay interfaces | Builds the routed point-to-point uplinks from each leaf to the spines with jumbo MTU for the fabric. | `SC` |
| 71 | Configure BGP underlay base for leafs | Instantiates the leaf BGP process for underlay reachability and advertises Loopback0 plus the VTEP loopback into the IP fabric. | `SC` |
| 82 | Configure UNDERLAY peer group on leafs | Creates a reusable BGP peer-group for underlay spine neighbors to keep the leaf underlay policy consistent. | `SC` |
| 89 | Configure leaf underlay BGP neighbors | Adds each directly connected spine as a routed eBGP underlay neighbor. | `SC` |
| 97 | Activate leaf underlay neighbors for IPv4 | Enables the underlay peer-group inside the IPv4 address family so the point-to-point sessions actually exchange routes. | `SC` |
| 108 | Configure EVPN base | Sets the BGP router-id and enables EVPN route distinguisher behavior for overlay advertisements. | `OV`, `SC` |
| 115 | Configure EVPN peer group on leafs | Creates a reusable EVPN peer-group for loopback-based multihop overlay sessions toward the spines. | `SC` |
| 126 | Configure leaf EVPN neighbors (leaf -> spines only) | Adds the spine loopbacks as EVPN overlay peers so the leaf can advertise and learn tenant reachability through the fabric. | `SC` |
| 133 | Activate leaf EVPN neighbors | Enables the EVPN peer-group under the EVPN address family. | `OV`, `SC` |
| 144 | Configure VLANs | Creates the tenant VLAN objects that back the bridged VXLAN services. | `SC` |
| 150 | Configure VLAN gateways | Creates the SVI gateways, binds them to VRFs, and gives each VLAN its anycast default gateway. | `SC` |
| 158 | Configure VXLAN interface | Creates the Vxlan1 interface and points it at the dedicated VTEP loopback using the standard VXLAN UDP port. | `OV`, `SC` |
| 166 | Map VNIs | Maps each tenant L2 VLAN to its corresponding L2 VNI on the VTEP. | `OV`, `SC` |
| 173 | Map L3 VNIs | Maps each tenant VRF to its L3 VNI so inter-subnet tenant routing can be carried across the overlay. | `OV`, `SC` |
| 180 | Enable EVPN VLANs | Enables per-VLAN EVPN advertisement, sets an RD/RT policy for the L2 VNI, and redistributes learned host information into EVPN. | `OV`, `SC` |
| 191 | Configure EVPN VRF base | Enables per-VRF EVPN advertisement, defines RD/RT policy for each tenant VRF, and redistributes connected tenant routes. | `OV`, `SC` |
| 203 | Configure EVPN VRF IPv4 route-targets | Repeats the tenant VRF import/export policy under IPv4 AF so tenant prefixes can be imported and exported correctly as EVPN IP routes. | `OV`, `SC` |

## Spine Play

| Line | Play or Task | Reason | Source |
| --- | --- | --- | --- |
| 218 | Configure spines (VXLAN EVPN) | Applies the routed underlay and EVPN control-plane configuration to the transit spines. | `OV`, `SC` |
| 227 | Enable routing | Enables EOS routing services and local exec authorization on each spine. | `SC` |
| 234 | Configure local cg user | Creates the same local administrative account on the spines for direct access and automation. | Local playbook |
| 240 | Configure Loopback | Creates Loopback0 as the stable router-id and EVPN peering source for each spine. | `SC` |
| 249 | Configure spine underlay interfaces | Builds the routed point-to-point downlinks from each spine to the leaves with jumbo MTU for the fabric. | `SC` |
| 263 | Configure BGP underlay base for spines | Instantiates the spine BGP process and advertises the spine loopback into the underlay. | `SC` |
| 273 | Configure spine underlay BGP neighbors | Adds each leaf as a directly connected eBGP underlay neighbor using the per-leaf AS from inventory. | `SC` |
| 281 | Activate spine underlay neighbors for IPv4 | Enables each underlay leaf neighbor under the IPv4 address family so the routed fabric exchanges loopback reachability. | `SC` |
| 294 | Configure EVPN base | Sets the spine router-id and preserves EVPN next-hop values so remote leaves keep the originating VTEP as next hop. | `OV`, `SC` |
| 301 | Configure EVPN peer group on spines | Creates the reusable EVPN peer-group used for loopback-based overlay sessions toward all leaves. | `SC` |
| 310 | Configure spine EVPN neighbors (spines -> leafs only) | Adds each leaf Loopback0 as an EVPN neighbor and sets the correct per-leaf remote AS. | `SC` |
| 318 | Activate spine EVPN neighbors | Enables the EVPN peer-group under the EVPN address family so the spine can relay EVPN routes between leaves. | `OV`, `SC` |

## Review Notes

- The playbook is aligned to a simple eBGP underlay plus eBGP EVPN overlay design, not an iBGP EVPN design.
- The leafs are the only VTEPs in this topology; the spines do not own tenant VNIs or act as gateways.
- The configuration uses explicit EVPN route-target values derived from the tenant and VLAN data in the playbook.
- The current structure is adequate for a small lab and matches the rendered configs in `rendered-configs/`, but it is not yet a fully declarative lifecycle playbook because stale objects are not removed when inventory data shrinks.

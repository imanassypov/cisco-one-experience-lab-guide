# DC fabric verification - Pseudoco-DC1

Generated 2026-11-12T18:00:52Z by `08_verify_fabric.yml`.

Declared intent comes from `group_vars/all/dc_switches.yml`,
`group_vars/all/dc_vrf_lite.yml`,
`host_vars/Pseudoco-DC1/vrfs.nac.yaml` and
`host_vars/Pseudoco-DC1/networks.nac.yaml`.

Observed state has two sources. Every table except the last one is read from
the Nexus Dashboard API, needs no device SSH, and is therefore valid even when
the switch management addresses are unreachable from here. The BGP session
table is read off the border leaf itself over SSH, because the controller has
no API that reports routing protocol state. When that switch cannot be
reached, those rows read `not determined` and are excluded from the count
below rather than being reported as sessions that are down.

**20 of 20 check(s) match intent.**

## Switches

| Switch | Mgmt IP | Serial | Role declared | Role observed | Mode | Config compliance | Match |
|---|---|---|---|---|---|---|---|
| DC-Leaf1 | 198.18.128.101 | <serial> | leaf | leaf | Normal | In-Sync | yes |
| DC-Leaf2 | 198.18.128.102 | <serial> | leaf | leaf | Normal | In-Sync | yes |
| DC-SPINE-1 | 198.18.128.11 | <serial> | spine | spine | Normal | In-Sync | yes |
| DC-SPINE-2 | 198.18.128.12 | <serial> | spine | spine | Normal | In-Sync | yes |
| DC-Service-Leaf | 198.18.128.13 | <serial> | border | border | Normal | In-Sync | yes |

A switch fails this table when Nexus Dashboard has no inventory record at its
management IP, or when the role it holds is not the role declared for it.

## VRFs

| VRF | ID declared | ID observed | Switches declared | Switches missing | Not attached | Attachment states | Match |
|---|---|---|---|---|---|---|---|
| MAIN | 50000 | 50000 | DC-Leaf1, DC-Leaf2, DC-Service-Leaf | none | none | DEPLOYED, DEPLOYED, DEPLOYED | yes |
| PROD | 50001 | 50001 | DC-Leaf1, DC-Leaf2, DC-Service-Leaf | none | none | DEPLOYED, DEPLOYED, DEPLOYED | yes |
| IOT | 50002 | 50002 | DC-Leaf1, DC-Leaf2, DC-Service-Leaf | none | none | DEPLOYED, DEPLOYED, DEPLOYED | yes |

A VRF must exist with the declared VNI and be attached, in `DEPLOYED`, to
every switch in its `vrf_attach_group`. Nexus Dashboard lists every switch
that could carry the VRF, marking the ones it has not attached with
`isLanAttached: false` and `lanAttachState: NA`. `NA` is not a fault and not
a pending state, so those switches are excluded from "Attachment states" and
named under "Not attached" instead. A switch that
*was* declared and is not among the attached ones is the real fault, and
shows up under "Switches missing". An attachment stuck in `PENDING` means the
intent was created but never pushed - re-run `07_recalculate_and_deploy.yml`.

## Networks

| Network | VRF declared | VRF observed | Gateway declared | Gateway observed | Attachments declared | Attachments missing | Not attached | Attachment states | Match |
|---|---|---|---|---|---|---|---|---|---|
| MainNetwork1 | MAIN | MAIN | 10.10.252.1/24 | 10.10.252.1/24 | DC-Leaf1 Port-channel3, DC-Leaf2 Port-channel3 | none | DC-Service-Leaf | DEPLOYED, DEPLOYED | yes |
| ProdNetwork1 | PROD | PROD | 10.101.252.1/24 | 10.101.252.1/24 | DC-Leaf1 Port-channel4, DC-Leaf2 Port-channel4 | none | DC-Service-Leaf | DEPLOYED, DEPLOYED | yes |
| IOTNetwork1 | IOT | IOT | 10.102.252.1/24 | 10.102.252.1/24 | DC-Leaf1 Port-channel5, DC-Leaf2 Port-channel5 | none | DC-Service-Leaf | DEPLOYED, DEPLOYED | yes |

A network must exist in the declared VRF with the declared anycast gateway,
and each declared port-channel must be attached **on every switch it was
declared on**, with every attachment in `DEPLOYED`. Attachments are listed as
`switch port-channel` pairs, so a network that reached one leaf but not its
vPC peer shows up here rather than passing. As with the VRFs, the switches
Nexus Dashboard lists but has not attached come back as `NA` and are named
under "Not attached" rather than judged; a declared pair that is missing from
the attached set is what fails, under "Attachments missing".

## VRF-Lite extensions

| Extension | Declared | Attached | State | Interface seen | Dot1q seen | Neighbour seen | Match |
|---|---|---|---|---|---|---|---|
| VRF-Lite PROD on DC-Service-Leaf | Ethernet1/8 dot1q 2 192.168.252.5/30 -> 192.168.252.6 peer PROD MTU 1500 | yes | deployed | yes | yes | yes | yes |
| VRF-Lite IOT on DC-Service-Leaf | Ethernet1/8 dot1q 3 192.168.252.9/30 -> 192.168.252.10 peer IOT MTU 1500 | yes | deployed | yes | yes | yes | yes |
| VRF-Lite MAIN on DC-Service-Leaf | Ethernet1/8 dot1q 4 192.168.252.13/30 -> 192.168.252.14 peer MAIN MTU 1500 | yes | deployed | yes | yes | yes | yes |

Each extension must be attached to the border leaf, in `DEPLOYED`, and its
`extensionValues` must carry the declared interface, dot1q tag and neighbour
address. Those three columns read named fields out of that document, which
Nexus Dashboard returns as JSON nested inside a JSON string.

A whole row reading `NO` with state `absent` almost always means stage 02 ran
after stage 05: the create role uses `state: replaced`, and the data model has
no key for an extension, so a create run clears `extensionValues` even though
it keeps the border leaf's attachment. Re-run `06_vrf_lite.yml`. If it then
fails with `No VRF LITE capable interfaces found on this switch`, run
`05_recalculate_and_deploy.yml` first, then retry stage 06.
A row that is attached and `DEPLOYED` but has a `NO` in one
of the seen columns means the extension exists with different values than
`dc_vrf_lite.yml` declares - reconcile those values before running stage 06
again.

## VRF-Lite sub-interfaces on the switch

| Sub-interface | Declared | Oper | Dot1q | Address | MTU | Match |
|---|---|---|---|---|---|---|
| Sub-interface Ethernet1/8.2 on DC-Service-Leaf | dot1q 2 192.168.252.5/30 mtu 1500 | up | yes | yes | yes | yes |
| Sub-interface Ethernet1/8.3 on DC-Service-Leaf | dot1q 3 192.168.252.9/30 mtu 1500 | up | yes | yes | yes | yes |
| Sub-interface Ethernet1/8.4 on DC-Service-Leaf | dot1q 4 192.168.252.13/30 mtu 1500 | up | yes | yes | yes | yes |

The section above is what the controller believes; this one is what the switch
actually runs. `show interface` is parsed with pyATS/Genie, so the dot1q tag,
address and MTU are compared as named fields rather than matched out of screen
text.

The two can disagree, and the disagreement is the finding. Extensions
`DEPLOYED` on the controller with no sub-interface here means the deploy did
not reach the switch - run `07_recalculate_and_deploy.yml`. A sub-interface
that is present but `down` usually means the far end is not configured to
match; the BGP table below will show the session never leaving Idle.

## VRF-Lite BGP sessions

Read from `DC-Service-Leaf` with
`show bgp sessions vrf <vrf>`, one command per declared extension.

| VRF | Neighbour | Peer ASN observed | State expected | State observed | Match |
|---|---|---|---|---|---|
| PROD | 192.168.252.6 | 65531 | Established | Established | yes |
| IOT | 192.168.252.10 | 65531 | Established | Established | yes |
| MAIN | 192.168.252.14 | 65531 | Established | Established | yes |

This is the only table in this report that describes the fabric rather than
the controller's intent for it. The tables above prove Nexus Dashboard pushed
a sub-interface, a dot1q tag and a neighbour statement; this one asks the
border leaf whether the neighbour answered.

`Established` is the only state that means external connectivity works. `Idle`
usually means the session is not being attempted or is being refused, and
`Active` means the leaf is trying and getting nothing back; both point at the
far end, DC-SITE11-CEDGE8Kv, whose side of these links is pre-built by the pod
and which Nexus Dashboard never writes to. Check that the router's
sub-interface for that dot1q tag carries the address `dc_vrf_lite.yml` names
as `neighbor_ipv4`, that it is up, and that the peer ASN in the table above is
the AS the router really uses. `absent` is a different fault: the switch
answered and has no session to that neighbour in that VRF at all, so the
extension never reached the running configuration - re-run
`06_vrf_lite.yml` and then `07_recalculate_and_deploy.yml`.

## Failures

None.

## Not covered by this report

- **The edge router's side of the VRF-Lite links.** The extensions on
  DC-Service-Leaf are built by `06_vrf_lite.yml` and checked above, and the BGP
  session table says whether the router answered, but nothing here is read from
  DC-SITE11-CEDGE8Kv itself. The External fabric is in Monitor Mode, so Nexus
  Dashboard never writes to that router and has nothing to report about it. Its
  side of the three links is pre-built by the pod, which is why the addresses in
  `dc_vrf_lite.yml` are pinned rather than allocated.
- **What the sessions carry.** `Established` means the two routers are talking.
  It does not say how many prefixes crossed, or that the ones that did are the
  ones you wanted. `show bgp ipv4 unicast neighbors <neighbour> vrf <vrf>` on
  the border leaf reports the prefixes sent and received on that session.
- **The fabric underlay and the EVPN overlay.** The BGP session table covers
  only the three VRF-Lite eBGP sessions to the edge router. The iBGP EVPN
  sessions to the spines are not checked here.
- **Data plane.** Nothing here sends a packet between two workloads, so it does
  not prove a server on MainNetwork1 can reach one on ProdNetwork1.

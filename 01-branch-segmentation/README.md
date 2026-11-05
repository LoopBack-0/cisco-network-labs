# Project 1 – [Branch Site Segmentation]

## Scenario

A manufacturing branch runs everything on one flat network: office PCs, VoIP phones, printers, warehouse handheld scanners and servers. A malware infection on one PC reached the scanners and halted shipping for a day. Management wants the network segmented by function, with scanners isolated, centralized DHCP, and a design that can grow.

## Topology

Phase A - Flat network 
![](branch-seg-phaseA-flat_network.png)

Phase A - Router-on-a-Stick/Segmented
![](branch-seg-phaseA-ROAS.png)

Phase B - Layer 3 Inter-VLAN routing
![](branch-seg-phaseB-final.png)

| Device   | Model     | Role                                     |
| -------- | --------- | ---------------------------------------- |
| R1       | ISP4331   | WAN edge (router-on-a-stick in Phase A)  |
| CORE-SW1 | 3650-24PS | Layer 3 core, default gateways (Phase B) |
| ACC-SW1  | 2960-24TT | Office Access Switch                     |
| ACC-SW2  | 2960-24TT | Warehouse Access Switch                  |
| SRV1     | Server-PT | DHCP Central Server                      |

## Addressing Plan & VLAN Plan

| VLAN | Name     | Subnet         | Gateway    | Notes                              |
| ---- | -------- | -------------- | ---------- | ---------------------------------- |
| 10   | USERS    | 10.10.10.0/24  | 10.10.10.1 | Office PCs (DHCP)                  |
| 20   | VOICE    | 10.10.20.0/24  | 10.10.20.1 | IP phones (DHCP)                   |
| 30   | PRINTERS | 10.10.30.0/24  | 10.10.30.1 | Static                             |
| 40   | SCANNERS | 10.10.40.0/24  | 10.10.40.1 | Warehouse handhelds (DHCP)         |
| 50   | SERVERS  | 10.10.50.0/24  | 10.10.50.1 | static ; DHCP server = 10.10.50.10 |
| 99   | MGMT     | 10.10.99.0/24  | 10.10.99.1 | static ; Network Infra             |
| 666  | PARKING  | -------------  | ---------- | Unused ports, shut down            |
| 999  | NATIVE   | -------------- | ---------- | Unsued native VLAN on trucks       |

| Device             | Management IP |
| ------------------ | ------------- |
| CORE-SW1 (Phase B) | 10.10.99.2    |
| ACC-SW1            | 10.10.99.11   |
| ACC-SW2            | 10.10.99.12   |
**Phase B routed link:** R1 G0/0/0 = 10.10.255.1/30, CORE-SW1 G1/0/24 = 10.10.255.2/30
**Phase B Loop Back 0**: L0 = 8.8.8.8/32 to simulate External address (Internet)

## Port Plan

### ACC-SW1

| Ports    | Use                             | VLAN                |
| -------- | ------------------------------- | ------------------- |
| Fa0/1–4  | PCs + phones                    | Access 10, voice 20 |
| Fa0/5–6  | Printers                        | 30                  |
| Fa0/7–8  | Servers                         | 50                  |
| Fa0/9–24 | Unused                          | 666, shutdown       |
| G0/1     | Uplink to R1 (A) / CORE-SW1 (B) | Trunk               |
| G0/2     | Link to ACC-SW2 (Phase A only)  | Trunk               |

### ACC-SW2

|Ports|Use|VLAN|
|---|---|---|
|Fa0/1–4|PCs + phones|Access 10, voice 20|
|Fa0/5–10|Scanners (use PCs labeled "Scanner")|40|
|Fa0/11–24|Unused|666, shutdown|
|G0/1|Uplink to ACC-SW1 (A) / CORE-SW1 (B)|Trunk|
## Design Decisions

- **Baseline configs per device:** Every device starts from the same hardened template (hostname, banner, SSH only, etc). That way, when something breaks, I can rule out basic misconfiguration and roll back to a known-good state.
- **Separated devices based on function**: Created VLANS for each device function and allow segmentation of broadcast domains. This will help prevent compromise in one broadcast domain reaching another at layer 2.
- **Shutdown and "PARK" Unused ports**: By shutting them down and placing into an unused VLAN I prevented anyone from finding a random port and establishing connection to the switch or other VLAN further restricting the attack surface
- **Set native VLAN to unused VLAN**: This keeps untagged traffic away from user traffic and prevents VLAN hopping and double tagging.
- **Configured Central DHCP server**: This will allow easy management of DHCP leases and scopes as it will all be in one location instead of having to configure them across each VLAN in cisco IOS.
- **Used ACL is further isolate the scanners**: VLAN break up broadcast domains however scanners may still be able to send traffic to all other devices via R1. An ACL will limit what traffic is forwarded across VLANs. Business requirement states the scanner need only to communicate with servers.
- **Migrated Inter-VLAN routing to Core switch**: This eliminates the bottleneck from ACC-SW1 to R1 and allows R1 to act as the WAN edge only allowing for easier and more manageable expansion in the future. 

## Implementation Summary

1.) Created and named VLANS on all switches
2.) Placed Each port in the correct VLAN based on device types 
3.) Configured access port with Port Fast, Voice VLAN and Port Security where needed.
4.) Placed all unused ports in PARKING VLAN with the status of "shut down" 
5.) Configured trunks with native VLAN 999, pruned allowed lists and DTP disabled
6.) Configured ROAS on R1 and used  *ip helper-address* to set the DHCP relay for each VLAN
7.) Configured SRV1 with DHCP pools for USERS, VOICE and SCANNERS
8.) Configured ACL to restrict scanners to only servers 
9.) Returned Interface G0/2 on ACC-SW1 back to default and placed in VLAN 666 with shutdown status
10.) Configured VLAN on CORE-SW1 and Trunk links to access layer with VLANs pruned and DTP disabled; Native 999
11.) Configured routed link to R1 on CORE-SW1 with next hop 10.10.255.1(R1)
12.) Removed all sub-interfaces from R1 
13.) Configured return route back to the LAN 


Full configs: [configs/](configs/)

## Verification

### VLANS and Port Assignments
==ACC-SW1==
```
ACC-SW1#sh vlan br

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                           active    Gig0/1, Gig0/2
10   USERS                             active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
20   VOICE                             active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
30   PRINTERS                          active    Fa0/5, Fa0/6
40   SCANNERS                          active
50   SERVERS                           active    Fa0/7, Fa0/8
99   MGMT                              active
666  PARKING                           active    Fa0/9, Fa0/10, Fa0/11, Fa0/12
                                                 Fa0/13, Fa0/14, Fa0/15, Fa0/16
                                                 Fa0/17, Fa0/18, Fa0/19, Fa0/20
										      Fa0/21, Fa0/22, Fa0/23, Fa0/24
999 NATIVE                             active
```

==ACC-SW2==
```
ACC-SW2#sh vlan br

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig0/1, Gig0/2
10   USERS                            active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
20   VOICE                            active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
30   PRINTERS                         active
40   SCANNERS                         active    Fa0/5, Fa0/6, Fa0/7, Fa0/8
                                                Fa0/9, Fa0/10
50   SERVERS                          active
99   MGMT                             active
666  PARKING                          active    Fa0/11, Fa0/12, Fa0/13, Fa0/14
                                                Fa0/15, Fa0/16, Fa0/17, Fa0/18
                                                Fa0/19, Fa0/20, Fa0/21, Fa0/22
                                                Fa0/23, Fa0/24
999  NATIVE                           active
```

### Trunk Ports and Allowed Switch-to-Switch VLAN traffic 

==ACC-SW1==   NOTE: G0/1 Trunk does not show up until the sub interfaces on R1 are up
```
ACC-SW1(config-if)#do sh int tr

Port      Mode       Encapsulation    Status     Native vlan
Gig0/1    on         802.1q           trunking   999
Gig0/2    on         802.1q           trunking   999

Port      Vlans allowed on trunk
Gig0/1    10,20,30,40,50,99,999
Gig0/2    10,20,30,40,50,99,999

Port      Vlans allowed and active in management domain
Gig0/1    10,20,30,40,50,99,999
Gig0/2    10,20,30,40,50,99,999

Port      Vlans in spanning tree forwarding state and not pruned
Gig0/1    10,20,30,40,50,99,999
Gig0/2    10,20,30,40,50,99,999
```

==ACC-SW2==
```
ACC-SW2(config-if)#do sh int tr

Port      Mode       Encapsulation    Status       Native vlan
Gig0/1    on         802.1q           trunking     999

Port      Vlans allowed on trunk
Gig0/1    10,20,30,40,50,99,999

Port      Vlans allowed and active in management domain
Gig0/1    10,20,30,40,50,99,999

Port      Vlans in spanning tree forwarding state and not pruned
Gig0/1    10,20,30,40,50,99,999
```

### Sub interfaces configured For ROAS on R1 (Phase A)

```
R1#sh ip int br

Interface                 IP-Address   OK?   Method    Status  Protocol
GigabitEthernet0/0/0      unassigned   YES   unset     up      up
GigabitEthernet0/0/0.10   10.10.10.1   YES   manual    up      up
GigabitEthernet0/0/0.20   10.10.20.1   YES   manual    up      up
GigabitEthernet0/0/0.30   10.10.30.1   YES   manual    up      up
GigabitEthernet0/0/0.40   10.10.40.1   YES   manual    up      up
GigabitEthernet0/0/0.50   10.10.50.1   YES   manual    up      up
GigabitEthernet0/0/0.99   10.10.99.1   YES   manual    up      up
GigabitEthernet0/0/0.999  unassigned   YES   unset     up      up
```

### DHCP Central server Scopes
==SERV1==
![](branch-Seg-dhcp-scopes.png)

### ACL restricts Scanners to only servers 

==SCANNER-IN acl==
![](branch-seg-acl.png)

==Scanner cant ping PC1 but can ping Serv1==
![](branch-seg-acl-proof.png)

### Core-SW1 routes 

![](branch-seg-phaseB-CoreSW-routes.png)

### CORE-SW1 acts as gateway for VLAN
==PC2 ping SERV1==
![](branch-seg-core-gateway.png)
## Troubleshooting Log

| Symptom                                                                                                                                                          | Root cause                                                 | How I found it                                                                                                                                                                                                                                               | Fix                                                                                      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Blocking GigabitEthernet0/1 on VLAN0001; NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet0/1 (1), with ACC-SW1 GigabitEthernet0/2 (999). | ACC-SW2 was still in the default native vlan 1             | Ran show interface trunk on both access switches and noticed I had missed setting the native vlan of 999 on ACC-SW2                                                                                                                                          | Entered interface config mode on ACC-SW2 g0/1 and set the native VLAN to 999             |
| DHCP request failed on all devices                                                                                                                               | No static IP was configured on the DHCP server interface   | Double check all configs related to DHCP both on R1 (ip helper-address) and DHCP scopes and found all to be correct. It was then I noticed I never set the server to IP address 10.10.50.10                                                                  | Configured IP address and gateway on SRV1 host                                           |
| Scanner was still not pulling an IP address. DHCP request failed.                                                                                                | IP helper-address was set with the wrong IP address (Typo) | At first I thought maybe the ACL had blocked the scanners from reaching DHCP and as I was looking at the running config I noticed sub interface g0/0/0.40 was showing the wrong ip helper address                                                            | Corrected the typo on g0/0/0.40                                                          |
| Created a loopback on R1 to simulate "Internet" to test routing between R1 and CORE-SW1 and the pings from PC2 were failing                                      | No return route was configured for 10.10.0.0/16 (LAN)      | Ran tracert from PC2 and confirmed it hit the gateway on CORE-SW1. From CORE-SW1 G1/0/24 interface ping were successful how ever due to there being no NAT configured traffic was leaving core switch as 10.10.10.3 (PC2) and R1 has no route to that subnet | Created a route to 10.10.0.0/16 on R1 to enable routing back to the LAN for all Subnets. |

## Lessons Learned

-**A loopback on R1 wasn't reachable from PC2.** I added a loopback on R1 to test reachability from PC2, and the pings failed. I expected them to succeed. No route to the loopback subnet on the return path. In production, that kind of unverified change could cause an outage.

-**Simple mistakes will take down connectivity**: Verifying configurations with show commands after each task seems redundant. If I had been doing that along the way I would not have to trace back steps to figure out where a typo was or what commands were incorrectly configured. Moving forward I will use show commands more often and after each task and reduce overall troubleshooting/back tracking times. 

## Skills Demonstrated
VLANs, trunking, port security, L3 switching, DHCP relay, ACLs


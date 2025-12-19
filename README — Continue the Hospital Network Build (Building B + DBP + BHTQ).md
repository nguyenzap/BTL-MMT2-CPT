## 0) What this README is for

This guide continues the work already completed for **HCMC core + Building A**. Your friend will:

1. Rebuild **Building B** from scratch:

   * 5 floor access switches (L2)
   * 1 distribution switch **B-DIST (Catalyst 3650)** doing inter-VLAN routing (SVIs)
   * Uplink B-DIST to **HCMC-CORE1** using a **routed port**
   * OSPF Area 20 (stub) for Building B
   * Guest VLAN isolation (Internet only)

2. Build **DBP site** and **BHTQ site** using the same design style:

   * Replace their “core” switches with **Catalyst 3650 multilayer**
   * Use routed uplinks + OSPF over WAN to HCMC
   * Unique site addressing blocks (no overlap)

This README assumes HCMC-CORE1 and ASA are already in place and working (you already reached that point).

---

## 1) Design summary (so you understand the goal)

### Roles

* **Floor switches (Access):** Layer-2 only, VLANs, trunks to B-DIST.
* **B-DIST (Distribution):** Layer-3 gateway for Building B VLANs (SVIs), ACL for guest, OSPF with CORE.
* **HCMC-CORE1 (Core):** Routed backbone only, OSPF Area 0 hub, default route to ASA, advertises default route.
* **ASA:** Firewall/NAT/DMZ.
* **HCMC-WAN router:** Connects HCMC to DBP and BHTQ over serial links (WAN).
* **DBP-CORE / BHTQ-CORE:** Use 3650 multilayer switch (same pattern as B-DIST/A-DIST) as the site gateway switch.

### Addressing plan (must not overlap)

* **Building A (already done):** `10.10.0.0/16`
* **Building B (you will build now):** `10.20.0.0/16`
* **DBP site:** `10.30.0.0/16`
* **BHTQ site:** `10.40.0.0/16`
* **Transit/routed links:** `10.255.0.0/16` (use /30 per point-to-point)
* **Server farm VLAN 100:** `10.50.100.0/24` (gateway on CORE1)
* **DMZ:** should be outside the building summaries (best design uses `10.60.100.0/27` on ASA)

### OSPF areas

* **Area 0:** CORE backbone, links to A-DIST, B-DIST, WAN router, CORE2 (optional)
* **Area 10 (stub):** Building A behind A-DIST
* **Area 20 (stub):** Building B behind B-DIST
* DBP/BHTQ can be in area 0 (simple lab), or separate areas later if you want.

---

## 2) Hardware preparation (Packet Tracer physical changes)

### 2.1 Floor switches (each floor)

For each floor switch in Building B:

1. Place **Switch-PT-Empty**
2. Go to **Physical** tab
3. Insert **6× `PT-SWITCH-NM-1CGE`** modules
   This gives you enough **GigabitEthernet ports** to match the port plan.

### 2.2 PCs and laptops (each floor)

For each PC and laptop:

1. Open device → **Physical** tab
2. Power off if needed
3. Remove current LAN module
4. Insert **`PT-HOST-NM-1CGE`**
5. Power on

### 2.3 Add floor endpoints

Each floor should have:

* 1 PC
* 1 Laptop
* 1 IP Phone
* 1 Camera
* 1 Guest Access Point

### 2.4 Cabling order (important)

On each **floor access switch**, plug devices into these ports (exact order):

* **Gig0/1** → uplink trunk to **B-DIST**
* **Gig1/1** → PC
* **Gig2/1** → Laptop
* **Gig3/1** → Phone
* **Gig4/1** → Camera
* **Gig5/1** → Guest Access Point

> If your switch shows different numbering, use the same *concept*: Gi0/1 uplink, remaining ports for endpoints.

---

## 3) Naming convention (must be consistent)

Use these hostnames:

### Building B floor access switches

* `B-F1-ACC`
* `B-F2-ACC`
* `B-F3-ACC`
* `B-F4-ACC`
* `B-F5-ACC`

### Building B distribution

* `B-DIST`

### DBP site

* `DBP-CORE` (3650 multilayer)
* Access switches can follow similar style, e.g. `DBP-ACC1`

### BHTQ site

* `BHTQ-CORE` (3650 multilayer)
* Access switches e.g. `BHTQ-ACC1`

> Easiest method: copy/paste a configured floor switch device and only change hostname and VLAN IDs.

---

## 4) VLAN plan for Building B

Building B uses **building-wide shared VLANs**, but different floor department VLANs.

### VLANs for Building B

* Floor 1: VLAN **21** `B_F1_ADMIN`
* Floor 2: VLAN **22** `B_F2_CLINICAL`
* Floor 3: VLAN **23** `B_F3_LAB`
* Floor 4: VLAN **24** `B_F4_RADIOLOGY`
* Floor 5: VLAN **25** `B_F5_IT_MGMT`

Shared across Building B:

* VLAN **26** `B_VOIP`
* VLAN **27** `B_CAMERA`
* VLAN **28** `B_GUEST`

---

## 5) IP plan for Building B (SVIs on B-DIST)

Use /29 per VLAN (small lab, enough for gateway + a few devices):

* VLAN 21: `10.20.21.0/29` gateway `10.20.21.1`
* VLAN 22: `10.20.22.0/29` gateway `10.20.22.1`
* VLAN 23: `10.20.23.0/29` gateway `10.20.23.1`
* VLAN 24: `10.20.24.0/29` gateway `10.20.24.1`
* VLAN 25: `10.20.25.0/29` gateway `10.20.25.1`
* VLAN 26: `10.20.26.0/29` gateway `10.20.26.1`
* VLAN 27: `10.20.27.0/29` gateway `10.20.27.1`
* VLAN 28: `10.20.28.0/29` gateway `10.20.28.1`

DHCP server (central) lives on **HCMC-CORE1**, so B-DIST will use:

* `ip helper-address 10.255.255.1` (CORE1 loopback)

---

## 6) Build floor switches (Building B) — configs

### 6.1 Template for B-F1-ACC (Floor 1 Access Switch)

> Repeat this template for floors 2–5 with correct VLAN ID + names + allowed VLAN list.

```cisco
enable
conf t
hostname B-F1-ACC

! VLANs
vlan 21
 name B_F1_ADMIN
vlan 26
 name B_VOIP
vlan 27
 name B_CAMERA
vlan 28
 name B_GUEST

! Uplink trunk to B-DIST
interface gig0/1
 description UPLINK_TO_B-DIST
 switchport mode trunk
 switchport trunk allowed vlan 21,26,27,28
 no shutdown

! Admin PC
interface gig1/1
 description B_F1_ADMIN_PC
 switchport mode access
 switchport access vlan 21
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Laptop
interface gig2/1
 description B_F1_ADMIN_LAPTOP
 switchport mode access
 switchport access vlan 21
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! IP Phone (voice VLAN)
interface gig3/1
 description B_F1_PHONE
 switchport mode access
 switchport voice vlan 26
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Camera
interface gig4/1
 description B_F1_CAMERA
 switchport mode access
 switchport access vlan 27
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Guest AP
interface gig5/1
 description B_F1_GUEST_AP
 switchport mode access
 switchport access vlan 28
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

end
wr mem
```

### 6.2 Floor 2–5 configs (only VLAN changes)

#### Floor 2

* Department VLAN = **22**
* Allowed VLANs on trunk: `22,26,27,28`

Change hostname to `B-F2-ACC`, VLAN 22 name `B_F2_CLINICAL`, and access VLAN on PC/laptop to 22.

#### Floor 3

* Department VLAN = **23**
* Allowed VLANs: `23,26,27,28`

#### Floor 4

* Department VLAN = **24**
* Allowed VLANs: `24,26,27,28`

#### Floor 5

* Department VLAN = **25**
* Allowed VLANs: `25,26,27,28`

---

## 7) Build B-DIST (3650 multilayer) — full config

### 7.1 Hardware and cabling

* Replace B-DIST with a **3650-24PS multilayer switch** (same model as A-DIST)
* Downlinks to floors:

  * `Gi1/0/1` → Floor 1 uplink (B-F1-ACC Gi0/1)
  * `Gi1/0/2` → Floor 2 uplink
  * `Gi1/0/3` → Floor 3 uplink
  * `Gi1/0/4` → Floor 4 uplink
  * `Gi1/0/5` → Floor 5 uplink
* Uplink to CORE1:

  * `Gi1/0/6` → CORE1 `Gi1/0/2` (this matches the core port mapping)

### 7.2 B-DIST config (copy/paste)

```cisco
enable
conf t
hostname B-DIST

ip routing

! VLANs
vlan 21
 name B_F1_ADMIN
vlan 22
 name B_F2_CLINICAL
vlan 23
 name B_F3_LAB
vlan 24
 name B_F4_RADIOLOGY
vlan 25
 name B_F5_IT_MGMT
vlan 26
 name B_VOIP
vlan 27
 name B_CAMERA
vlan 28
 name B_GUEST

! Trunks down to floors
interface gig1/0/1
 description TRUNK_TO_B-F1-ACC
 switchport mode trunk
 switchport trunk allowed vlan 21,26,27,28
 no shutdown

interface gig1/0/2
 description TRUNK_TO_B-F2-ACC
 switchport mode trunk
 switchport trunk allowed vlan 22,26,27,28
 no shutdown

interface gig1/0/3
 description TRUNK_TO_B-F3-ACC
 switchport mode trunk
 switchport trunk allowed vlan 23,26,27,28
 no shutdown

interface gig1/0/4
 description TRUNK_TO_B-F4-ACC
 switchport mode trunk
 switchport trunk allowed vlan 24,26,27,28
 no shutdown

interface gig1/0/5
 description TRUNK_TO_B-F5-ACC
 switchport mode trunk
 switchport trunk allowed vlan 25,26,27,28
 no shutdown

! Uplink to CORE1 is routed (no switchport)
interface gig1/0/6
 description L3_UPLINK_TO_HCMC-CORE1_Gi1/0/2
 no switchport
 ip address 10.255.0.6 255.255.255.252
 no shutdown

! SVIs (Building B)
interface vlan 21
 description GW_B_F1_ADMIN
 ip address 10.20.21.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 22
 description GW_B_F2_CLINICAL
 ip address 10.20.22.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 23
 description GW_B_F3_LAB
 ip address 10.20.23.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 24
 description GW_B_F4_RADIOLOGY
 ip address 10.20.24.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 25
 description GW_B_F5_IT_MGMT
 ip address 10.20.25.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 26
 description GW_B_VOIP
 ip address 10.20.26.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 27
 description GW_B_CAMERA
 ip address 10.20.27.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

interface vlan 28
 description GW_B_GUEST
 ip address 10.20.28.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

! Guest ACL (internet only)
ip access-list extended B_GUEST_IN
 deny ip any 10.0.0.0 0.255.255.255
 deny ip any 172.16.0.0 0.15.255.255
 deny ip any 192.168.0.0 0.0.255.255
 permit ip any any
exit

interface vlan 28
 ip access-group B_GUEST_IN in

! OSPF: Area 0 uplink, Area 20 stub for Building B
router ospf 1
 router-id 10.255.0.6
 passive-interface default
 no passive-interface gig1/0/6

 network 10.255.0.4 0.0.0.3 area 0
 network 10.20.0.0 0.0.255.255 area 20

 area 20 stub
 area 20 range 10.20.0.0 255.255.0.0

end
wr mem
```

### 7.3 CORE1 side (Building B link)

CORE1 already uses:

* CORE1 `Gi1/0/2` = `10.255.0.5/30`

So after B-DIST comes up, OSPF neighbor should form automatically.

Verify on CORE1:

```cisco
show ip ospf neighbor
show ip route ospf
```

---

## 8) DHCP pools on HCMC-CORE1 for Building B

DHCP is centralized on CORE1. You can add pools anytime (even before devices exist).

### Building B DHCP pools (VLAN 21–28)

```cisco
enable
conf t

service dhcp

ip dhcp excluded-address 10.20.21.1 10.20.21.2
ip dhcp excluded-address 10.20.22.1 10.20.22.2
ip dhcp excluded-address 10.20.23.1 10.20.23.2
ip dhcp excluded-address 10.20.24.1 10.20.24.2
ip dhcp excluded-address 10.20.25.1 10.20.25.2
ip dhcp excluded-address 10.20.26.1 10.20.26.2
ip dhcp excluded-address 10.20.27.1 10.20.27.2
ip dhcp excluded-address 10.20.28.1 10.20.28.2

ip dhcp pool B_VLAN21_ADMIN
 network 10.20.21.0 255.255.255.248
 default-router 10.20.21.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN22_CLINICAL
 network 10.20.22.0 255.255.255.248
 default-router 10.20.22.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN23_LAB
 network 10.20.23.0 255.255.255.248
 default-router 10.20.23.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN24_RADIOLOGY
 network 10.20.24.0 255.255.255.248
 default-router 10.20.24.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN25_ITMGMT
 network 10.20.25.0 255.255.255.248
 default-router 10.20.25.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN26_VOIP
 network 10.20.26.0 255.255.255.248
 default-router 10.20.26.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN27_CAMERA
 network 10.20.27.0 255.255.255.248
 default-router 10.20.27.1
 dns-server 8.8.8.8

ip dhcp pool B_VLAN28_GUEST
 network 10.20.28.0 255.255.255.248
 default-router 10.20.28.1
 dns-server 8.8.8.8

end
wr mem
```

Verify:

```cisco
show ip dhcp binding
show ip dhcp pool
```

---

## 9) Testing checklist (Building B)

### Layer 2 checks

On B-DIST:

```cisco
show interfaces trunk
show vlan brief
```

### Layer 3 checks

On B-DIST:

```cisco
show ip interface brief
show ip route
show ip ospf neighbor
```

### Client tests

On a Building B PC:

* Should get DHCP
* Ping gateway (e.g., `10.20.21.1`)
* Ping server farm gateway (e.g., `10.50.100.1`)
* Ping DMZ web (if DMZ is fixed and not overlapping summaries)
* Guest VLAN should NOT ping internal 10.x networks, but should reach internet

---

# PART B — DBP and BHTQ Sites (from scratch)

## 10) Hardware changes (aux sites)

### 10.1 Change “core” hardware

Replace:

* `DBP-CORE` with **Catalyst 3650 multilayer**
* `BHTQ-CORE` with **Catalyst 3650 multilayer**

This is the same reason you changed A-DIST/B-DIST: you want L3 SVIs + OSPF capability.

### 10.2 Change access switches and endpoints like floors

If the aux site has access switches:

* Use **Switch-PT-Empty + 6× PT-SWITCH-NM-1CGE**
* Upgrade PC/Laptop NIC modules to **PT-HOST-NM-1CGE**
* Add camera and AP similarly

---

## 11) WAN routing approach (recommended for this lab)

Keep it simple:

* Run **OSPF** between:

  * HCMC-CORE1 ↔ HCMC-WAN (already cabled on CORE1 Gi1/0/3)
  * HCMC-WAN ↔ DBP router (serial)
  * HCMC-WAN ↔ BHTQ router (serial)
* DBP-CORE and BHTQ-CORE can either:

  * run OSPF with their local site router, OR
  * the site router can have static route pointing to its local core
    For simplicity: **run OSPF everywhere**.

### Site summaries

* DBP advertises `10.30.0.0/16`
* BHTQ advertises `10.40.0.0/16`

---

## 12) Example DBP site plan

### DBP internal VLANs (example)

You can mirror Building A/B style:

* VLAN 31 DBP_ADMIN: `10.30.31.0/29` GW `10.30.31.1`
* VLAN 36 DBP_VOIP: `10.30.36.0/29` GW `10.30.36.1`
* VLAN 37 DBP_CAMERA: `10.30.37.0/29` GW `10.30.37.1`
* VLAN 38 DBP_GUEST: `10.30.38.0/29` GW `10.30.38.1`

### DBP-CORE uplink (routed) to DBP router

Use a transit /30 in `10.255.0.0/16` that does not conflict with existing ones.
Example:

* DBP-CORE ↔ DBP router: `10.255.1.0/30`

  * DBP router = `10.255.1.1`
  * DBP-CORE = `10.255.1.2`

### DBP OSPF

* DBP router participates in OSPF with WAN router (serial)
* DBP router also participates with DBP-CORE over the /30
* DBP-CORE advertises `10.30.0.0/16` (summarize)

> BHTQ will follow the exact same pattern but using `10.40/16`.

---

## 13) Example DBP-CORE config (template)

> Adjust interfaces and VLANs to match your actual topology.

```cisco
enable
conf t
hostname DBP-CORE
ip routing

! Routed uplink to DBP router
interface gig1/0/1
 description L3_LINK_TO_DBP_ROUTER
 no switchport
 ip address 10.255.1.2 255.255.255.252
 no shutdown

! Example VLANs
vlan 31
 name DBP_ADMIN
vlan 36
 name DBP_VOIP
vlan 37
 name DBP_CAMERA
vlan 38
 name DBP_GUEST

interface vlan 31
 ip address 10.30.31.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown
interface vlan 36
 ip address 10.30.36.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown
interface vlan 37
 ip address 10.30.37.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown
interface vlan 38
 ip address 10.30.38.1 255.255.255.248
 ip helper-address 10.255.255.1
 no shutdown

! Guest ACL (internet only)
ip access-list extended DBP_GUEST_IN
 deny ip any 10.0.0.0 0.255.255.255
 deny ip any 172.16.0.0 0.15.255.255
 deny ip any 192.168.0.0 0.0.255.255
 permit ip any any
exit
interface vlan 38
 ip access-group DBP_GUEST_IN in

! OSPF with DBP router
router ospf 1
 router-id 10.255.1.2
 passive-interface default
 no passive-interface gig1/0/1

 network 10.255.1.0 0.0.0.3 area 0
 network 10.30.0.0 0.0.255.255 area 0

! Summarize DBP routes if DBP router is an ABR (optional)
! (On routers you can summarize; on this lab keep simple)

end
wr mem
```

---

## 14) Example DBP router config (template)

> This router connects to WAN and DBP-CORE.

```cisco
enable
conf t
hostname DBP-RTR

interface gig0/0
 description L3_LINK_TO_DBP-CORE
 ip address 10.255.1.1 255.255.255.252
 no shutdown

! serial to HCMC-WAN router (example)
interface serial0/0/0
 description WAN_TO_HCMC-WAN
 ip address 10.255.2.2 255.255.255.252
 clock rate 64000
 no shutdown

router ospf 1
 router-id 30.30.30.30
 network 10.255.1.0 0.0.0.3 area 0
 network 10.255.2.0 0.0.0.3 area 0
end
wr mem
```

> BHTQ site uses same pattern with:

* core: `10.40/16`
* its own transit /30s, e.g. `10.255.3.0/30` for core↔router, `10.255.4.0/30` for serial.

---

## 15) Notes about “copy/paste devices”

Yes, your friend can copy devices to save time:

* Copy a finished floor switch (e.g., B-F1-ACC) → paste it 4 times → change:

  * hostname
  * department VLAN id (21→22→23→24→25)
  * trunk allowed list
  * interface descriptions

Same for site cores: copy DBP-CORE and change IP blocks and names for BHTQ.

---

## 16) Troubleshooting quick reference

### Trunk not showing / no CDP neighbor

* Check `show int status`
* If `err-disabled`: likely BPDU guard on uplink
* Ensure uplink trunk is not portfast/bpduguard

### VLAN SVI is up/down

* Means VLAN exists but **no active L2 port** is forwarding in that VLAN
* Check trunks allowed VLANs and access ports

### DHCP not working

* Ensure SVI has `ip helper-address 10.255.255.1`
* On CORE1: `show ip dhcp binding`
* Ensure OSPF has reachability to CORE1 loopback

### Guest can reach internal

* Verify ACL applied inbound on guest SVI:

  * `interface vlan 28` → `ip access-group B_GUEST_IN in`

### DMZ unreachable / dropped at DIST

* DMZ must not overlap with a summarized building block.
* Best design: DMZ on `10.60.100.0/27` (ASA), not `10.10.100.0/27`.

---

## 17) Final checklist

Before handing back the file:

* ✅ B-DIST forms OSPF neighbor with CORE1
* ✅ Building B clients get DHCP and can ping gateway
* ✅ Building B can reach Server Farm VLAN 100
* ✅ Guest VLAN is internet-only
* ✅ DBP and BHTQ routes appear on CORE1 via OSPF
* ✅ No overlapping subnets

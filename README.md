# Campus University Network - Complete Configuration Guide
## Step-by-Step Documentation (Tested and Working)

---

## Network Overview

### Network Structure
- **Main Campus** → Main Multilayer Switch → Main Router → VLANs 10, 20, 30
- **Branch Campus** → Branch Multilayer Switch → Branch Router → VLANs 40, 50, 60
- **Connection**: Main Router ↔ Branch Router via Serial links (10.0.0.1 and 10.0.0.2)

### VLAN Assignment

| VLAN | Name | Network | Gateway | Location | Color Code |
|------|------|---------|---------|----------|------------|
| 10 | Building-A | 192.168.10.0/24 | 192.168.10.1 | Main Campus | Green |
| 20 | Building-B | 192.168.20.0/24 | 192.168.20.1 | Main Campus | Orange |
| 30 | Admin | 192.168.30.0/24 | 192.168.30.1 | Main Campus | Tan |
| 40 | Building01 | 192.168.40.0/24 | 192.168.40.1 | Branch Campus | Blue |
| 50 | Building02 | 192.168.50.0/24 | 192.168.50.1 | Branch Campus | Pink |
| 60 | Hosting | 192.168.60.0/24 | 192.168.60.1 | Branch Campus | Yellow |

---

## Configuration Order (CRITICAL - Follow This Sequence!)

1. **Main Multilayer Switch** (Core) - Create all VLANs, configure trunks
2. **Main Router** - Configure subinterfaces for VLANs 10, 20, 30 and DHCP
3. **Main Campus Access Switches** - Configure one building at a time, test each
4. **Branch Multilayer Switch** - Create VLANs 40, 50, 60, configure trunks
5. **Branch Router** - Configure subinterfaces for VLANs 40, 50, 60 and DHCP
6. **Main Router Serial Link** - Connect to Branch Router
7. **Branch Campus Access Switches** - Configure one building at a time, test each

---

## STEP 1: Main Multilayer Switch (Core Switch)

**Purpose**: Central hub for all Main Campus VLANs

```cisco
enable
configure terminal
hostname MainCore

! Create VLANs for Main Campus
vlan 10
name Building-A
exit

vlan 20
name Building-B
exit

vlan 30
name Admin
exit

! Configure trunk ports for access switches
! Note: Multilayer switches need encapsulation set first
interface range fa0/1-24
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan all
no shutdown
exit

interface range gi0/1-2
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan all
no shutdown
exit

end
write memory
```

**Verification**:
```cisco
show vlan brief
show interfaces trunk
```

---

## STEP 2: Main Router Configuration

**Purpose**: Inter-VLAN routing and DHCP for Main Campus (VLANs 10, 20, 30)

### Part A: Subinterfaces (Router on a Stick)

```cisco
enable
configure terminal
hostname MainRouter

! Bring up physical interface
interface gi0/0
no shutdown
exit

! VLAN 10 Subinterface
interface gi0/0.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
no shutdown
exit

! VLAN 20 Subinterface
interface gi0/0.20
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
no shutdown
exit

! VLAN 30 Subinterface
interface gi0/0.30
encapsulation dot1Q 30
ip address 192.168.30.1 255.255.255.0
no shutdown
exit
```

### Part B: DHCP Configuration

```cisco
! DHCP Pool for VLAN 10
ip dhcp pool VLAN10-Pool
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
dns-server 8.8.8.8
exit

! DHCP Pool for VLAN 20
ip dhcp pool VLAN20-Pool
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
dns-server 8.8.8.8
exit

! DHCP Pool for VLAN 30
ip dhcp pool VLAN30-Pool
network 192.168.30.0 255.255.255.0
default-router 192.168.30.1
dns-server 8.8.8.8
exit

! Exclude gateway addresses from DHCP
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.20.1
ip dhcp excluded-address 192.168.30.1

end
write memory
```

**Verification**:
```cisco
show ip interface brief
show ip dhcp pool
```

---

## STEP 3: Main Campus Access Switches

### Important Notes:
- Each building may have **multiple switches** connected to the Main Core
- Switches within the same building use the **same VLAN**
- **ALWAYS identify ports first** before configuring:
  - Which port connects to Core? (Make it a trunk)
  - Which ports connect to PCs? (Make them access ports)

### Green Building (VLAN 10) - Example Configuration

**For each switch in Green Building:**

```cisco
enable
configure terminal
vlan 10
name Building-A
exit

! Trunk port to Main Core (identify the correct port first!)
! Example: if Fa0/2 connects to Core
interface fa0/2
switchport mode trunk
no shutdown
exit

! Access ports for PCs (adjust port numbers based on actual connections)
! Example: if Fa0/1 connects to PC
interface fa0/1
switchport mode access
switchport access vlan 10
no shutdown
exit

end
write memory
```

**Verification**:
```cisco
show vlan brief
show interfaces trunk
show ip interface brief    ! To see which ports are UP (connected)
```

### Orange Building (VLAN 20)

```cisco
enable
configure terminal
vlan 20
name Building-B
exit

! Trunk to Core (identify correct port - could be Fa0/2, Fa0/6, etc.)
interface [TRUNK_PORT]
switchport mode trunk
no shutdown
exit

! Access ports for PCs
interface [PC_PORT]
switchport mode access
switchport access vlan 20
no shutdown
exit

end
write memory
```

### Tan Building (VLAN 30) - Admin

```cisco
enable
configure terminal
vlan 30
name Admin
exit

! Trunk to Core
interface [TRUNK_PORT]
switchport mode trunk
no shutdown
exit

! Access ports for PCs and Servers
interface range [PC_PORTS]
switchport mode access
switchport access vlan 30
no shutdown
exit

end
write memory
```

### Testing After Each Building:
1. Click on a PC in that building
2. Desktop → IP Configuration → Select DHCP
3. Verify it gets correct IP (192.168.10.x, 192.168.20.x, or 192.168.30.x)
4. Command Prompt → `ping 192.168.X.1` (ping gateway)
5. Only proceed to next building after successful test!

---

## STEP 4: Branch Multilayer Switch

**Purpose**: Central hub for Branch Campus VLANs

```cisco
enable
configure terminal
hostname BranchCore

! Create VLANs for Branch Campus
vlan 40
name Building01
exit

vlan 50
name Building02
exit

vlan 60
name Hosting
exit

! Configure trunk ports
! Multilayer switch - needs encapsulation
interface range fa0/1-24
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan all
no shutdown
exit

interface range gi0/1-2
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan all
no shutdown
exit

end
write memory
```

**Verification**:
```cisco
show vlan brief
show interfaces trunk
```

---

## STEP 5: Branch Router Configuration

**Purpose**: Inter-VLAN routing and DHCP for Branch Campus (VLANs 40, 50, 60)

### Part A: Subinterfaces

```cisco
enable
configure terminal
hostname BranchRouter

! Bring up physical interface
interface gi0/0
no shutdown
exit

! VLAN 40 Subinterface
interface gi0/0.40
encapsulation dot1Q 40
ip address 192.168.40.1 255.255.255.0
no shutdown
exit

! VLAN 50 Subinterface
interface gi0/0.50
encapsulation dot1Q 50
ip address 192.168.50.1 255.255.255.0
no shutdown
exit

! VLAN 60 Subinterface
interface gi0/0.60
encapsulation dot1Q 60
ip address 192.168.60.1 255.255.255.0
no shutdown
exit
```

### Part B: DHCP Configuration

```cisco
! DHCP Pool for VLAN 40
ip dhcp pool VLAN40-Pool
network 192.168.40.0 255.255.255.0
default-router 192.168.40.1
dns-server 8.8.8.8
exit

! DHCP Pool for VLAN 50
ip dhcp pool VLAN50-Pool
network 192.168.50.0 255.255.255.0
default-router 192.168.50.1
dns-server 8.8.8.8
exit

! DHCP Pool for VLAN 60
ip dhcp pool VLAN60-Pool
network 192.168.60.0 255.255.255.0
default-router 192.168.60.1
dns-server 8.8.8.8
exit

! Exclude gateway addresses
ip dhcp excluded-address 192.168.40.1
ip dhcp excluded-address 192.168.50.1
ip dhcp excluded-address 192.168.60.1

end
write memory
```

**Verification**:
```cisco
show ip interface brief
show ip dhcp pool
```

---

## STEP 6: Router-to-Router Connection

### Configure Serial Connection on Branch Router

```cisco
enable
configure terminal

! Serial interface to Main Router
interface se0/0/1
ip address 10.0.0.2 255.255.255.252
no shutdown
exit

! Static routes to reach Main Campus networks
ip route 192.168.10.0 255.255.255.0 10.0.0.1
ip route 192.168.20.0 255.255.255.0 10.0.0.1
ip route 192.168.30.0 255.255.255.0 10.0.0.1

end
write memory
```

### Configure Serial Connection on Main Router

```cisco
enable
configure terminal

! Serial interface to Branch Router
interface se0/0/1
ip address 10.0.0.1 255.255.255.252
no shutdown
exit

! Static routes to reach Branch Campus networks
ip route 192.168.40.0 255.255.255.0 10.0.0.2
ip route 192.168.50.0 255.255.255.0 10.0.0.2
ip route 192.168.60.0 255.255.255.0 10.0.0.2

end
write memory
```

**Verification**:
```cisco
! On both routers:
show ip interface brief
show ip route

! Test connectivity:
ping 10.0.0.1    ! From Branch Router
ping 10.0.0.2    ! From Main Router
```

---

## STEP 7: Branch Campus Access Switches

### Blue Building (VLAN 40)

```cisco
enable
configure terminal
vlan 40
name Building01
exit

! Trunk to Branch Multilayer Switch
! IMPORTANT: Identify correct port first
interface [TRUNK_PORT]
switchport mode trunk
no shutdown
exit

! Access port for PC
interface [PC_PORT]
switchport mode access
switchport access vlan 40
no shutdown
exit

end
write memory
```

### Pink Building (VLAN 50)

```cisco
enable
configure terminal
vlan 50
name Building02
exit

! Trunk to Branch Multilayer Switch
interface [TRUNK_PORT]
switchport mode trunk
no shutdown
exit

! Access port for PC
interface [PC_PORT]
switchport mode access
switchport access vlan 50
no shutdown
exit

end
write memory
```

### Yellow Building (VLAN 60) - Hosting

```cisco
enable
configure terminal
vlan 60
name Hosting
exit

! Trunk to Branch Multilayer Switch
interface [TRUNK_PORT]
switchport mode trunk
no shutdown
exit

! Access port for PC/Server
interface [PC_PORT]
switchport mode access
switchport access vlan 60
no shutdown
exit

end
write memory
```

---

## PC Configuration

### For ALL PCs in the network:

1. Click on PC
2. Desktop → IP Configuration
3. Select **DHCP**
4. Wait 5-10 seconds for IP assignment

**Expected IP Ranges**:
- Green Building PCs: 192.168.10.2 - 192.168.10.254
- Orange Building PCs: 192.168.20.2 - 192.168.20.254
- Tan Building PCs: 192.168.30.2 - 192.168.30.254
- Blue Building PCs: 192.168.40.2 - 192.168.40.254
- Pink Building PCs: 192.168.50.2 - 192.168.50.254
- Yellow Building PCs: 192.168.60.2 - 192.168.60.254

---

## Testing and Verification

### Test 1: Within Same VLAN (Same Building)
From any PC, ping another PC in the same building:
```
ping [other_PC_IP]
```
**Expected**: Success

### Test 2: Between Different VLANs (Different Buildings in Same Campus)
From Green Building PC, ping Orange Building PC:
```
ping 192.168.20.X
```
**Expected**: Success (via Main Router inter-VLAN routing)

### Test 3: Between Main and Branch Campus
From Green Building PC, ping Blue Building PC:
```
ping 192.168.40.X
```
**Expected**: Success (via Main Router → Branch Router → VLAN 40)

### Test 4: Gateway Connectivity
From any PC, ping its gateway:
```
ping 192.168.X.1
```
**Expected**: Success

### Verification Commands

**On Switches**:
```cisco
show vlan brief              ! View VLAN assignments
show interfaces trunk        ! View trunk ports
show ip interface brief      ! See which ports are UP
show interfaces status       ! Detailed port status
```

**On Routers**:
```cisco
show ip interface brief      ! View interface status and IPs
show ip dhcp binding         ! See DHCP assignments
show ip dhcp pool            ! DHCP pool statistics
show ip route                ! Routing table
```

---

## Troubleshooting Guide

### Issue 1: PC Not Getting DHCP Address

**Check List**:
1. Is PC set to DHCP mode? (Not Static)
2. Is the switch port in the correct VLAN?
   ```cisco
   show vlan brief
   ```
3. Is the trunk between switch and core working?
   ```cisco
   show interfaces trunk
   ```
4. Does the router have the correct subinterface IP?
   ```cisco
   show ip interface brief
   ```
5. Does the DHCP pool exist on the router?
   ```cisco
   show ip dhcp pool
   ```

### Issue 2: Native VLAN Mismatch Error

**Error Message**:
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered
```

**Solution**: 
- This means a port configured as access (VLAN X) is connected to a trunk (VLAN 1)
- Identify which port should be trunk
- Reconfigure it:
  ```cisco
  interface [PORT]
  no switchport access vlan X
  switchport mode trunk
  ```

### Issue 3: Ping Between VLANs Fails

**Check List**:
1. Can PC ping its own gateway?
2. Is router subinterface configured and UP?
3. For Main↔Branch: Are static routes configured on both routers?
4. Are serial interfaces UP on both routers?

### Issue 4: Port Shows "Down"

**Check**:
1. Is cable connected in Packet Tracer?
2. Is interface shutdown?
   ```cisco
   interface [PORT]
   no shutdown
   ```

---

## Key Lessons Learned

1. **Always identify ports BEFORE configuring**
   - Use `show ip interface brief` to see which ports are UP
   - Hover over cables in Packet Tracer to see port numbers

2. **Configure one building at a time and TEST**
   - Don't rush to configure everything at once
   - Test each building before moving to next

3. **Multilayer switches need encapsulation**
   - Regular switches: Just `switchport mode trunk`
   - Multilayer switches: Need `switchport trunk encapsulation dot1q` FIRST

4. **Trunk ports don't appear in VLAN list**
   - If a port is a trunk, it won't show in `show vlan brief`
   - Use `show interfaces trunk` to see trunks

5. **Router subinterfaces need IP addresses**
   - Creating the subinterface is not enough
   - Must assign IP address with `ip address` command

6. **DHCP pools must exist on the router handling that VLAN**
   - Main Router: DHCP for VLANs 10, 20, 30
   - Branch Router: DHCP for VLANs 40, 50, 60

---

## Quick Reference - Port Identification

### How to Find Which Port Connects Where:

1. **In Packet Tracer GUI**:
   - Hover mouse over cable
   - Shows: "Device1 Port X ↔ Device2 Port Y"

2. **Via CLI**:
   ```cisco
   show ip interface brief
   ```
   - Ports with Status "up" are connected
   - Ports with Status "down" are not connected

3. **Check Trunk Status**:
   ```cisco
   show interfaces trunk
   ```
   - Shows all ports configured as trunks

4. **Check CDP Neighbors** (Cisco Discovery Protocol):
   ```cisco
   show cdp neighbors
   ```
   - Shows directly connected devices

---

## Configuration Templates

### Access Switch Template

```cisco
enable
configure terminal
vlan [VLAN_NUMBER]
name [VLAN_NAME]
exit

! Trunk to Core/Distribution Switch
interface [TRUNK_PORT]
switchport mode trunk
no shutdown
exit

! Access ports for end devices
interface [ACCESS_PORT]
switchport mode access
switchport access vlan [VLAN_NUMBER]
no shutdown
exit

end
write memory
```

### Router Subinterface Template

```cisco
enable
configure terminal

interface [PHYSICAL_INTERFACE]
no shutdown
exit

interface [PHYSICAL_INTERFACE].[VLAN_NUMBER]
encapsulation dot1Q [VLAN_NUMBER]
ip address [GATEWAY_IP] [SUBNET_MASK]
no shutdown
exit

ip dhcp pool VLAN[VLAN_NUMBER]-Pool
network [NETWORK_ADDRESS] [SUBNET_MASK]
default-router [GATEWAY_IP]
dns-server 8.8.8.8
exit

ip dhcp excluded-address [GATEWAY_IP]

end
write memory
```

---

## Final Checklist

### Main Campus Configuration:
- [ ] Main Multilayer Switch: VLANs 10, 20, 30 created
- [ ] Main Multilayer Switch: All ports configured as trunks
- [ ] Main Router: Subinterfaces gi0/0.10, .20, .30 with IPs
- [ ] Main Router: DHCP pools for VLANs 10, 20, 30
- [ ] Green Building switches configured and tested
- [ ] Orange Building switches configured and tested
- [ ] Tan Building switches configured and tested

### Branch Campus Configuration:
- [ ] Branch Multilayer Switch: VLANs 40, 50, 60 created
- [ ] Branch Multilayer Switch: All ports configured as trunks
- [ ] Branch Router: Subinterfaces gi0/0.40, .50, .60 with IPs
- [ ] Branch Router: DHCP pools for VLANs 40, 50, 60
- [ ] Blue Building switch configured and tested
- [ ] Pink Building switch configured and tested
- [ ] Yellow Building switch configured and tested

### Inter-Campus Connectivity:
- [ ] Main Router: Serial interface se0/0/1 configured (10.0.0.1)
- [ ] Branch Router: Serial interface se0/0/1 configured (10.0.0.2)
- [ ] Main Router: Static routes to 192.168.40.0, 50.0, 60.0
- [ ] Branch Router: Static routes to 192.168.10.0, 20.0, 30.0
- [ ] Routers can ping each other (10.0.0.1 ↔ 10.0.0.2)

### Final Testing:
- [ ] All PCs get DHCP addresses
- [ ] PCs can ping their gateways
- [ ] PCs within same VLAN can ping each other
- [ ] PCs in different VLANs can ping each other
- [ ] Main Campus PCs can ping Branch Campus PCs
- [ ] No CDP errors or Native VLAN mismatches

---

## Project Complete! 

This network implements:
✅ Hierarchical network design (Core, Distribution, Access layers)
✅ VLAN segmentation (6 VLANs across 2 campuses)
✅ Inter-VLAN routing (Router on a Stick method)
✅ DHCP services (Centralized on routers)
✅ Multi-campus connectivity (via serial WAN links)
✅ Proper trunk configuration
✅ Redundant access layer (multiple switches per building)

**Total Devices Configured**: 
- 2 Routers (Main + Branch)
- 2 Multilayer Switches (Core switches)
- 10+ Access Switches (2960 switches)
- 16+ End devices (PCs and Servers)

**Skills Demonstrated**:
- VLAN creation and management
- Trunk port configuration
- Router subinterfaces (802.1Q encapsulation)
- DHCP server configuration
- Static routing
- Troubleshooting (Native VLAN mismatches, port identification)
- Hierarchical network design
- Multi-campus WAN connectivity

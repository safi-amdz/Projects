# Phase 1  Offline-first Network Build with MikroTik CRS328 SwOS, OPNsense 25.7, UniFi AP, Raspberry Pi control plane, and NAS 


## Objective and build order
You will build the network completely offline first, then cut over to the Internet in Phase 2. The sequence avoids lockouts and makes every change testable.

1. Prepare the switch while it is at factory defaults.
2. Put OPNsense behind the switch with a temporary LAN address that does not collide with the switch default.
3. Plan and create VLANs on the switch with clear reasons for tagged and untagged choices.
4. Create VLAN interfaces on OPNsense and move management to VLAN 10 safely.
5. Bring the Raspberry Pi control plane online on VLAN 10 and run Pi hole, Step CA, and FreeRADIUS in containers.
6. Create the Storage VLAN 50 for the NAS on SFP+3.
7. Configure DHCP and firewall policy per VLAN.
8. Validate offline. No Internet is required to finish Phase 1.

## Hardware and names
1. Firewall Protectli VP6670 running OPNsense 25.7
2. Switch MikroTik CRS328 24P 4S plus RM running SwOS layer 2 only
3. Access Point UniFi Enterprise AP with PoE
4. Server Dell PowerEdge R720 for Proxmox used in Phase 2
5. Control plane Raspberry Pi 4B 8 GB with 256 GB SSD
6. NAS on SFP+3 of the switch
7. Admin laptop for configuration
8. Local domain home.arpa

## Final VLAN plan and addressing
This is the end state you will reach by the end of Phase 1.

| Name                 | VLAN | IPv4 subnet     | Gateway      | DHCP            | Notes                                      |
|----------------------|------|-----------------|--------------|-----------------|--------------------------------------------|
| Management           | 10   | 10.10.10.0/24   | 10.10.10.1   | Reservations    | switch, AP, iDRAC, ctlpln01, core infra    |
| Home                 | 20   | 10.10.20.0/24   | 10.10.20.1   | Yes             | trusted user devices                       |
| Dev Apps             | 30   | 10.10.30.0/24   | 10.10.30.1   | Yes             | lab apps and services                      |
| Untrusted IoT        | 40   | 10.10.40.0/24   | 10.10.40.1   | Yes             | TVs, cameras, IoT                          |
| Storage              | 50   | 10.10.50.0/24   | 10.10.50.1   | No              | NAS only                                   |
| Guest                | 60   | 10.10.60.0/24   | 10.10.60.1   | Captive or PSK  | client isolation                           |
| DMZ                  | 70   | 10.10.70.0/24   | 10.10.70.1   | No              | reverse proxy Phase 2                      |
| VPN                  | 80   | 10.10.80.0/24   | 10.10.80.1   | No              | future                                     |
| Quarantine           | 90   | 10.10.90.0/24   | 10.10.90.1   | Yes             | parking VLAN                               |
| Blackhole Native     | 999  | none            | none         | none            | used only as native on trunks              |

Reserved addresses
| IP          | Hostname              | Purpose                                         |
|-------------|-----------------------|-------------------------------------------------|
| 10.10.10.1  | gw.home.arpa          | OPNsense Management interface on VLAN 10        |
| 10.10.10.2  | sw01.home.arpa        | CRS328 SwOS management                          |
| 10.10.10.5  | idrac.home.arpa       | iDRAC                                           |
| 10.10.10.10 | ox1.home.arpa         | Proxmox host management                         |
| 10.10.10.20 | mgmt01.home.arpa      | VM that hosts UniFi controller Phase 2          |
| 10.10.10.21 | ap01.home.arpa        | UniFi AP                                        |
| 10.10.10.30 | ctlpln01.home.arpa    | Raspberry Pi control plane                      |
| 10.10.50.10 | nas01.home.arpa       | NAS on Storage VLAN 50                          |
| 10.10.70.10 | proxy01.home.arpa     | Reverse proxy VM Phase 2                        |

Service names
1. step.home.arpa is a CNAME to ctlpln01.home.arpa
2. radius.home.arpa is a CNAME to ctlpln01.home.arpa

## Port to VLAN quick map
Use this table when you are plugging devices.

| Switch ports | Device or use     | Mode   | PVID | VLANs carried or assigned                     |
|--------------|--------------------|--------|------|-----------------------------------------------|
| SFP+1   | VP6670 OPNsense    | Trunk  | 999  | 10,20,30,40,50,60,70,80,90,999                |
| SFP+2   | Dell R720 Proxmox  | Trunk  | 999  | 10,20,30,40,50,60,70,80,90,999                |
| SFP+3   | NAS                | Access | 50   | 50                                            |
| SFP+4   | Spare              | Trunk  | 999  | 10,20,30,40,50,60,70,80,90,999                |
| 1 to 4       | Cameras            | Access | 40   | 40                                            |
| 5            | Raspberry Pi       | Access | 10   | 10                                            |
| 6            | UniFi AP           | Access then Trunk | 10 then 999 | 10 first. Later 10,20,40,60,999 |
| 7            | iDRAC              | Access | 10   | 10                                            |
| 8 to 12      | Home clients       | Access | 20   | 20                                            |
| 13 to 14     | Dev wired          | Access | 30   | 30                                            |
| 23           | Quarantine jack    | Access | 90   | 90                                            |
| 24           | Admin laptop       | Access | 10   | 10                                            |

Important notes
1. VLAN Mode must be strict on every port.
2. Trunks use VLAN Receive only tagged and PVID 999. Never check Force VLAN ID on trunks.
3. Access ports use VLAN Receive only untagged and PVID set to that access VLAN. Check Force VLAN ID on access ports to drop any tagged frames sent by clients.
4. Do not include VLAN 999 in access port membership. Keep VLAN 999 only on trunks.

---

## Stage 0  Offline bootstrap approach
Why the temporary 192.168.88.0 slash 24 network
1. SwOS ships at 192.168.88.1 with no VLANs. You need to reach it first.
2. Give OPNsense LAN a temporary IP of 192.168.88.2 so laptop and switch and firewall all talk before you create VLANs.
3. After VLAN 10 exists and works you will move management to 10.10.10.0 slash 24 and remove the temporary network.

What tagged and untagged mean in SwOS
1. Tagged frames carry a VLAN ID inside the Ethernet header. Trunks use tags to carry many VLANs over one link.
2. Untagged frames have no VLAN tag. Access ports use untagged frames for simple endpoints like laptops.
3. Default VLAN ID also called PVID on an access port assigns untagged ingress traffic to a VLAN.
4. Strict VLAN mode enforces membership so a port only accepts and sends VLANs you explicitly allow.
5. Native VLAN on a trunk is the VLAN used for untagged frames arriving on the trunk. You will not use a real VLAN here. You will use VLAN 999 as a blackhole so stray untagged frames do nothing.

---

## A) Physical cabling for the offline lab
1. Connect Protectli SFP+1 to CRS328 SFP+1. This is internal trunk later.
2. Connect the admin laptop to copper port 24 on the CRS328.
3. Keep the ONT and Internet disconnected. Leave the WAN port on the Protectli empty.
4. Keep the UniFi AP unplugged for now. You will adopt it in Phase 2 after the controller exists.
5. Keep the Raspberry Pi powered but leave Ethernet disconnected until VLAN 10 exists.
6. Leave the NAS unplugged until VLAN 50 exists.

---

## B) Reach SwOS at its factory IP and harden basics
Laptop temporary IP
1. On your laptop set a manual IPv4 of 192.168.88.10 with mask 255.255.255.0 and gateway empty.

Log in and change core settings
1. Open http://192.168.88.1 in a browser.
2. Log in as admin with an empty password. Set a strong password immediately.
3. Click System. Set Identity to sw01.home.arpa. Change the IP Address to Static at 10.10.10.2. Enable Independent VLAN Learning. Save.
4. Change the jump01 management pc IP address to 10.10.10.3.

---

## C) Give OPNsense a temporary LAN IP so you can reach its GUI
Use the console on the Protectli
1. Connect a keyboard and HDMI or serial cable to the Protectli.
2. Boot OPNsense and wait for the text menu.
3. Choose Assign Interfaces if needed and make sure your internal NIC is LAN.
4. Choose Set interface IP address then select LAN.
5. Set IPv4 to 10.10.10.4 slash 24. No upstream gateway on LAN. Skip IPv6 for now.

Log in to the OPNsense GUI
1. On the laptop open https://10.10.10.4 and accept the certificate warning.
2. Run the setup wizard. Set a strong admin password and your timezone.
3. Do not change WAN. Leave it disconnected for now.

---

## D) Plan VLANs and mark ports before you configure
Decide the roles now so you do not guess later.
1. SFP+1 to Protectli will become a trunk. Carries VLANs 10, 20, 30, 40, 50, 60, 70, 80, 90. Native PVID set to 999.
2. SFP+2 to Proxmox will be a trunk. Same membership and PVID 999.
3. SFP+3 to NAS will be access in VLAN 50. No tags.
4. SFP+4 spare trunk. Same as SFP+1. PVID 999.
5. Port 24 admin laptop will be access in VLAN 10 while you are building.
6. Port 5 Raspberry Pi will be access in VLAN 10.
7. Port 6 UniFi AP will be access in VLAN 10 for adoption. After adoption you will convert it to a trunk that carries VLANs 10, 20, 40 and 60.
8. Ports 1 to 4 cameras will be access in VLAN 40.
9. Ports 8 to 12 Home wired access in VLAN 20.
10. Ports 13 to 14 Dev wired access in VLAN 30.
11. Port 23 Quarantine access in VLAN 90.

Why PVID 999 on trunks
1. If any untagged frame appears on a trunk it gets mapped into VLAN 999 which has no Layer 3 gateway or DHCP. That traffic dies safely.

---

## E) Create VLANs in SwOS and apply per port behavior

VLANs tab
1. Add VLAN IDs 10, 20, 30, 40, 50, 60, 70, 80, 90, 999. Name them Mgmt, Home, Dev, IoT, Storage, Guest, DMZ, VPN, Quarantine, Blackhole.
2. Learning enabled for all real VLANs.
3. IGMP Snooping enabled for Home and IoT if you use casting.
4. Members for each VLAN. Tick the ports exactly as follows.
   1. VLAN 10 Mgmt members SFP+1, SFP+2, SFP+4, ports 5, 6, 7, 24.
   2. VLAN 20 Home members SFP+1, SFP+2, SFP+4, ports 8, 9, 10, 11, 12.
   3. VLAN 30 Dev members SFP+1, SFP+2, SFP+4, ports 13, 14.
   4. VLAN 40 IoT members SFP+1, SFP+2, SFP+4, ports 1, 2, 3, 4.
   5. VLAN 50 Storage members SFP+1, SFP+2, SFP+3, SFP+4.
   6. VLAN 60 Guest members SFP+1, SFP+2, SFP+4.
   7. VLAN 70 DMZ members SFP+1, SFP+2, SFP+4.
   8. VLAN 80 VPN members SFP+1, SFP+2, SFP+4.
   9. VLAN 90 Quarantine members SFP+1, SFP+2, SFP+4, port 23.
   10. VLAN 999 Blackhole members SFP+1, SFP+2, SFP+4.
5. Save and then go to the VLAN tab.

VLAN tab per port
Set VLAN Mode strict for every port.
1. SFP+1 to OPNsense. VLAN Receive only tagged. Default VLAN ID 999. Force VLAN ID unchecked. Save.
2. SFP+2 to Proxmox. VLAN Receive only tagged. Default VLAN ID 999. Force VLAN ID unchecked. Save.
3. SFP+3 to NAS. VLAN Receive only untagged. Default VLAN ID 50. Force VLAN ID checked. Save.
4. SFP+4 spare trunk. VLAN Receive only tagged. Default VLAN ID 999. Force VLAN ID unchecked. Save.
5. Ports 1 to 4 cameras. VLAN Receive only untagged. Default VLAN ID 40. Force VLAN ID checked. Save.
6. Port 5 Raspberry Pi. VLAN Receive only untagged. Default VLAN ID 10. Force VLAN ID checked. Save.
7. Port 6 UniFi AP during adoption. VLAN Receive only untagged. Default VLAN ID 10. Force VLAN ID checked. Save.
8. Port 7 iDRAC. VLAN Receive only untagged. Default VLAN ID 10. Force VLAN ID checked. Save.
9. Ports 8 to 12 Home. VLAN Receive only untagged. Default VLAN ID 20. Force VLAN ID checked. Save.
10. Ports 13 to 14 Dev. VLAN Receive only untagged. Default VLAN ID 30. Force VLAN ID checked. Save.
11. Port 23 Quarantine. VLAN Receive only untagged. Default VLAN ID 90. Force VLAN ID checked. Save.
12. Port 24 Admin laptop. VLAN Receive only untagged. Default VLAN ID 10. Force VLAN ID checked. Save.



---

# SwOS + OPNsense VLAN10 Cutover (Now, these are Proven Working Steps)

> Scope: CRS328-24P-4S+RM running SwOS and OPNsense 25.7.  
> Goal: Management on VLAN10 only. Switch at 10.10.10.2. OPNsense at 10.10.10.1.  
> Important: **Do not change Port Isolation** from its current default on the switch. Leave it as is.

## Step 1. Basic Access (confirm current state)
1. Connect your laptop to a copper port on the switch.
2. Ensure you can open the SwOS GUI at `http://10.10.10.2`.
3. Ensure OPNsense has the VLAN10 interface enabled with `10.10.10.1/24`:
   - OPNsense: Interfaces → Assignments → select **VLAN10** → Enable checked → IPv4 Static `10.10.10.1/24` → **Save** → **Apply**.
4. OPNsense DHCP for VLAN10:
   - Services → DHCPv4 → **VLAN10** → Enable → Range `10.10.10.200–10.10.10.210` → Save.

## Step 2. Create VLANs (SwOS → VLANs tab)
Add the following VLAN IDs:

- 10 Management  
- 20 Home  
- 30 Dev  
- 40 IoT  
- 50 Storage  
- 60 Guest  
- 70 DMZ  
- 80 VPN  
- 90 Quarantine  
- 999 Blackhole

Keep “Learning” enabled. If you use Chromecast or AirPlay, enable IGMP Snooping for VLAN 20 and 40.

## Step 3. VLAN Memberships (SwOS → VLANs tab)
Tick ports as members exactly:

- **VLAN 10**: SFP+1, SFP+2, SFP+4, ports 5, 6, 7, 24  
- **VLAN 20**: SFP+1, SFP+2, SFP+4, ports 8–12  
- **VLAN 30**: SFP+1, SFP+2, SFP+4, ports 13–14  
- **VLAN 40**: SFP+1, SFP+2, SFP+4, ports 1–4  
- **VLAN 50**: SFP+1, SFP+2, **SFP+3**, SFP+4  
- **VLAN 60**: SFP+1, SFP+2, SFP+4  
- **VLAN 70**: SFP+1, SFP+2, SFP+4  
- **VLAN 80**: SFP+1, SFP+2, SFP+4  
- **VLAN 90**: SFP+1, SFP+2, SFP+4, port 23  
- **VLAN 999**: SFP+1, SFP+2, SFP+4

Click **Save**.

> Note on Port Isolation: **Do not change** the Port Isolation defaults in this tab or in the Port Isolation tab. Leave them as is, per your working setup.

## Step 4. Per-Port Behavior (SwOS → VLAN tab)
Configure each port:

### Trunks
- **SFP+1 (to OPNsense)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only tagged**  
  - Default VLAN ID (PVID) = **999**  
  - Force VLAN ID = **unchecked**

- **SFP+2 (to Proxmox)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only tagged**  
  - Default VLAN ID (PVID) = **999**  
  - Force VLAN ID = **unchecked**

- **SFP+4 (spare trunk)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only tagged**  
  - Default VLAN ID (PVID) = **999**  
  - Force VLAN ID = **unchecked**

### Access
- **SFP+3 (NAS)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **50**  
  - Force VLAN ID = **checked**

- **Ports 1–4 (POE Security Cameras)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **40**  
  - Force VLAN ID = **checked**

- **Port 5 (Raspberry Pi)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **10**  
  - Force VLAN ID = **checked**

- **Port 6 (UniFi AP)**  
  - Start as **access** for adoption: strict, only untagged, **PVID = 10**, Force **checked**.  
  - Later in Phase 2 convert to trunk carrying **10, 20, 40, 60, 999**.

- **Port 7 (iDRAC)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **10**  
  - Force VLAN ID = **checked**

- **Ports 8–12 (Home)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **20**  
  - Force VLAN ID = **checked**

- **Ports 13–14 (Dev wired)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **30**  
  - Force VLAN ID = **checked**

- **Port 23 (Quarantine)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **90**  
  - Force VLAN ID = **checked**

- **Port 24 (Admin laptop)**  
  - VLAN Mode = **strict**  
  - VLAN Receive = **only untagged**  
  - Default VLAN ID (PVID) = **10**  
  - Force VLAN ID = **checked**

Click **Save**.

## Step 5. Other Tabs
- **Link**: verify SFP+ ports link at 10G; copper at 1G.  
- **PoE**: enable PoE Out Auto on ports 1–4 (cameras) and 6 (AP). Others Off.  
- **RSTP**: enable RSTP. Mark **access ports** (1–4, 5, 6, 7, 8–14, 23, 24) as **Edge = Yes**. Leave trunks (SFP+1, SFP+2, SFP+4) **Edge = No**.  
- **Forwarding**: set broadcast and multicast storm control to **1%** on access ports.  
- **IGMP**: enable global IGMP Snooping. Do not enable Querier yet.  
- **SNMP**: disable unless monitoring. If enabled, restrict to `10.10.10.0/24` with a strong community.  
- **System**: keep switch IP at `10.10.10.2/24`. Restrict management to VLAN10 and `10.10.10.0/24` only.

## Step 6. OPNsense — retire the old LAN IP (this was your final fix)
1. OPNsense → Interfaces → **LAN** → set **IPv4 Configuration Type = None** → **Save** → **Apply**.  
   - This removes `10.10.10.4` and eliminates the same-subnet overlap that caused conflicts.  
2. From your admin laptop on **Port 24 (PVID 10)**:  
   - Set NIC to DHCP. Confirm lease `10.10.10.200–10.10.10.210`, gateway `10.10.10.1`.  
   - Open `https://10.10.10.1` (OPNsense GUI).  
   - Open `http://10.10.10.2` (SwOS GUI).

## Step 7. Validate and Backup
- Confirm you can consistently reach:
  - OPNsense at `https://10.10.10.1` from Port 24.  
  - SwOS at `http://10.10.10.2` from Port 24.  
- SwOS → System → **Backup** → download config.  
- OPNsense → System → Configuration → Backups → download config.

---

## Notes and rationale
- Leaving **Port Isolation** at current default was necessary in this environment, since they're isolated anyway. Do not change it now that everything is stable.  
- Retiring the **LAN IPv4** was essential because having `10.10.10.4` (LAN) and `10.10.10.1` (VLAN10) in the same /24 causes ARP and routing ambiguity. Removing LAN’s IP resolved this.  
- **VLAN999** is your native “blackhole” VLAN for trunks; it is not the trunk itself. We set PVID 999 on trunks to safely dispose of stray untagged frames.  
- Convert the AP port to a trunk later (Phase 2) after the controller is up and SSIDs are mapped to their VLANs.

---


# OPNsense Bootstrap, VLAN Mapping, 802.1Q, and Security Audit

> Target hardware: Protectli VP6670 (OPNsense 25.7) + MikroTik CRS328-24P-4S+RM (L2)  
> Goal: Clean, least‑privilege segmentation with standards‑based 802.1Q tagging and defense‑in‑depth.

---

## Part 1. Initial OPNsense Bootstrap

**When first booting the Protectli (OPNsense 25.7):**

### A. Assign temporary LAN IP
1. On the OPNsense console menu, select **Option 2 — Assign Interfaces**.  
2. Choose **LAN = `ixl0`** (the 10 GbE interface cabled to **CRS328 SFP+1**).  
3. Set temporary **static IPv4** to **`10.10.10.4/24`**.  
4. This gives GUI access via the switch (which will be at **`10.10.10.2`**).

### B. Access the GUI
1. On **jump01** (plugged into **switch port 24**), set **static IP `10.10.10.3/24`**.  
2. Browse to **`https://10.10.10.4`** and log in.

### C. Create **VLAN10 (Management)**
1. Navigate **Interfaces → Devices → VLAN → Add**.  
2. **Parent interface**: `ixl0`  
3. **VLAN Tag**: `10`  
4. **Description**: `Management`  
5. **Save** and **Apply**.

### D. Assign the new interface
1. **Interfaces → Assignments → +** to add the newly created VLAN device.  
2. Enable the interface; set **static IPv4 = `10.10.10.1/24`**.  
3. **Apply**.

### E. Migrate management to VLAN10
1. Test access to **`https://10.10.10.1`** from **jump01**.  
2. Once confirmed, go to **Interfaces → LAN** and set **IPv4 = None**.  
3. This retires **`10.10.10.4`** and leaves management on **`10.10.10.1`** only.

---

## Part 2. VLAN‑to‑OPNsense Interface Mapping

| VLAN ID | Name        | OPNsense Interface   | Subnet            | DHCP Pool               | Purpose / Devices                                                                     | Notes                                                                                                 |
|--------:|-------------|----------------------|-------------------|-------------------------|---------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 10      | Management  | VLAN10 (`ixl0.10`)   | 10.10.10.1/24     | 10.10.10.200–210        | Switch mgmt (10.10.10.2), OPNsense GUI, Pi, UniFi Controller VM, iDRAC               | Primary mgmt only. Allow HTTPS/DNS/SSH to mgmt tools. **Block Internet** except updates.              |
| 20      | Home        | VLAN20 (`ixl0.20`)   | 10.10.20.1/24     | 10.10.20.100–150        | Personal devices: PCs, TVs, printers                                                  | Internet allowed. **No access** to Mgmt/Servers except DNS.                                           |
| 30      | Dev         | VLAN30 (`ixl0.30`)   | 10.10.30.1/24     | 10.10.30.100–150        | Dev VMs, coding boxes, lab tools                                                      | Internet allowed. May reach Servers when explicitly permitted.                                        |
| 40      | IoT         | VLAN40 (`ixl0.40`)   | 10.10.40.1/24     | 10.10.40.100–200        | Security cams (ports 1–4), smart plugs                                                | Internet allowed (apps). **No lateral** or Mgmt access.                                               |
| 50      | Storage     | VLAN50 (`ixl0.50`)   | 10.10.50.1/24     | 10.10.50.100–120        | NAS (SFP+3), backup appliances                                                        | Internet only for updates. Accessible from Dev/Servers as needed.                                     |
| 60      | Guest       | VLAN60 (`ixl0.60`)   | 10.10.60.1/24     | 10.10.60.50–150         | Visitors’ phones, laptops                                                             | **Internet‑only**. Block LAN/Mgmt/Servers.                                                            |
| 70      | DMZ         | VLAN70 (`ixl0.70`)   | 10.10.70.1/24     | Static/manual           | Reverse proxy, public‑facing services                                                 | Controlled Internet + inbound NAT. **No LAN access**.                                                 |
| 80      | VPN         | VLAN80 (`ixl0.80`)   | 10.10.80.1/24     | 10.10.80.100–150        | VPN tunnel endpoints, jump box                                                        | Internet allowed. Policy routes to internal nets via VPN policies.                                    |
| 90      | Quarantine  | VLAN90 (`ixl0.90`)   | 10.10.90.1/24     | 10.10.90.100–120        | Suspicious/infected machines                                                          | **Internet only** (DNS + updates). No local access.                                                   |
| 999     | Blackhole   | —                    | —                 | —                       | Catch‑all for untagged/unused ports                                                   | Exists **only** on the switch, **not** in OPNsense.                                                    |

---

## Part 3. Defense‑in‑Depth Firewall Rules (per‑VLAN)

> Create these under **Firewall → Rules → [interface]** in OPNsense. Log denies where noted.

### VLAN10 (Management)
- **Allow** `VLAN10 net → 10.10.10.30` **TCP/UDP 53 (DNS)** *(Pi-hole).*
- **Block** `VLAN10 net → any` **TCP/UDP 53** *(block external DNS; keep above the Internet-allow rule).*
- **Allow** `VLAN10 net → This Firewall` TCP 443 (GUI).  
- **Allow** `VLAN10 net → This Firewall` TCP 22 (SSH, if enabled).  
- **Allow** `VLAN10 net → *` DNS, NTP, ICMP.  
- **Allow** `VLAN10 net → Internet` (firmware/OS updates only; optionally use an alias).  
- **Block** `VLAN10 net →` all other internal VLANs (except DNS resolver if off‑box).  
- **Log all denies**.

### VLAN20 (Home)
- **Allow** `VLAN20 net → 10.10.10.30` **TCP/UDP 53 (DNS)** *(Pi-hole).*
- **Block** `VLAN20 net → any` **TCP/UDP 53** *(block external DNS; keep above the Internet-allow rule).*
- **Allow** `VLAN20 net → Internet (any)`  
- **Allow** `VLAN20 net → VLAN20 net` (intra‑LAN as needed)  
- **Block** `VLAN20 net → VLAN10 (Mgmt)`  
- **Block** `VLAN20 net → VLAN30, VLAN40, VLAN50` (permit specific NAS/printer access if desired)  
- DNS via Pi‑hole or OPNsense.

### VLAN30 (Dev)
- **Allow** `VLAN30 net → 10.10.10.30` **TCP/UDP 53 (DNS)** *(Pi-hole).*
- **Block** `VLAN30 net → any` **TCP/UDP 53** *(block external DNS; keep above the Internet-allow rule).*
- **Allow** `VLAN30 net → Internet`  
- **Allow** `VLAN30 net → VLAN50 (Storage)`  
- **Block** `VLAN30 net → VLAN10 (Mgmt)`  
- **Block** `VLAN30 net → VLAN20 (Home)`  
- Optional: **allow** SSH/RDP into Servers/DMZ for testing (tight scope + auth).

### VLAN40 (IoT / Cameras)
- **Allow** `VLAN40 net → 10.10.10.30` **TCP/UDP 53 (DNS)** *(Pi-hole).*
- **Block** `VLAN40 net → any` **TCP/UDP 53** *(block external DNS; keep above the Internet-allow rule).*
- **Allow** `VLAN40 net → Internet` (app/cloud)  
- **Allow** `VLAN40 net → VLAN50` (NVR/NAS target only)  
- **Block** `VLAN40 net → VLAN10, VLAN20, VLAN30`  
- **Block** `VLAN40 net → VLAN40 net` (isolate IoT devices from each other, if desired)  
- Only **DNS, NTP, HTTPS** outbound otherwise.

### VLAN50 (Storage)
- **Allow** `VLAN50 net → Internet` (updates)  
- **Allow** from `VLAN30 (Dev)` (file access)  
- **Allow** from `VLAN20 (Home)` if media access desired  
- **Block** `VLAN50 net → VLAN10 (Mgmt)`  
- **Block** `VLAN50 net → VLAN40 (IoT)`  
- Default **block** all else.

### VLAN60 (Guest)
- **Allow** `VLAN60 net → 10.10.10.30` **TCP/UDP 53 (DNS)** *(Pi-hole).*
- **Block** `VLAN60 net → any` **TCP/UDP 53** *(block external DNS; keep above the Internet-allow rule).*
- **Allow** `VLAN60 net → Internet`  
- **Block** `VLAN60 net →` all RFC1918 (10/8, 172.16/12, 192.168/16)  
- **Block** `VLAN60 net → VLAN10,20,30,40,50,70,80,90`  
- **Allow** DNS only to OPNsense/Pi‑hole.

### VLAN70 (DMZ)
- **Allow** `VLAN70 net → Internet`  
- **Allow** inbound **NAT**: `Internet → VLAN70` **specific** services only (e.g., 443 to reverse proxy)  
- **Block** `VLAN70 net → LANs (Mgmt, Home, Dev, IoT, etc.)`  
- **Allow** `VLAN70 net → VLAN50` only if reverse proxy must fetch content from NAS  
- **Log all denies**.

### VLAN80 (VPN)
- **Allow** `VPN clients → VLAN10 (Mgmt)` only if remote admin required (strong auth + MFA).  
- **Allow** `VPN clients → VLAN20, VLAN30, VLAN50` per access policy.  
- **Block** `VPN clients → VLAN40 (IoT), VLAN60 (Guest)`  
- **Allow** `VPN clients → Internet`  
- Enforce **MFA** on VPN service.

### VLAN90 (Quarantine)
- **Allow** `VLAN90 net → 10.10.10.30` **TCP/UDP 53 (DNS)** *(Pi-hole).*
- **Block** `VLAN90 net → any` **TCP/UDP 53** *(block external DNS; keep above the Internet-allow rule).*
- **Allow** `VLAN90 net → Internet` **only DNS, HTTP/HTTPS, Windows Update**  
- **Block** `VLAN90 net → all other VLANs`  
- **Log all denies**.

### VLAN999 (Blackhole)
- No interface in OPNsense. Not routed. Switch drops or sinks unintended traffic.

---

## Part 4. DHCP Best Practices
- Use **ISC DHCPv4** in OPNsense.  
- **Enable DHCP** only on dynamic‑client VLANs: **20, 30, 40, 60, 80, 90**.  
- **Static reservations** for infrastructure: switch, NAS, APs, Pi, controller VMs, iDRAC.  
- On **Mgmt (VLAN10)**, **disable DHCP** except a **small jump01 pool (200–210)** if you want plug‑and‑play access on a jump port.  
- Use **short leases** on Guest and Quarantine (e.g., 2–8 hours).

---

## Part 5. Defense‑in‑Depth Extras
- **Block Bogon Networks** on all internal VLANs (Interfaces → [VLAN] → Block bogons).  
- **Block Private Networks** on **WAN**.  
- **DNS Resolver** (or Forwarder) with **DNSSEC** enabled.  
- **Syslog** OPNsense and switch logs to your NAS/Pi (remote syslog).  
- **Auto‑backup** OPNsense config (System → Configuration → Backups).  
- Enable **port‑security / storm control** and **isolation** on switch access ports (already planned).  
- Use **NTP** from OPNsense or a trusted upstream; pin NTP in rules.  
- Maintain **allow‑list** for firmware/update domains (aliases) to minimize broad egress.

---

## Part 6. 802.1Q VLAN Tagging — Switch & Firewall

### Why 802.1Q here?
- **802.1Q** is the standard for **single‑tag VLAN trunking** between your switch (CRS328) and OPNsense.  
- **Do _not_ use 802.1ad (QinQ)** unless you intentionally need double‑tag encapsulation (you don’t in this design).  
- Avoid “Auto” mode ambiguity; **explicitly choose 802.1Q** where the UI offers a choice.

### OPNsense (ixl0 trunk)
- The physical **`ixl0`** acts as a **trunk** carrying **tags 10,20,30,40,50,60,70,80,90**.  
- Each VLAN interface in OPNsense is created as **`ixl0.<tag>`** (e.g., `ixl0.10`).  
- **No untagged/native VLAN** on the OPNsense side of the trunk.

### MikroTik CRS328 (SwOS guidance)
1. **VLANs table:** Create entries for **10,20,30,40,50,60,70,80,90,999**.  
2. **Ports → VLAN Mode:** Set **Trunk** on the uplink to OPNsense (**SFP+1**).  
3. **Ports → Allowed VLANs:** On the trunk, allow **10,20,30,40,50,60,70,80,90** (**exclude 999** from the trunk so blackhole stays local to the switch).  
4. **Access ports:**  
   - Assign per‑VLAN **PVID** (e.g., camera ports PVID=40, NAS port PVID=50, jump01 port PVID=10).  
   - **VLAN Mode:** Access (egress untagged, ingress tagged → PVID).  
5. **Native VLAN:** Prefer **no native** on trunks (tag everything). If the UI forces one, set an unused value and block it on OPNsense.  
6. **Blackhole VLAN 999:**  
   - Create VLAN **999** only on the switch.  
   - Set **unused ports** PVID=999 and **isolate** them. No uplink allowed. This safely sinks stray/untagged traffic.

> **If you see “VLAN number must be unique”** when adding VLANs: ensure each VLAN ID appears **only once** in the VLAN table and then applied to multiple ports via **Allowed/Member** lists instead of creating duplicate entries for the same ID.

> **If presented with `Auto`, `802.1Q`, `802.1ad` in the VLAN type dropdown:** choose **802.1Q** for all VLANs and trunks in this design.

---

## Part 7. Configuration Security Audit & Hardening Actions

### Summary verdict
- **Segmentation:** Sound. Traffic constrained by per‑VLAN rules and explicit inter‑VLAN allowances.  
- **Choke points:** Single OPNsense trunk is clear and auditable.  
- **Residual risk:** Mis‑tagged/native VLANs, overly broad egress, weak VPN auth, IoT lateral movement, logging gaps.

### Important fixes that is better to apply them now
1. **Tag everything on trunks**; avoid native/untagged VLANs. Confirm switch trunk to OPNsense has **only tagged 10–90**.  
2. **Lock down egress** with **aliases** per VLAN (e.g., restrict Mgmt/Storage to update domains).  
3. **VPN MFA** enforced; restrict VPN to **named user groups** with **per‑group firewall rules**.  
4. **IoT isolation:** Keep **VLAN40 → VLAN40** blocked; allow only required outbound (DNS/NTP/HTTPS) and specific NVR targets.  
5. **DMZ inbound NAT minimization:** Only required ports; terminate TLS on reverse proxy with strong ciphers, forward internally over mTLS where possible.  
6. **Admin plane hygiene:**  
   - OPNsense GUI **TLS** only, **no WAN exposure**; restrict GUI/SSH to **VLAN10** and VPN admin IPs.  
   - **Key‑based SSH**, disable password auth.  
   - **Per‑admin accounts**; no shared credentials.  
7. **Logging & alerting:**  
   - Remote **syslog** for OPNsense + switch.  
   - Enable **intrusion detection** (Suricata) in IPS or IDS mode on selected VLANs (monitor first, then enforce).  
8. **DHCP discipline:** No DHCP on Mgmt (except tiny jump pool). Short leases on Guest/Quarantine. Static reservations for infrastructure.  
9. **Time sync:** Lock NTP to OPNsense; block outbound NTP except from OPNsense and designated servers.  
10. **Firmware & backups:** Auto‑backup OPNsense; keep Protectli BIOS/firmware current; snapshot configuration before major changes.

### Will wrap it up with these hardenings as well
- **DNS over TLS** or **DoH** on resolver/forwarder; pin upstreams via aliases.  
- **802.1X/EAP‑TLS** on wired switch ports where feasible (Mgmt/Servers) with a RADIUS VM.  
- **mTLS** inside the LAN for critical services (proxy ↔ backend).  
- **Threat‑intel blocklists** (pfBlockerNG or native aliases) with careful testing.

# IPv4-Only Lockdown Checklist (Will be implementing IPv6 dual stack of ULA and GUA Later)

> Goal: ensure all VLANs and WAN run **IPv4 only**, no IPv6 addresses are advertised or routed. Prevents clients from bypassing IPv4 firewall rules via auto-IPv6.

---

## 1. Disable IPv6 on all interfaces
1. In **OPNsense GUI** → **Interfaces → [each interface]**  
   - Set **IPv6 Configuration Type** = **None**.  
   - Apply on: **WAN**, **VLAN10–90**, and any spares.  
   - Click **Save** → **Apply Changes**.

---

## 2. Disable Router Advertisements
1. **Services → Router Advertisements**  
   - For every interface listed, set **Router Advertisements** = **Disabled**.  
   - Save.

---

## 3. Disable DHCPv6
1. **Services → DHCPv6 → [each interface]**  
   - Ensure **Enable DHCPv6 Server** is **unchecked**.  
   - Save.

---

## 4. Firewall “Block all IPv6” rules
For belt-and-suspenders protection:

1. **Firewall → Rules → [each VLAN interface] → IPv6 tab**  
   - Add rule:  
     - **Action**: Block  
     - **Interface**: VLANxx  
     - **Protocol**: IPv6 (any)  
     - **Source**: any  
     - **Destination**: any  
     - Enable **Log** (optional).  
   - Move to top of ruleset.  
   - Save & Apply.

2. Repeat for **WAN (IPv6 tab)**.

---

## 5. DNS / Unbound
1. **Services → Unbound DNS → General**  
   - Under **Network Interfaces**, deselect **All** if checked.  
   - Explicitly include only IPv4 interfaces (VLANs + loopback).  
   - Save & Apply.

---

## 6. Pi-hole (if used)
1. **Settings → DNS**  
   - Disable IPv6 listening.  
   - Or, bind only to IPv4 addresses (10.10.x.x).  
2. **UFW**: ensure no IPv6 rules are present (`sudo ufw status` → should say “inactive (v6)”).

---

## 7. Verification
On a test client (Windows/macOS/Linux):
- Run `ipconfig` / `ip -6 addr` → should only see **link-local IPv6 (fe80::)**, no global (`2000::/3`) or ULA (`fdxx::/8`).  
- Run `ping -6 ipv6.google.com` → should **fail**.  
- Run `tracert -6 ipv6.google.com` / `traceroute -6` → should **fail**.

---

✅ With these steps in place:
- No IPv6 config exists in OPNsense.  
- No RA/DHCPv6 runs, so clients can’t get routable IPv6 addresses.  
- Firewall explicitly drops stray IPv6.  
- DNS only listens/responds on IPv4.  

You can safely leave IPv6 untouched until you’re ready in Phase 7.
---

## ✅ Outcome

With this structure:

- Every VLAN is represented in OPNsense with a distinct interface and subnet.  
- Each VLAN has a minimal **DHCP scope** and **narrow firewall rules**.  
- **802.1Q** tagging is explicit and consistent; there is **no native VLAN** on trunks.  
- Defense‑in‑depth = **segmentation + least privilege + strong auth + logging**.


---

## X) Bring up the Raspberry Pi control plane on VLAN 10
1. Plug the Pi Ethernet into port 5 which is access VLAN 10.
2. Give the Pi static IPv4 10.10.10.30 slash 24 with gateway 10.10.10.1.
3. If using optional local IPv6 give the Pi fd10:10:10:10::30 slash 64.
4. Verify Pi hole admin at http://10.10.10.30/admin opens from a VLAN 10 host.
5. Keep Conditional Forwarding disabled in Pi hole.

Secure the firewall on ctlpln01 with UFW
1. Ensure UFW default deny incoming and default allow outgoing.
2. Allow SSH only from 10.10.10.0 slash 24 to port 22 TCP.
3. Allow Pi hole admin only from 10.10.10.0 slash 24 to port 80 TCP.
4. Allow Step CA only from 10.10.10.0 slash 24 to port 443 TCP.
5. Allow RADIUS from 10.10.10.21 and 10.10.10.1 to ports 1812 and 1813 UDP.
6. Allow DNS from client VLANs to port 53 TCP and UDP with the following commands.
```
sudo ufw allow from 10.10.20.0/24 to any port 53 proto tcp
sudo ufw allow from 10.10.20.0/24 to any port 53 proto udp
sudo ufw allow from 10.10.30.0/24 to any port 53 proto tcp
sudo ufw allow from 10.10.30.0/24 to any port 53 proto udp
sudo ufw allow from 10.10.40.0/24 to any port 53 proto tcp
sudo ufw allow from 10.10.40.0/24 to any port 53 proto udp
sudo ufw allow from 10.10.60.0/24 to any port 53 proto tcp
sudo ufw allow from 10.10.60.0/24 to any port 53 proto udp
sudo ufw allow from 10.10.90.0/24 to any port 53 proto tcp
sudo ufw allow from 10.10.90.0/24 to any port 53 proto udp
```
8. If you add new client VLANs later, repeat matching **ufw allow … port 53** TCP and UDP rules for each new subnet.

7. Enable UFW and verify status.

Point DHCP to Pi hole and block DNS bypass
1. Services then DHCPv4 then each client VLAN. Set DNS servers to 10.10.10.30. Save.
2. System then Settings then General. Set firewall DNS to 10.10.10.30. Confirm WAN override is unchecked.
3. Firewall then Rules then each client VLAN. Create rules in this order.
   1. Pass to 10.10.10.30 TCP 53.
   2. Pass to 10.10.10.30 UDP 53.
   3. Block to This Firewall TCP 53 with Log.
   4. Block to This Firewall UDP 53 with Log.
   5. Leave default block below or add a general allow for intra VLAN testing as needed.

Unbound binding on OPNsense
1. Services then Unbound DNS then General.
2. Network Interfaces set only Localhost and VLAN10.
3. Save and Apply.

---

## Y) Storage VLAN 50 for NAS
Why a storage VLAN
1. Isolates SMB, NFS and iSCSI from user and IoT traffic.
2. Lets you write precise firewall rules for storage flows.
3. Keeps broadcast and discovery noise away from the NAS.

OPNsense interface and rules
1. Interfaces then Other Types then VLAN. Add tag 50 if not already present.
2. Interfaces then Assignments. Add VLAN50. Enable it.
3. Description VLAN50 Storage. IPv4 Static 10.10.50.1 slash 24. Save and Apply.
4. Do not enable DHCP on VLAN 50.
5. Firewall then Aliases. Add host alias nas01 10.10.50.10.
6. Firewall then Rules then VLAN50.
   1. Pass from VLAN10 net to VLAN50 net TCP 443 and TCP 22 for NAS management as needed.
   2. Pass from VLAN20 net to nas01 TCP 445 for SMB.
   3. Pass from VLAN30 net to nas01 TCP 2049 and TCP 111 for NFS.
   4. Leave default block under these passes.

NAS side
1. Plug the NAS into SFP+3.
2. Set NAS IP to 10.10.50.10 slash 24. Gateway 10.10.50.1. DNS 10.10.10.30.
3. Disable SMB1. Prefer SMB3 or NFSv4 only.
4. Disable UPnP and DLNA unless required.
5. Optional. Use MTU 9000 only if every hop and endpoint supports it. Otherwise keep 1500.
Checklist for MTU 9000
- NAS NIC MTU set to 9000
- CRS328 SFP+3 port MTU set to 9000 if exposed in SwOS, else leave default and keep 1500 everywhere
- OPNsense VLAN50 interface MTU set to 9000
- Every endpoint or VM that talks to the NAS set to 9000
If any one item is not 9000, keep everything at 1500 to avoid path MTU issues.


Validation for storage
1. From VLAN 20 map \\nas01.home.arpa\share and confirm access.
2. From VLAN 30 mount an NFS export from nas01.home.arpa and confirm access.
3. From VLAN 40 IoT and VLAN 60 Guest confirm the NAS is not reachable.


---

## Z) Phase 1 validation all offline
Basic reachability
1. From the laptop on port 24 open https://10.10.10.1 and http://10.10.10.2.
2. From the laptop open http://10.10.10.30/admin.

DHCP and DNS
1. On a test host in VLAN 20 confirm it gets a 10.10.20.x address and DNS 10.10.10.30.
2. From the same host run nslookup ctlpln01.home.arpa 10.10.10.30 and see 10.10.10.30.
3. Run nslookup google.com 10.10.10.1 and it should fail which proves the firewall resolver is blocked for clients.

Isolation
1. From a VLAN 20 host try to open the switch or OPNsense GUI. It should fail.
2. From a VLAN 10 host both GUIs should work.

Snapshots and backups
1. Download a fresh SwOS backup.
2. In OPNsense open System then Configuration then Backups and save a local copy.

Now ready for Phase 2.

# Azure Mini SOC Project

## Overview

This project aims to deploy a segmented network in Azure with a firewall, honeypot, target host (equipped with an EDR agent), an EDR server, an attacker host, and a SIEM. The goal is to create a controlled environment for cybersecurity analysis, monitoring, and testing.

### Components:
- **Firewall:** Palo Alto VM
- **Network Segmentation:** Virtual subnets with enforced security policies
- **EDR & SIEM:** Detection and response mechanisms
- **Honeypot:** Simulated vulnerable system
- **Attacker Host:** Controlled environment for testing attacks
- **Azure Virtual Network (VNet)**: Structured network with routing and access controls

## Initial Network Design and Challenges

Initially, I planned a traditional network topology with physical-like connectivity, requiring a VM with four interfaces: one for management and three for passing traffic. However, Azure's free-tier has a constraint of 4 vCPUs, limiting the number of VMs that can be created.

To optimize resources:
- The firewall VM must use a maximum of **2 vCPUs**, leaving **2 vCPUs** for other VMs.
- The ideal choice, **DS12-1_v2** (1 vCPU, 4 NICs), did not boot due to a critical error.
- After multiple attempts, **D2ds_v4** was found to be a functional alternative. It provides **2 vCPUs** and supports **3 NICs**, meaning only **2 data interfaces** are available, necessitating a redesign of the topology.

## Redesigned Topology

To accommodate the limitations, I structured the network as follows:

- **Virtual Network (VNet):** Named `Mini_SOC_VNet` with a **10.0.0.0/16** address space.
- **Subnets:**
  - `FW-Untrust` (10.0.1.0/24)
  - `FW-Trust` (10.0.2.0/24)
  - `DMZ` (10.0.20.0/24)
  - `Private1` (10.0.30.0/24)
  - `Private2` (10.0.40.0/24)

## Deploying the Firewall

### Step 1: Provisioning the Palo Alto VM

- Locate **Palo Alto VM Image** in the **Azure Marketplace**.
- Select the appropriate subnet for the **management interface**.
- Deploy the VM and retrieve its **public IP**.

### Step 2: Initial Firewall Configuration

1. Connect via SSH using the assigned public IP.
2. Change the default password for Web UI access:
   ```shell
   configure
   set mgt-config users admin password
   commit
   ```
3. Log in to the Web UI and configure:
   - **Management Profile:** Allow SSH, HTTPS, and ICMP only from a trusted home IP.
   - **Static IP Allocation:** Assign static IPs to interfaces instead of relying on Azure DHCP.
   - **Virtual Router:**
     - Default route (`eth1`) via **10.0.1.1** (Azure Gateway in FW-Untrust subnet)
     - Routes to **Private1** and **Private2** subnets via **eth2** (next-hop 10.0.2.1)
   - **DNS and NTP Configuration**

### Step 3: Adjusting Network Interfaces

- **Attach Untrust and Trust interfaces** with appropriate IPs.
- **Detach public IP** from the management interface and reassign it to **Untrust**, ensuring remote access security.
- This configuration ensures:
  - The firewall is only accessible externally via **Untrust**.
  - The **Mgmt interface** remains private, reserving access for internal use or emergencies.

## Enforcing Firewall-Centric Routing

By default, Azure routes traffic between subnets directly. To force traffic through the firewall:

1. **Create a Route Table for `Private1` and `Private2`:**
   - **Within-VNet Rule:** Directs all `Mini_SOC_VNet` traffic (`/16`) to the firewall (Virtual Appliance).
   - **Within-Subnet Rule:** Ensures intra-subnet traffic remains local (VNetLocal) to prevent accidental redirection.
   - **Default Route:** Points to the internet via Azure Gateway (for initial SSH access).

2. **Create a Route Table for the DMZ:**
   - Similar configuration as `Private1`/`Private2`, but tailored for the DMZ subnet.

## Testing Initial Connectivity

1. Deploy **two Debian VMs**, one in `Private1`, another in `Private2`.
2. SSH into both and attempt **inter-subnet communication**.
3. Traffic logs confirm that packets are inspected and allowed due to the **default interzone policy**.

4. Move a VM to `DMZ` and attempt connectivity:
   - Ping requests fail due to the **default deny rule** for different zones.
   - Create an explicit firewall **policy** to allow `Private1` and `Private2` to reach `DMZ`.
   - Re-test, and observe the logs confirming allowed traffic.

## Enabling Internet Access via Firewall

To route outbound traffic through the firewall:
1. Modify **default routes** in **DMZ, Private1, and Private2** to use the firewall as the **next hop**.
2. This introduces **asymmetric routing**, preventing SSH via public IPs.
3. Instead of configuring **GlobalProtect VPN (overkill for this use case)**, set up **Point-to-Site VPN** in Azure:
   - Detach public IPs from Debian VMs.
   - Create a **Gateway Subnet** and deploy a **Virtual Network Gateway**.
   - Configure **Point-to-Site VPN** (address pool, authentication, routing).
   - Once connected, observe the **new subnet and routes** assigned.

4. Update firewall and Azure routing:
   - Direct all **internet-bound traffic** through the firewall.
   - Ensure **VPN traffic** passes through the firewall before reaching subnets.
   - Disable **gateway route propagation** for `Private1`, `Private2`, and `DMZ` but **enable it for FW-Trust**.
   - Add a **VPN subnet route** to the **firewall's Virtual Router**.

## Finalizing Security & Optimization

1. **Tighten Security Policies:**
   - Make rules **more granular**.
   - Set the **default intrazone policy** to **deny** instead of allow.

2. **Restrict Firewall Management Access:**
   - Remove the **management profile** from `Untrust`.
   - Configure service routes to use **Untrust** for necessary external services.

## Conclusion

This project successfully establishes a **firewall-protected Azure network** with:
- **Segmented subnets** enforcing strict security policies.
- **Traffic inspection and logging** via Palo Alto.
- **Point-to-Site VPN** for secure administrative access.
- **Internet-bound traffic control** through firewall rules and NAT.

This infrastructure lays the foundation for the next phase, where additional security tools (EDR, SIEM, Honeypot) will be introduced.


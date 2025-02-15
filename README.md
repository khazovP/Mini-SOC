# Azure Mini SOC Project (phase 1)

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

![Toopology1 drawio](https://github.com/user-attachments/assets/d239883c-b67f-42f1-a510-3f365b2244ed)

## Redesigned Topology

To accommodate the limitations, I structured the network as follows:

- **Virtual Network (VNet):** Named `Mini_SOC_VNet` with a **10.0.0.0/16** address space.
- **Subnets:**
  - `FW-Untrust` (10.0.1.0/24)
  - `FW-Trust` (10.0.2.0/24)
  - `DMZ` (10.0.20.0/24)
  - `Private1` (10.0.30.0/24)
  - `Private2` (10.0.40.0/24)

![Untitled Diagram drawio](https://github.com/user-attachments/assets/3859f7fe-2294-441d-add1-909736d4f8d7)

![1 vnet](https://github.com/user-attachments/assets/10ef18b8-9787-4615-80b7-9c89e656c4b0)

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
![3 pa ssh cropped](https://github.com/user-attachments/assets/9ce0f1d1-4bbb-4300-8849-0fe9ee994fba)

3. Log in to the Web UI and configure:
   - **Management Profile:** Allow SSH, HTTPS, and ICMP only from a trusted home IP.
   - Configure **DNS and NTP**.
   - **IP Allocation:** Currently, interfaces obtain IP addresses via DHCP, where Azure assigns IPs based on the interface settings of VM. However, we will later switch to static IPs to allow for customized service route configurations. Uncheck "Automatically create default route pointing to default gateway provided by server"
   - **Virtual Router:**
     - Default route (`eth1`) via **10.0.1.1** (Azure Gateway in FW-Untrust subnet)
     - Routes to **Private1** and **Private2** subnets via **eth2** (next-hop 10.0.2.1)
   ![4-6 VR](https://github.com/user-attachments/assets/99fb08bc-f7b7-4d2d-9bef-93876558a954)

![4-6 VR](https://github.com/user-attachments/assets/88c17c96-adda-423a-859e-91139105c1bf) 

### Step 3: Adjusting Network Interfaces

- **Stop VM and attach Untrust and Trust interfaces** with appropriate IPs.
- Create and attach **Network Security Group** allowing any traffic to aforemntioned interfaces.
- **Detach public IP** from the management interface and reassign it to **Untrust**.
- This configuration ensures:
  - The firewall is only accessible externally via **Untrust** from our home network.
  - The **Mgmt interface** remains private, reserving access for internal use or emergencies.

![7 vm int dhcp](https://github.com/user-attachments/assets/cdcfc131-85b6-420a-872d-9ce8bfbd09f7)

## Enforcing Firewall-Centric Routing

The next step is crucial: integrating our firewall into Azure's routing. Since this was my first time working with Azure, I encountered a challenge — Azure’s overlay network, by default, routes traffic directly between hosts across subnets. To ensure traffic inspection by firewall, we must manually redirect traffic through the firewall:

1. **Create a Route Table for `Private1` and `Private2`:**
   - **Within-VNet Rule:** Directs all `Mini_SOC_VNet` traffic (`/16`) to the firewall (Virtual Appliance).
   - **Within-Subnet Rule:** Ensures intra-subnet (`/24`) traffic remains local (VNetLocal) to prevent accidental redirection.
   - **Default Route:** Points to the internet via Azure Gateway (for initial SSH access, will be changed later).
![6-4 azure vm route private](https://github.com/user-attachments/assets/13eefb5b-80e1-4333-a039-5b5ee8e2ff35) 

2. **Create a Route Table for the DMZ:**
   - Similar configuration as `Private1`/`Private2`, but tailored for the DMZ subnet.
![9-1 azure route dmz](https://github.com/user-attachments/assets/9b5e6e84-8fb3-4e84-b076-b1d6f15ac72c)

## Testing Initial Connectivity

1. Deploy **two Debian VMs**, one in `Private1`, another in `Private2`.
2. SSH into both and attempt **inter-subnet communication**.
![8-2 debian ping cropped](https://github.com/user-attachments/assets/9c1326df-065c-45b3-9439-55cba67c79df)

4. Traffic logs confirm that packets are inspected and allowed due to the **default intrazone policy**.
![8-3 debian traffic](https://github.com/user-attachments/assets/792a2cf4-2f83-48d5-bb33-bdd169a817c5)

5. Move one of Debian VMs to `DMZ` and attempt connectivity:
   - Ping requests fail due to the **default deny rule** for different zones.
![9-4 dmz ping traffic](https://github.com/user-attachments/assets/9974fbd9-ad77-4bee-8b34-93f84a72ebcf)

   - Create an explicit firewall **policy** to allow `Private1` and `Private2` to reach `DMZ`.
![9-7 dmz ping policy](https://github.com/user-attachments/assets/ed43a19d-4de3-4de3-952b-f06d36be1b85)

   - Re-test, and observe the logs confirming allowed traffic between hosts in different zones.
![9-8 dmz ping traffic](https://github.com/user-attachments/assets/9b5dab23-fc05-48aa-bc31-f29838b38fa9)

## Enabling Internet Access via Firewall

Currently outbound internet traffic goes through Azure GW. To route outbound traffic through the firewall:
1. Modify **default routes** in **DMZ, Private1, and Private2** to use the firewall as the **next hop**.
![10-1 vpn routes](https://github.com/user-attachments/assets/7b2f0bae-d05b-4636-ba47-245c928495c4)

3. This introduces **asymmetric routing**, preventing SSH access to Debian hosts via public IPs.
4. Instead of configuring **GlobalProtect VPN (overkill for this use case)**, set up **Point-to-Site VPN** in Azure:
   - Detach public IPs from Debian VMs.
   - Create a **Gateway Subnet** in Mini Soc VNet and deploy a **Virtual Network Gateway**. Deploying may take ~20 minutes, let's make a coffee break :)
![10 vpn](https://github.com/user-attachments/assets/86520deb-b46c-4997-a066-467825bfd757)

   - Go to newly deployed **Virtual Network Gateway** -> **Point-to-Site VPN** and configure address pool and authentication (instructions on certificate generation can be found on internet).
   - Once connected, we can see new **subnet and routes** on our home machine.
![10 vpn3](https://github.com/user-attachments/assets/90561d50-ab4a-4709-8924-e8df30ab8f6b)

6. Update firewall and Azure routing:
   - Direct all **internet-bound traffic** through the firewall.
   - Edit routes in GatewaySubnet to ensure **VPN traffic** passes through the firewall before reaching Mini SOC subnets.
![10-3 vpn routes gateway](https://github.com/user-attachments/assets/39cbc281-69d8-400d-b44a-5c07a932a9c9)

   - Disable **gateway route propagation** for `Private1`, `Private2`, and `DMZ` but **enable it for FW-Trust subnet**. This will automatically create routes to remote VPN subnet in FW-Trust, where firewall egresses traffic to Trust zone.
![10-2 vpn route propagation](https://github.com/user-attachments/assets/1f253623-5d92-4824-bf69-a324af67e56f)

   - Add a **VPN subnet route** to the **firewall's Virtual Router**.
![10-4 pa vpn route](https://github.com/user-attachments/assets/73dd97c9-852a-4a01-8f1b-633e821161a0)

7. Update firewall policies:
   - Add policy for VPN traffic.
![10-6 vpn policies](https://github.com/user-attachments/assets/994e0970-d485-4b12-a000-bb085ee51d32)
   - Add NAT for VPN and internet access 
![10-7 vpn nat](https://github.com/user-attachments/assets/a4fc1122-aa2a-4472-a886-73292994856d)

### Testing VPN Access

Once VPN and FW are configured, we can test connectivity:
   - **Test Firewall connectivity** Log in to Palo via it's Mgmt interface private IP.
   - **SSH into a Debian VMs:** Use SSH to connect to a Debian VM using its private IP to ensure that traffic is properly routed through the VPN.
   - **Ping each other's private IP** to check connectivity between subnets.
   - **Test Internet Access via Firewall:** From the connected VPN client, perform a `curl ipinfo.io/ip` to confirm that VMs can go to internet via the firewall.

## Voila! our network is ready.
![10-8 test](https://github.com/user-attachments/assets/2096633d-bdf1-4a9c-a91e-a264a9ff1d21)


## Let's optimize security policies and finalize out setup

1. **Tighten Security Policies:**
   - Make rules **more granular**. They should be as strict as possible.
   - Set the **default intrazone policy** to **deny** instead of allow.
![11 refined policies](https://github.com/user-attachments/assets/b7319e21-9db8-47e1-b9f7-d1832844c291)

2. **Tighten NAT policies:**
![11 refined NAT](https://github.com/user-attachments/assets/1d220383-3553-4554-bb62-81d474c38f06)

3. **Restrict Firewall Management Access:**
   - Remove the **management profile** from `Untrust`.
   - Change Untrust interface IP to **static**. Configure service routes to use **Untrust interface** for necessary external services.
![11-3 service routes](https://github.com/user-attachments/assets/cbf1749e-f52c-4e88-8e4e-d2710045b598)

## Conclusion

This project successfully establishes a **firewall-protected Azure network** with:
- **Segmented subnets** enforcing strict security policies.
- **Traffic inspection and logging** via Palo Alto.
- **Point-to-Site VPN** for secure administrative access.
- **Internet-bound traffic control** through firewall rules and NAT.

This infrastructure lays the foundation for the next phase, where additional security tools (EDR, SIEM, Honeypot) will be introduced.


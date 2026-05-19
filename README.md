# Lab-02-Building-a-Secure-2-Tier-Web-Application-on-Azure
Lab 02 — Building a Secure 2-Tier Web Application on Azure. Covers IaaS architecture, VNet/subnet design, VM deployment, jump host SSH validation, NSG firewall hardening, and clean-up. Built to demonstrate both technical execution and cloud security fundamentals.

# 🔐 Lab 02 — Building a Secure 2-Tier Web Application on Azure

![Azure](https://img.shields.io/badge/Azure-IaaS%20Architecture-0078d4?style=flat&logo=microsoftazure&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Beginner-brightgreen?style=flat)
![Time](https://img.shields.io/badge/Estimated%20Time-60%20Minutes-blue?style=flat)
![Type](https://img.shields.io/badge/Type-IaaS%20%2F%20Networking%20%2F%20NSG-orange?style=flat)
![VMs](https://img.shields.io/badge/VMs-2%20Ubuntu%20Servers-purple?style=flat)

---

## 📋 Overview

This lab walks through the design and deployment of a classic **2-Tier IaaS (Infrastructure as a Service)** architecture on Microsoft Azure. Two Ubuntu virtual machines are provisioned inside a shared Virtual Network — one acting as a **public-facing web server** and one as a **private backend database server** — each isolated in their own dedicated subnet.

Network Security Groups (NSGs) are configured to enforce **least-privilege access**, ensuring the database tier is completely unreachable from the internet and only accepts traffic originating from the web subnet. Connectivity is validated using SSH and an internal `ping` test, mirroring how real-world multi-tier cloud environments are built, secured, and verified.

---

## 🏗️ Architecture Diagram

```
                ┌──────────────────────────────────────────────────────────────────┐
                │                    AZURE SUBSCRIPTION                            │
                │                                                                  │
                │  ┌────────────────────────────────────────────────────────────┐  │
                │  │          Resource Group: rg-lab02-[yourname]               │  │
                │  │                                                            │  │
                │  │  ┌──────────────────────────────────────────────────────┐  │  │
                │  │  │         Virtual Network: vnet-lab02                  │  │  │
                │  │  │              Address Space: 10.0.0.0/16              │  │  │
                │  │  │                                                      │  │  │
                │  │  │  ┌─────────────────────┐   ┌─────────────────────┐  │  │  │
                │  │  │  │   PUBLIC SUBNET      │   │   PRIVATE SUBNET    │  │  │  │
                │  │  │  │   snet-web           │   │   snet-db           │  │  │  │
                │  │  │  │   10.0.1.0/24        │   │   10.0.2.0/24       │  │  │  │
                │  │  │  │                      │   │                     │  │  │  │
                │  │  │  │  ┌────────────────┐  │   │  ┌──────────────┐  │  │  │  │
                │  │  │  │  │  vm-web-01     │  │   │  │  vm-db-01    │  │  │  │  │
                │  │  │  │  │  Ubuntu 20.04  │  │   │  │  Ubuntu 20.04│  │  │  │  │
                │  │  │  │  │  Standard_B1s  │  │   │  │  Standard_B1s│  │  │  │  │
                │  │  │  │  │                │  │   │  │              │  │  │  │  │
                │  │  │  │  │  ✅ Public IP  │  │   │  │  ❌ No Pub  │  │  │  │  │
                │  │  │  │  │  ✅ Port 80    │  │   │  │     IP      │  │  │  │  │
                │  │  │  │  │  ✅ Port 22    │  │   │  │             │  │  │  │  │
                │  │  │  │  │  [NSG-web]     │  │   │  │  [NSG-db]   │  │  │  │  │
                │  │  │  │  └───────┬────────┘  │   │  └──────▲──────┘  │  │  │  │
                │  │  │  └─────────┼────────────┘   └─────────┼─────────┘  │  │  │
                │  │  │            │                           │            │  │  │
                │  │  │            └───── Allow-Web-Subnet ───►│            │  │  │
                │  │  │                   10.0.1.0/24 → Any    │            │  │  │
                │  │  └──────────────────────────────────────────────────────┘  │  │
                │  └────────────────────────────────────────────────────────────┘  │
                └──────────────────────────────────────────────────────────────────┘
                              ▲                         ✖
                              │                         │
                        HTTP / SSH                BLOCKED ❌
                        (Allowed)             (No Public IP)
                              │
                    ┌─────────┴────────┐
                    │  🌐  Internet    │
                    └─────────────────┘
```

**Traffic Flow Summary:**
```
[Internet] ──► Public IP ──► vm-web-01 (snet-web) ──► INTERNAL ONLY ──► vm-db-01 (snet-db)
[Internet] ──► vm-db-01 direct ──► ❌ BLOCKED (No Public IP + NSG)
```

**Jump Host SSH Pattern:**
```
Your Computer ──SSH──► vm-web-01 (Public IP) ──ping──► vm-db-01 (Private IP: 10.0.2.x)
```

---

## 🛠️ Services & Concepts Used

| Service / Concept | Description |
|---|---|
| **Azure Virtual Network (VNet)** | Private isolated network that hosts all lab resources |
| **Subnets** | Network segments within the VNet that logically separate the web and database tiers |
| **Network Security Group (NSG)** | Virtual firewall attached to each VM controlling inbound and outbound traffic |
| **Azure Virtual Machine (IaaS)** | Compute instances running Ubuntu Server used as web and DB servers |
| **Public IP Address** | Internet-routable IP assigned only to the web VM — the DB VM has none |
| **SSH Key Pair (.pem / .ppk)** | Cryptographic key used to securely authenticate into Linux VMs |
| **Jump Host Pattern** | Security pattern where a public VM is the sole entry point to access private resources |
| **Least-Privilege Access** | Security principle: grant only the minimum network access required, enforced via NSG |
| **CIDR Notation** | IP addressing scheme used to define network ranges (e.g., `10.0.1.0/24`) |

---

## ✅ Prerequisites

Before starting this lab, ensure you have the following:

- [ ] An active **Azure Subscription** (Free Tier is sufficient)
- [ ] Access to the **Azure Portal** at [portal.azure.com](https://portal.azure.com)
- [ ] A **terminal** installed on your machine:
  - **Linux / macOS:** Built-in Terminal
  - **Windows:** [PuTTY](https://www.putty.org/) + [PuTTYgen](https://www.puttygen.com/) for SSH key conversion
- [ ] Completed **Week 2 Video Modules** (if following a course curriculum)

---

## 📌 Naming Conventions & IP Schema

| Resource | Name / Value |
|---|---|
| Resource Group | `rg-lab02-[yourname]` |
| Virtual Network | `vnet-lab02` |
| VNet Address Space | `10.0.0.0/16` |
| Web Subnet | `snet-web` → `10.0.1.0/24` |
| DB Subnet | `snet-db` → `10.0.2.0/24` |
| Web VM | `vm-web-01` |
| Database VM | `vm-db-01` |
| SSH Key Pair | `key-lab02` |
| NSG Allow Rule | `Allow-Web-Subnet` |
| Region | `East US` |

> 💡 **Why two subnets?** Even inside the same VNet, subnet isolation gives you the ability to apply different security rules per tier. It also mirrors real-world production architecture where web, app, and database layers are always segregated.

---

## 🚀 Step-by-Step Deployment

### Phase 1 — Build the Network Foundation

1. Log in to the [Azure Portal](https://portal.azure.com).
2. Search for **Virtual Networks** → click **+ Create**.
3. Fill in the **Basics** tab:
   - **Resource Group:** Click *Create new* → `rg-lab02-[yourname]`
   - **Name:** `vnet-lab02`
   - **Region:** `East US`
4. Navigate to the **IP Addresses** tab:
   - **Address space:** `10.0.0.0/16`
   - Add **Subnet 1:** Name `snet-web`, Range `10.0.1.0/24`
   - Add **Subnet 2:** Name `snet-db`, Range `10.0.2.0/24`
5. Click **Review + create** → **Create**.

---

### Phase 2 — Deploy the Web Server (Public Tier)

1. Search for **Virtual Machines** → click **+ Create**.
2. Fill in the **Basics** tab:
   - **Resource Group:** `rg-lab02-[yourname]`
   - **Name:** `vm-web-01`
   - **Region:** `East US`
   - **Image:** `Ubuntu Server 20.04 LTS`
   - **Size:** `Standard_B1s`
   - **Authentication type:** SSH public key
   - **Key pair name:** `key-lab02`
   - **Public inbound ports:** Allow selected → `HTTP (80)` and `SSH (22)`
3. Navigate to the **Networking** tab:
   - **Virtual Network:** `vnet-lab02`
   - **Subnet:** `snet-web`
   - **Public IP:** Create new (Standard SKU)
4. Click **Review + create** → **Create**.
5. When prompted, **download the private key** (`.pem` file) and store it securely.

> ⚠️ You cannot download the private key again after this step. Save it immediately.

---

### Phase 3 — Deploy the Database Server (Private Tier)

1. Search for **Virtual Machines** → click **+ Create**.
2. Fill in the **Basics** tab:
   - **Resource Group:** `rg-lab02-[yourname]`
   - **Name:** `vm-db-01`
   - **Region:** `East US`
   - **Image:** `Ubuntu Server 20.04 LTS`
   - **Size:** `Standard_B1s`
   - **Authentication type:** SSH public key → Use existing key → `key-lab02`
   - **Public inbound ports:** `SSH (22)`
3. Navigate to the **Networking** tab — ⚠️ **Critical configuration:**
   - **Virtual Network:** `vnet-lab02`
   - **Subnet:** Change to `snet-db`
   - **Public IP:** Select **None**
4. Click **Review + create** → **Create**.

> 🔐 **Why no Public IP?** Removing the public IP eliminates the internet exposure surface entirely. A database server has no reason to be internet-routable. The only valid path in is through the web server — this enforces the 2-tier security boundary at the infrastructure level.

---

### Phase 4 — Validate Connectivity (The Jump)

`vm-db-01` has no public IP, so it cannot be reached directly from your local machine. You must SSH into `vm-web-01` first and test connectivity to the database server from inside the VNet. This is the **Jump Host pattern**.

**Step 1 — Get the DB Server's Private IP**

1. Go to `vm-db-01` in the Azure Portal.
2. On the Overview page, copy the **Private IP address** (e.g., `10.0.2.5`).

**Step 2 — SSH into the Web Server**

*Linux / macOS:*
```bash
# Set correct permissions on your key
chmod 400 key-lab02.pem

# SSH into the web server
ssh -i key-lab02.pem azureuser@<PUBLIC-IP-OF-VM-WEB-01>
```

*Windows (PuTTY):*
1. Open **PuTTYgen** → click **Load** → set file filter to *All Files* → select your `.pem` file.
2. Click **Save private key** → save as `key-lab02.ppk`.
3. Open **PuTTY** → enter `vm-web-01`'s Public IP as the **Host Name**.
4. Go to **Connection → SSH → Auth** → Browse → select `key-lab02.ppk`.
5. Click **Open** → log in as `azureuser`.

**Step 3 — Test Internal Connectivity**

Once inside `vm-web-01`, ping the database server using its private IP:

```bash
ping 10.0.2.5
```

**Expected output (success):**
```
64 bytes from 10.0.2.5: icmp_seq=1 ttl=64 time=0.5 ms
64 bytes from 10.0.2.5: icmp_seq=2 ttl=64 time=1.1 ms
64 bytes from 10.0.2.5: icmp_seq=3 ttl=64 time=0.9 ms
```

Press `Ctrl + C` to stop.

**What a successful ping confirms:**
- Both VMs are inside the same VNet (`vnet-lab02`)
- Routing between subnets is correctly configured
- `vm-db-01` is placed in the correct subnet (`snet-db`)
- Your 2-tier architecture is connected and operational

> ✅ **Validated result:** 148 packets transmitted, 148 received, **0% packet loss**, all round-trip times under 3 ms.

---

### Phase 5 — Harden the Database Server (NSG Rule)

By default, internal VNet traffic flows freely between subnets. This phase adds an explicit NSG inbound rule to `vm-db-01` ensuring only the web subnet can communicate with it.

1. Navigate to `vm-db-01` in the Azure Portal.
2. Click **Networking** in the left menu.
3. Click on the attached **Network Security Group** (e.g., `vm-db-01-nsg`).
4. Select **Inbound security rules** → click **+ Add**.
5. Configure the rule:

| Field | Value |
|---|---|
| Source | `IP Addresses` |
| Source IP addresses / CIDR | `10.0.1.0/24` |
| Source port ranges | `*` |
| Destination | `Any` |
| Service | `Custom` |
| Destination port ranges | `*` |
| Protocol | `Any` |
| Action | `Allow` |
| Priority | `100` |
| Name | `Allow-Web-Subnet` |

6. Click **Add**.

**What this rule enforces:**
- Only `snet-web` (`10.0.1.0/24`) can initiate connections to `vm-db-01`
- All other subnets and all internet traffic remain blocked by default
- Together with the missing public IP, this implements **defense-in-depth** for the database tier

> 💡 **NSG Priority:** Lower number = higher priority. Priority `100` processes before Azure's default rules (which start at `65000`). No higher-priority deny rule should exist above `100` or it will override this allow.

---

## 🔍 Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| Ping from web VM to DB VM fails | VMs in different VNets | Verify both VMs show `vnet-lab02` under *Virtual network/subnet* in the portal |
| Ping fails but VNet is correct | `vm-db-01` in wrong subnet | Confirm `vm-db-01`'s private IP is in the `10.0.2.x` range. If it's `10.0.0.x` or `10.0.1.x`, it was placed in the wrong subnet |
| Ping fails and subnets are correct | NSG blocking ICMP | Check `vm-db-01-nsg` inbound rules. Ensure `Allow-Web-Subnet` exists at priority `100` with no conflicting deny above it |
| Cannot SSH into `vm-db-01` from home | No public IP — expected behavior | SSH into `vm-web-01` first (jump host), then test DB from there |
| SSH refused on `vm-web-01` | Wrong key, user, or port blocked | Confirm you're using `azureuser`, the correct key file, and port 22 is open in `vm-web-01`'s NSG |
| PuTTY fails to authenticate on Windows | Key not converted to `.ppk` format | Use PuTTYgen: Load `.pem` → Save private key → use `.ppk` in PuTTY Auth settings |

---

## 🧹 Clean Up Resources

> ⚠️ Running VMs accrue compute charges even when idle. Always clean up after completing a lab.

1. Navigate to **Resource Groups** in the Azure Portal.
2. Click on `rg-lab02-[yourname]`.
3. Click **Delete resource group** at the top.
4. Type the resource group name to confirm.
5. Click **Delete**.

This single action removes all dependent resources: both VMs, both NSGs, both network interfaces, both OS disks, the public IP, the SSH key pair, and the VNet — all in one operation.

---

## 💡 Key Takeaways

- **Subnet isolation** is foundational to multi-tier cloud architecture. Placing different workloads in separate subnets gives you the control plane to enforce traffic rules between tiers.
- **NSGs are stateful, layer-4 firewalls** evaluated by priority order. Lower number = higher priority. They are your primary network-level access control mechanism in Azure.
- **No public IP = no internet attack surface.** Removing a public IP from the database VM is the most effective single step to protect a backend resource.
- **The Jump Host pattern** is a real-world standard used in production environments to provide controlled, auditable access to private infrastructure without exposing it directly.
- **Least-privilege networking** means only allowing exactly what your architecture requires — the `Allow-Web-Subnet` NSG rule is a direct implementation of this principle.
- **SSH key hygiene matters.** Your `.pem` file is a credential. Treat it like a password — restrict permissions (`chmod 400`) and never share it.

---

## 📁 Repository Structure

```
lab-02-azure-2tier-webapp/
│
└── README.md          # This file — full lab documentation
```

---

## 📚 References

- [Azure Virtual Network Documentation](https://learn.microsoft.com/en-us/azure/virtual-network/)
- [Network Security Groups Overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Azure Virtual Machines — Linux](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/)
- [SSH Keys for Linux VMs in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed)
- [PuTTY & PuTTYgen Download](https://www.putty.org/)
- [Azure Free Account](https://azure.microsoft.com/en-us/free/)

---

*Lab authored by Jhante Charles | Azure Cloud Labs Series | Lab 02 of Series*

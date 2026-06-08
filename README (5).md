# Azure Virtual Network — Step-by-Step Setup Guide

> **Microsoft Azure Portal · Virtual Network Configuration**  
> Complete Step-by-Step Guide — everything done through the Azure Portal UI  
> Covers: Resource Group • VNet & Subnets • NSG Rules • Azure Bastion • Public IP • Virtual Machines • Connectivity Testing

---

## Table of Contents

1. [Overview & Architecture](#overview--architecture)
2. [Phase 1 — Create a Resource Group](#phase-1--create-a-resource-group)
3. [Phase 2 — Create the Virtual Network (VNet)](#phase-2--create-the-virtual-network-vnet)
4. [Phase 3 — Create Network Security Groups (NSGs)](#phase-3--create-network-security-groups-nsgs)
5. [Phase 4 — Deploy Azure Bastion](#phase-4--deploy-azure-bastion)
6. [Phase 5 — Create a Public IP Address](#phase-5--create-a-public-ip-address)
7. [Phase 6 — Deploy Virtual Machines](#phase-6--deploy-virtual-machines)
8. [Phase 7 — Validate Connectivity](#phase-7--validate-connectivity)
9. [GitHub Repository Structure](#github-repository-structure)

---

## Overview & Architecture

This guide walks through setting up a **three-tier Azure Virtual Network** entirely through the Azure Portal (`portal.azure.com`) — no command line needed. Resources created:

- **1 Resource Group** — container for all resources
- **1 Virtual Network (VNet)** with address space `10.0.0.0/16`
- **4 Subnets** — Web, App, Database tiers + AzureBastionSubnet
- **3 Network Security Groups (NSGs)** with inbound and outbound rules
- **1 Azure Bastion Host** — secure, browser-based SSH/RDP access (no public IP on VMs required)
- **1 Public IP address** — for the Web VM only
- **3 Virtual Machines** — one per subnet

> **Tip:** Work through each phase in order. Skipping ahead can cause dependency errors.

### Architecture Diagram

```
Internet  →  [nsg-web]  →  snet-web (10.0.1.0/24)  →  vm-web-01 (10.0.1.4)
                                  ↓ port 8080
                         [nsg-app]  →  snet-app (10.0.2.0/24)  →  vm-app-01 (10.0.2.4)
                                            ↓ port 5432
                                   [nsg-db]  →  snet-db (10.0.3.0/24)  →  vm-db-01 (10.0.3.4)

Azure Portal (HTTPS)  →  [Azure Bastion]  →  AzureBastionSubnet (10.0.4.0/27)  →  All VMs (private IPs)
```

---

## Phase 1 — Create a Resource Group

**Steps:**

1. Go to [portal.azure.com](https://portal.azure.com) and sign in.
2. Search **Resource Groups** → press Enter → click result.
3. Click **+ Create**.
4. Fill in the Basics tab:

   | Field | Value |
   |---|---|
   | Subscription | Select your Azure subscription |
   | Resource group name | `rg-networking-project` |
   | Region | `West Europe` |

5. Click **Review + create** → **Create**.
6. Wait for the notification: *"Resource group created successfully"*.

> ✅ `rg-networking-project` is ready.

---

## Phase 2 — Create the Virtual Network (VNet)

### Step 1 — Start VNet Creation

1. Search **Virtual Networks** → click **+ Create**.
2. Tabs shown: **Basics**, **Security**, **IP Addresses**, **Tags**, **Review + create**.

### Step 2 — Basics Tab

| Field | Value |
|---|---|
| Subscription | Your Azure subscription |
| Resource group | `rg-networking-project` |
| Name | `vnet-project-prod-westeurope` |
| Region | `West Europe` |

Click **Next: Security**.

### Step 3 — Security Tab (Enable Bastion Here)

> ⚠️ Enable Azure Bastion here — the most convenient place alongside VNet creation.

Toggle **Azure Bastion** to **Enabled** and fill in:

| Field | Value |
|---|---|
| Bastion host name | `bastion-project-prod` |
| AzureBastionSubnet address space | `10.0.4.0/27` |
| Public IP address | **Create new** → name `pip-bastion-prod` → OK |
| Bastion SKU | `Basic` |

Leave Firewall and DDoS Protection as default. Click **Next: IP Addresses**.

### Step 4 — IP Addresses Tab

1. Under **IPv4 address space**, clear and enter: `10.0.0.0/16`
2. Delete any default subnet (trash icon).
3. Click **+ Add a subnet** for each subnet below:

   **Web Subnet:**

   | Field | Value |
   |---|---|
   | Subnet name | `snet-web` |
   | Starting address | `10.0.1.0` |
   | Subnet size | `/24` |

   **App Subnet:**

   | Field | Value |
   |---|---|
   | Subnet name | `snet-app` |
   | Starting address | `10.0.2.0` |
   | Subnet size | `/24` |

   **Database Subnet:**

   | Field | Value |
   |---|---|
   | Subnet name | `snet-db` |
   | Starting address | `10.0.3.0` |
   | Subnet size | `/24` |

Final subnet summary:

| Subnet Name | CIDR Range | Purpose |
|---|---|---|
| `AzureBastionSubnet` | `10.0.4.0/27` | Azure Bastion host |
| `snet-web` | `10.0.1.0/24` | Web servers |
| `snet-app` | `10.0.2.0/24` | App / API layer |
| `snet-db` | `10.0.3.0/24` | Database tier |

> Azure reserves 5 IPs per subnet — a /24 gives **251 usable addresses**.

### Step 5 — Create the VNet

1. Click **Review + create** → verify settings → **Create**.
2. Wait for *"Deployment succeeded"* (Bastion deploys in background, ~5–10 min).

### Step 6 — Verify Subnets

1. Search **Virtual Networks** → click `vnet-project-prod-westeurope`.
2. Left menu → **Subnets** — all 4 subnets should appear.

> 📸 Screenshot this page for your report.

---

## Phase 3 — Create Network Security Groups (NSGs)

> ⚠️ Lower priority number = evaluated first. Priority `4096` must always be last.  
> ⚠️ Do **NOT** attach an NSG to `AzureBastionSubnet` — Microsoft prohibits this.

### Step 1 — Create NSG: `nsg-web`

1. Search **Network security groups** → **+ Create**:

   | Field | Value |
   |---|---|
   | Resource group | `rg-networking-project` |
   | Name | `nsg-web` |
   | Region | `West Europe` |

2. **Review + create** → **Create** → open `nsg-web`.

3. **Inbound security rules** → **+ Add** (one at a time):

   | Rule Name | Priority | Source/Dest | Protocol | Port | Action |
   |---|---|---|---|---|---|
   | `Allow-HTTP-Inbound` | 100 | Any (Internet) | TCP | 80 | **Allow** |
   | `Allow-HTTPS-Inbound` | 110 | Any (Internet) | TCP | 443 | **Allow** |
   | `Allow-SSH-Admin` | 120 | `10.0.0.0/16` | TCP | 22 | **Allow** |
   | `Deny-All-Inbound` | 4096 | Any | Any | Any | **Deny** |

4. **Outbound security rules** → **+ Add**:

   | Rule Name | Priority | Source/Dest | Protocol | Port | Action |
   |---|---|---|---|---|---|
   | `Allow-Web-to-App` | 100 | `10.0.2.0/24` | TCP | 8080 | **Allow** |
   | `Allow-SSH-to-VNet` | 110 | `10.0.0.0/16` | TCP | 22 | **Allow** |
   | `Allow-ICMP-to-VNet` | 120 | `10.0.0.0/16` | ICMP | * | **Allow** |
   | `Allow-Internet-Out` | 130 | `0.0.0.0/0` | TCP | 80,443 | **Allow** |
   | `Deny-All-Outbound` | 4096 | Any | Any | Any | **Deny** |

### Step 2 — Create NSG: `nsg-app`

1. **+ Create**:

   | Field | Value |
   |---|---|
   | Resource group | `rg-networking-project` |
   | Name | `nsg-app` |
   | Region | `West Europe` |

2. **Inbound security rules** → **+ Add**:

   | Rule Name | Priority | Source/Dest | Protocol | Port | Action |
   |---|---|---|---|---|---|
   | `Allow-From-Web` | 100 | `10.0.1.0/24` | TCP | 8080 | **Allow** |
   | `Allow-SSH-Admin` | 110 | `10.0.0.0/16` | TCP | 22 | **Allow** |
   | `Allow-ICMP-from-Web` | 120 | `10.0.1.0/24` | ICMP | * | **Allow** |
   | `Deny-All-Inbound` | 4096 | Any | Any | Any | **Deny** |

3. **Outbound security rules** → **+ Add**:

   | Rule Name | Priority | Source/Dest | Protocol | Port | Action |
   |---|---|---|---|---|---|
   | `Allow-App-to-DB` | 100 | `10.0.3.0/24` | TCP | 5432 | **Allow** |
   | `Allow-SSH-to-DB` | 110 | `10.0.3.0/24` | TCP | 22 | **Allow** |
   | `Deny-All-Outbound` | 4096 | Any | Any | Any | **Deny** |

### Step 3 — Create NSG: `nsg-db`

1. **+ Create**:

   | Field | Value |
   |---|---|
   | Resource group | `rg-networking-project` |
   | Name | `nsg-db` |
   | Region | `West Europe` |

2. **Inbound security rules** → **+ Add**:

   | Rule Name | Priority | Source/Dest | Protocol | Port | Action |
   |---|---|---|---|---|---|
   | `Allow-From-App` | 100 | `10.0.2.0/24` | TCP | 5432 | **Allow** |
   | `Allow-SSH-Admin` | 110 | `10.0.0.0/16` | TCP | 22 | **Allow** |
   | `Deny-Internet-In` | 200 | Internet | Any | Any | **Deny** |
   | `Deny-All-Inbound` | 4096 | Any | Any | Any | **Deny** |

> The DB NSG has **no outbound allow rules** — all outbound blocked by default (intentional for DB isolation).

### Step 4 — Associate Each NSG to Its Subnet

> ⚠️ Without this step, NSG rules have **no effect** on traffic.

1. Open `nsg-web` → **Subnets** → **+ Associate** → VNet: `vnet-project-prod-westeurope`, Subnet: `snet-web` → **OK**.
2. Open `nsg-app` → **Subnets** → **+ Associate** → `snet-app` → **OK**.
3. Open `nsg-db` → **Subnets** → **+ Associate** → `snet-db` → **OK**.

> ✅ All 3 NSGs protecting their subnets.  
> 📸 Screenshot each NSG's rules and subnet association pages.

---

## Phase 4 — Deploy Azure Bastion

> Secure, browser-based SSH/RDP — no public IPs needed on App or DB VMs.

Azure Bastion provides fully managed SSH and RDP over TLS directly from the Azure Portal.

### Option A — Verify Bastion from Phase 2

1. Search **Bastions** → confirm `bastion-project-prod` shows **Succeeded**.
2. Verify: VNet = `vnet-project-prod-westeurope`, Subnet = `AzureBastionSubnet`, Public IP = `pip-bastion-prod`.

If already **Succeeded**, skip to Phase 5.

### Option B — Deploy Bastion Manually (if skipped in Phase 2)

1. Search **Bastions** → **+ Create**.
2. Fill in:

   | Field | Value |
   |---|---|
   | Subscription | Your Azure subscription |
   | Resource group | `rg-networking-project` |
   | Name | `bastion-project-prod` |
   | Region | `West Europe` |
   | Tier / SKU | `Basic` |
   | Virtual network | `vnet-project-prod-westeurope` |
   | Subnet | `AzureBastionSubnet` (must be named exactly this) |
   | Public IP address | Create new → `pip-bastion-prod` → Standard → Static → OK |

3. Click **Review + create** → **Create**. Takes ~5–10 minutes.

### Why Azure Bastion?

| Without Bastion | With Bastion |
|---|---|
| Port 22 open to internet | Port 22 never exposed publicly |
| Need a jump box or VPN | Connect directly from Portal over HTTPS |
| Risk of brute-force attacks | Fully managed, auditable access |
| Complex admin firewall rules | Clean, session-logged connectivity |

> ✅ All VMs (including those with no public IP) accessible securely from the Portal.

---

## Phase 5 — Create a Public IP Address

> Only the Web VM gets a public IP — App and DB stay private.

1. Search **Public IP addresses** → **+ Create**.
2. Fill in:

   | Field | Value |
   |---|---|
   | Resource group | `rg-networking-project` |
   | Name | `pip-vm-web-01` |
   | Region | `West Europe` |
   | SKU | `Standard` (not Basic) |
   | IP version | `IPv4` |
   | IP address assignment | `Static` |

   > ⚠️ Always choose **Standard SKU** and **Static**. Basic SKU is being retired.

3. Click **Review + create** → **Create**.
4. Note the assigned IP for HTTP testing.

> ✅ Public IP `pip-vm-web-01` created. Attach to Web VM in Phase 6.

---

## Phase 6 — Deploy Virtual Machines

> One Ubuntu 24.04 VM per subnet — repeat 3 times.

| VM Name | Subnet | Private IP | Public IP | Role |
|---|---|---|---|---|
| `vm-web-01` | `snet-web` | `10.0.1.4` | `pip-vm-web-01` | Nginx web server |
| `vm-app-01` | `snet-app` | `10.0.2.4` | None | Node.js API |
| `vm-db-01` | `snet-db` | `10.0.3.4` | None | PostgreSQL database |

### Creating `vm-web-01` (Web Tier)

Search **Virtual Machines** → **+ Create** → **Azure virtual machine**.

**Basics Tab:**

| Field | Value |
|---|---|
| Resource group | `rg-networking-project` |
| Virtual machine name | `vm-web-01` |
| Region | `West Europe` |
| Availability options | No infrastructure redundancy required |
| Image | `Ubuntu Server 24.04 LTS - x64 Gen2` |
| Size | `Standard_F1ads_v7` (1 vcpu, 4 GiB RAM) |
| Authentication type | `Password` |
| Username | `azureuser` |
| Password | Strong password (min 12 chars, uppercase, number, symbol) |
| Public inbound ports | `None` — NSG handles this |

> ⚠️ Write your password down — Azure cannot retrieve it. Use the same password for all 3 VMs.

**Disks Tab:** OS disk = `Standard SSD`. Click **Next: Networking**.

**Networking Tab:**

| Field | Value |
|---|---|
| Virtual network | `vnet-project-prod-westeurope` |
| Subnet | `snet-web (10.0.1.0/24)` |
| Public IP | `pip-vm-web-01` |
| NIC network security group | `None` |

> ⚠️ NIC NSG must be `None` — a second NSG here conflicts with the subnet NSG.

Click **Review + create** → **Create**.

**Set Static Private IP:**

1. Go to `vm-web-01` → **Networking** → click the NIC name.
2. Left menu → **IP configurations** → `ipconfig1`.
3. Assignment: `Dynamic` → `Static`. IP: `10.0.1.4` → **Save**.

### Creating `vm-app-01` (App Tier)

| Field | Value |
|---|---|
| VM name | `vm-app-01` |
| Subnet | `snet-app (10.0.2.0/24)` |
| Public IP | `None` |
| NIC NSG | `None` |
| Static private IP | `10.0.2.4` |

> Access via Bastion (Portal → VM → Connect → Bastion), or from `vm-web-01`:
> ```bash
> ssh azureuser@10.0.2.4
> ```

### Creating `vm-db-01` (Database Tier)

| Field | Value |
|---|---|
| VM name | `vm-db-01` |
| Subnet | `snet-db (10.0.3.0/24)` |
| Public IP | `None` |
| NIC NSG | `None` |
| Static private IP | `10.0.3.4` |

> ✅ All 3 VMs deployed in their respective subnets.

---

## Phase 7 — Validate Connectivity

### Test 1 — Connect to All VMs via Azure Bastion

For **each** VM (`vm-web-01`, `vm-app-01`, `vm-db-01`):

1. **Virtual Machines** → click VM → **Connect** → **Bastion**.
2. Enter username `azureuser` and your password → **Connect**.

A browser-based terminal opens — no SSH client or exposed ports required.

> ✅ Bastion provides secure access to all VMs, even those with no public IP.

### Test 2 — Install Nginx on Web VM and Test HTTP

From `vm-web-01` (connected via Bastion):

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

From your local machine:

```bash
curl http://<Web-VM-Public-IP>
```

> ✅ Expected: HTML output from Nginx — port 80 inbound allowed by `nsg-web`.

### Test 3 — Ping App VM from Web VM

From `vm-web-01`:

```bash
ping 10.0.2.4 -c 4
```

> ✅ Expected: 0% packet loss — internal VNet routing between subnets works.

### Test 4 — Verify Web VM CANNOT Reach DB (NSG Blocking)

From `vm-web-01`:

```bash
nc -zv 10.0.3.4 5432
```

> ⚠️ Expected: **Connection timed out** — `nsg-db` blocks Web → DB on port 5432. This proves your security rules work.

### Test 5 — App VM CAN Reach DB VM

From `vm-web-01`, SSH to App VM:

```bash
ssh azureuser@10.0.2.4
```

Then from `vm-app-01`:

```bash
nc -zv 10.0.3.4 5432
```

> ✅ Expected: **Connection succeeded** — `nsg-db` allows port 5432 from the App subnet.

### Summary of Expected Results

| Test | From | To / Port | Expected | What It Proves |
|---|---|---|---|---|
| Bastion → Web VM | Azure Portal | `vm-web-01` (private) | **PASS** | Bastion works; no public SSH needed |
| Bastion → App VM | Azure Portal | `vm-app-01` (private) | **PASS** | Bastion covers VMs with no public IP |
| Bastion → DB VM | Azure Portal | `vm-db-01` (private) | **PASS** | Full Bastion coverage all tiers |
| HTTP to Web VM | Your browser | `Public IP:80` | **PASS** | `nsg-web` allows port 80 |
| Ping App from Web | `vm-web-01` | `10.0.2.4` ICMP | **PASS** | Internal VNet routing works |
| Web → DB port 5432 | `vm-web-01` | `10.0.3.4:5432` | **BLOCKED** | `nsg-db` denies Web subnet |
| App → DB port 5432 | `vm-app-01` | `10.0.3.4:5432` | **PASS** | `nsg-db` allows App subnet |
| Internet → App VM | Your computer | No public IP | **BLOCKED** | No public IP assigned |

> 📸 Screenshot every terminal result and every Azure Portal page.

---

## GitHub Repository Structure

```
azure-vnet-project/
├── README.md                             <- this file
├── docs/
│   ├── Azure_VNet_Project_Report.docx
│   └── screenshots/
│       ├── 01_resource_group.png
│       ├── 02_vnet_overview.png
│       ├── 03_subnets_list.png
│       ├── 04_bastion_overview.png
│       ├── 05_nsg_web_inbound.png
│       ├── 06_nsg_web_outbound.png
│       ├── 07_nsg_app_inbound.png
│       ├── 08_nsg_app_outbound.png
│       ├── 09_nsg_db_inbound.png
│       ├── 10_nsg_associations.png
│       ├── 11_public_ip.png
│       ├── 12_vm_overview.png
│       ├── 13_bastion_connect.png
│       ├── 14_ping_test.png
│       ├── 15_http_test.png
│       └── 16_nsg_block_confirmed.png
└── terraform/                            <- optional IaC bonus
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

> **Make your GitHub repo Public before submitting.**

---

> ✅ **Assignment complete!** You have built a production-style three-tier Azure VNet with NSG security controls, Azure Bastion for secure VM access, static IPs, and validated connectivity.

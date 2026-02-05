# Enterprise Multisite Active Directory Deployment (Windows Server 2022)

##  Project Overview
This project simulates a real-world enterprise infrastructure for a national company ("BoniTech") with three geographically separated branches. The goal was to design, deploy, and configure a **Multisite Active Directory Forest** using Windows Server 2022 and Hyper-V.

The infrastructure supports **2,000+ automated users**, segmented network traffic, and site-specific services (DHCP, DNS).

**Status:** ✅ Completed 

---

## 🏗️ Network Topology & Architecture
The network is divided into three physical sites connected via a software Router (RRAS). <!-- Changed it with PF sense -->

| Site Location | Subnet | Server Roles |
| :--- | :--- | :--- |
| **Winnipeg (HQ)** | `192.168.1.0/24` | Primary DC,File Server |
| **Montreal** | `192.168.2.0/24` | Secondary DC, File Server |
| **Calgary** | `192.168.3.0/24` | Secondary DC, File Server |

**Visual Topology:**
![Network Diagram](./docs/topology.png) <!-- Need To add a Screenshot-->

---

## 🛠️ Key Configurations

### 1. Virtualization & Networking (Hyper-V)
* **Hypervisor:** Windows Server 2022 Standard acting as the host.
* **Virtual Switching:** Configured three private vSwitches (Winnipeg, Montreal, Calgary) to isolate traffic.
* **Software Routing:** Deployed a **RRAS (Routing and Remote Access)** server with three interfaces to route traffic between the isolated subnets. <!-- Changed it with PF sense -->

### 2. Active Directory Domain Services (AD DS)
* **Forest Root:** `bonitech.int`
* **Replication:** Configured **Active Directory Sites and Services** to map subnets to their respective sites (WPG, MTL, CAL), ensuring efficient replication traffic.
* **OU Structure:** Designed a departmental hierarchy (Finance, HR, IT, Marketing, etc.) for organized Group Policy application.

### 3. Core Network Services
* **DHCP:** Deployed DHCP scopes for each site.
* **DHCP Relay:** Configured the Router (RRAS) to forward DHCP requests from client subnets (MTL/CAL) to the central DHCP server in Winnipeg. <!-- Changed it with PF sense -->
* **DNS:** Integrated with AD DS for seamless name resolution across sites.

---

## 💻 Automation: Bulk User Creation
To simulate a live enterprise environment, I wrote a **PowerShell script** to automatically generate 2,000 users from a CSV dataset. The script handles:
* Username generation (First letter of Firstname + Lastname).
* Duplicate checking (appends a number if the username exists).
* Automatic OU placement based on department.

**Snippet of the Script:**
```powershell
# Import users from CSV
$users = Import-Csv -Path $csvPath

foreach ($u in $users) {
    # Generate username (e.g., j.doe)
    $username = ($u.FirstName.Substring(0,1) + "." + $u.LastName).ToLower()
    $UPN = "$username@boniTech.int"

    # Check for duplicates and increment if necessary
    $counter = 1
    while (Get-ADUser -Filter {UserPrincipalName -eq $UPN} -ErrorAction SilentlyContinue) {
        $UPN = "$username$counter@boniTech.int"
        $counter++
    }

    # Create the User in the correct Department OU
    New-ADUser -Name "$($u.FirstName) $($u.LastName)" `
               -UserPrincipalName $UPN `
               -Department $u.Department `
               -Path $OUMap[$u.Department] `
               -Enabled $true
}

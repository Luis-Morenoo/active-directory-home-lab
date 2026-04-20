# 🏰 Active Directory Home Lab

> *Every morning at a Fortune 50 financial institution I'd log into my laptop and see "AD: Authenticating..." flash across the screen. I always wondered — what is that? Turns out it was Active Directory, the silent overlord of every corporate network, saying "You shall not pass" to anyone without valid credentials. So naturally, I built my own.*

A fully functional enterprise-style Active Directory environment built from scratch inside VirtualBox — Domain Controller, domain-joined client, Group Policy enforcement, PowerShell user management, and security audit logging.

---

## 🖥️ Lab Overview

| Component | Details |
|-----------|---------|
| **Domain** | `lab.local` / NetBIOS: `LAB` |
| **Domain Controller** | DC01 — Windows Server 2022 Standard |
| **Client Machine** | CLIENT01 — Windows 10 |
| **Hypervisor** | VirtualBox 7.2 |
| **Host Machine** | AMD Ryzen 7 5800XT, 16GB RAM |
| **DC01 IP** | 192.168.10.1 (static) |
| **CLIENT01 IP** | 192.168.10.10 (static) |
| **Network** | VirtualBox Internal Network `ADLab` |

---

## 🏗️ What Was Built

### Phase 1 — VM Setup & OS Installation
- Created DC01 (2GB RAM, 2 vCPUs, 50GB dynamic disk) in VirtualBox
- Configured dual network adapters: NAT (internet) + Internal Network `ADLab`
- Installed Windows Server 2022 Standard Evaluation with Desktop Experience
- Installed Windows 10 client (CLIENT01) on the same internal network

### Phase 2 — Static IP & Server Configuration
- Renamed server to `DC01` before promotion (critical — name baked into domain)
- Set static IP `192.168.10.1` on the Internal Network adapter
- Configured DNS to `127.0.0.1` — DC points to itself as DNS server

### Phase 3 — AD DS Role & Domain Promotion
- Installed Active Directory Domain Services role via Server Manager
- Dependencies installed: Group Policy Management, AD PowerShell Module, AD DS Snap-Ins
- Promoted DC01 to Domain Controller — created new forest `lab.local`
- DNS role auto-installed alongside AD DS

### Phase 4 — Domain Verification
```powershell
# Verified domain is fully operational
Get-ADDomain
# Returns: DNSRoot: lab.local, PDCEmulator: DC01.lab.local, NetBIOSName: LAB
```

### Phase 5 — CLIENT01 Domain Join
- Configured CLIENT01 with static IP and DNS pointing to DC01
- Verified connectivity: `ping 192.168.10.1` — 4/4 packets, 0% loss
- Joined CLIENT01 to `lab.local` — rebooted and logged in as `LAB\Administrator`
- Verified with `whoami` → returned `lab\administrator`

---

## 👥 Active Directory Structure

```
lab.local/
├── _USERS          → jsmith (John Smith), mlopez (Maria Lopez)
├── _ADMINS         → itadmin (IT Admin)
├── _GROUPS         → IT-Staff, HR-Staff, Helpdesk
└── _COMPUTERS      → CLIENT01
```

### Security Groups & Membership
| Group | Members |
|-------|---------|
| IT-Staff | itadmin |
| HR-Staff | jsmith |
| Helpdesk | mlopez |

---

## ⚡ PowerShell — User Lifecycle Management

```powershell
# Reset a user password
$pass = ConvertTo-SecureString "NewP@ss123" -AsPlainText -Force
Set-ADAccountPassword -Identity "jsmith" -Reset -NewPassword $pass

# Disable an account (offboarding)
Disable-ADAccount -Identity "jsmith"

# Re-enable an account
Enable-ADAccount -Identity "jsmith"

# Unlock a locked account
Unlock-ADAccount -Identity "jsmith"

# Check account status
Get-ADUser -Identity "jsmith" -Properties LockedOut, BadLogonCount |
    Select Name, Enabled, LockedOut, BadLogonCount

# Bulk create users from array
$users = @("tgrant","rchen","kpatel")
$password = ConvertTo-SecureString "P@ssword123" -AsPlainText -Force
foreach ($user in $users) {
    New-ADUser -Name $user -SamAccountName $user `
        -UserPrincipalName "$user@lab.local" `
        -Path "OU=_USERS,DC=lab,DC=local" `
        -AccountPassword $password -Enabled $true
}

# Find stale accounts (no login in 30+ days)
$cutoff = (Get-Date).AddDays(-30)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
    -Properties LastLogonDate | Select Name, LastLogonDate
```

---

## 📋 Group Policy Objects (GPOs)

### 1. Password Policy — linked to `lab.local`
| Setting | Value |
|---------|-------|
| Minimum password length | 10 characters |
| Password complexity | Enabled |
| Maximum password age | 90 days |
| Minimum password age | 30 days |
| Password history | 5 passwords remembered |

### 2. User Restrictions — linked to `_USERS` OU
Applies only to standard users. Admins are unaffected.

| Policy | State |
|--------|-------|
| Prohibit access to Control Panel | Enabled |
| Prevent access to command prompt | Enabled |
| Prevent access to registry editing tools | Enabled |

> ✅ **Verified:** Logged in as `jsmith` on CLIENT01 — Control Panel blocked with *"This operation has been cancelled due to restrictions in effect on this computer."* CMD blocked with *"The command prompt has been disabled by your administrator."*

### 3. Audit Policy — linked to `lab.local`
| Policy | Setting |
|--------|---------|
| Audit account logon events | Success, Failure |
| Audit account management | Success, Failure |
| Audit logon events | Success, Failure |
| Audit policy change | Success, Failure |

---

## 🔒 Security Audit Logging

### Simulated Brute Force Attack
```powershell
# Send 5 failed login attempts for jsmith
$i = 0
while ($i -lt 5) {
    net use \\DC01\IPC$ /user:LAB\jsmith wrongpassword 2>$null
    $i++
}
```

### Query Failed Logins (Event ID 4625)
```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} |
    Select -First 5 | Format-List TimeCreated, Message
```

### Critical Event IDs
| Event ID | Meaning | Security Significance |
|----------|---------|----------------------|
| 4624 | Successful logon | Baseline for anomaly detection |
| 4625 | Failed logon | Rapid failures = brute force |
| 4634 | Account logoff | Session tracking |
| 4720 | User account created | Unauthorized creation = insider threat |
| 4740 | Account locked out | Lockout trigger — often an attack |

> **Result:** Event Viewer captured 4,199 security events including the simulated failed logins. Event ID 4625 entries showed the exact account targeted, failure reason, source IP, and workstation — identical to what a SOC analyst reads in a real Splunk alert.

---

## 🛠️ Skills Demonstrated

| Area | Details |
|------|---------|
| IT Infrastructure | Built enterprise domain network from scratch in VirtualBox |
| Active Directory | Domain creation, OU design, user/group management, domain join |
| Windows Server | Server Manager, role installation, DNS, static IP configuration |
| PowerShell | AD module — user creation, password reset, bulk operations |
| Group Policy | GPO creation, OU-scoped linking, policy verification |
| Networking | Static IPs, DNS resolution, internal network design |
| Security & Auditing | Audit policy, Event Viewer analysis, brute force simulation |
| Troubleshooting | DNS failures, domain join errors, firewall rules |
| Virtualization | VirtualBox VMs, snapshots, multi-VM networking |
| IAM | Least privilege, delegation, account lockout, access control |

---

## 🔮 Future Expansions

- [ ] Second Domain Controller (DC02) for redundancy and failover
- [ ] DHCP Server role — dynamic IP assignment
- [ ] File Server with shared drives and NTFS permissions
- [ ] Splunk Universal Forwarder — ship Event Logs to SIEM
- [ ] Kali Linux VM — AD enumeration and attack simulation
- [ ] Fine-Grained Password Policies per security group
- [ ] Remote Desktop Services — RDP access control

---

## 📄 Documentation

Full technical write-up available in the `docs/` folder:
- `AD_Lab_Technical_WriteUp.pdf` — formatted project documentation with screenshots
- `AD_Lab_Technical_WriteUp.docx` — editable Word version

---

## 👤 About

**Luis Moreno** — Computer Engineer | IT Infrastructure & Cybersecurity

Pursuing MS in Computer Science (Cybersecurity concentration) at Arizona State University.  
Actively targeting IT Support, Data Center, and Cybersecurity roles.

- 🔗 [LinkedIn](https://linkedin.com/in/luis-moreno11)
- 🌐 [Portfolio](https://luis-morenoo.github.io)
- 💻 [GitHub](https://github.com/Luis-Morenoo)

---

*Built session by session, question by question — not from a tutorial, but from actually understanding why each step mattered. Every error was a lesson. Every working command was a win.*

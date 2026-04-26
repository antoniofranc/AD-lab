# 🛡️ Enterprise Active Directory Deployment & IAM Security Lab

**Project Report – Windows Server 2025 | Azure | PowerShell Automation**

## 📌 1. Project Overview
This project demonstrates the full deployment of an enterprise-grade Active Directory Domain Services (AD DS) environment using Windows Server 2025 hosted on Microsoft Azure.

It focuses on:

- Automated AD DS installation  
- Domain Controller promotion  
- Organizational structure design  
- Bulk identity provisioning  
- Security governance (GPO, PSO, RBAC)  
- Real-world troubleshooting  
- Verification and auditing  

This lab simulates how modern organizations build and secure identity infrastructure, making it ideal for IAM, SysAdmin, and Security Engineering portfolios.

---

## 🏗️ 2. Infrastructure Deployment

### 2.1 Server Role Installation
Automated installation of AD DS and management tools:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### 2.2 Forest & Domain Creation
Promoted the server to a Domain Controller and created a new forest:

```powershell
Install-ADDSForest -DomainName "adlab.local" -DomainNetbiosName "ADLAB" -InstallDNS:$true
```

### 2.3 Networking Configuration

- Configured DNS loopback (127.0.0.1)  
- Ensured ADWS and DNS services initialized correctly  
- Validated domain reachability post-promotion  

---

## 🛠️ 3. Troubleshooting & Resolution Log (Real-World Issues)

### Issue 1 — NetBIOS Name Collision
**Problem:** Forest promotion failed due to hostname = domain NetBIOS name.  

**Fix:** Renamed server to DC01 and used a unique NetBIOS name (ADLAB).

---

### Issue 2 — DNS Resolution Failure (ADWS Not Starting)
**Problem:** ADWS service failed because DNS was pointing to Azure DNS instead of local DC.  

**Fix:** Forced DNS to loopback:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

Rebooted → ADWS initialized successfully.

---

### Issue 3 — Authentication Context Change
**Problem:** Could not log in via RDP after promotion.

**Fix:** Recognized shift from local → domain authentication.

Logged in using:

```text
ADLAB\Administrator
```

---

## 🧩 4. Directory Architecture & Identity Provisioning

### 4.1 Organizational Unit Structure

Created a clean, department-based OU hierarchy:

- IT  
- HR  
- Finance  
- Contractors  
- Admins  

All OUs protected from accidental deletion.

---

### 4.2 Bulk User Creation (15 Users)

Automated provisioning using PowerShell:

- Unique UPNs  
- Department attributes  
- Forced password change at first logon  
- OU-based placement  

---

### 4.3 Security Groups

Created 1:1 departmental security groups:

- GRP_IT  
- GRP_HR  
- GRP_Finance  
- GRP_Contractors  
- GRP_Admins  

Users were automatically assigned based on OU.

---

## 🔐 5. Security Governance & Hardening

### 5.1 Group Policy Objects (GPOs)

| Department | Policy |
|-----------|--------|
| IT | PowerShell Script Block Logging |
| HR | 10-minute screen lock |
| Finance | 10-minute screen lock |
| Contractors | Disable Control Panel & CMD |
| Admins | Script Block Logging |

---

### 5.2 Fine-Grained Password Policies (PSOs)

| Group | Min Length | Max Age | Notes |
|------|------------|---------|-------|
| Admins | 16 chars | 30 days | Strictest |
| Contractors | 12 chars | 30 days | High-risk users |
| Standard Users | 10 chars | 90 days | Balanced |

---

## 🧑‍💼 6. Role-Based Access Control (RBAC)

Implemented a professional delegation model using Set-Acl and dsacls.

### Helpdesk (IT)
- Reset passwords domain-wide  
- Create/delete users in Finance  
- Manage HR user properties  

### HR
- Full control over HR OU only  

### Finance
- Read-only access (compliance model)  

### Contractors
- Explicit deny on write operations  

### Admins
- Full control (GenericAll) over all OUs  
- Nested into built-in Domain Admins  

This models real enterprise least-privilege IAM.

---

## 🧪 7. Verification & Auditing

Automated validation included:

- ✔️ OU ACL audits  
- ✔️ Group membership integrity  
- ✔️ Domain Admins membership check  
- ✔️ PSO application verification  
- ✔️ GPO link validation  

All checks passed successfully.

---

## 🖼️ 8. Architecture Diagram

```markdown
![AD DS Architecture Diagram](https://github.com/user-attachments/assets/90f43ff0-6add-4a22-b8da-28326786660f)
```

---

## 🧭 9. Key Takeaways

- Built a fully automated, enterprise-style AD DS environment  
- Implemented real IAM governance (PSO, RBAC, GPO)  
- Resolved real-world AD DS deployment issues  
- Designed a scalable OU and security group structure  
- Demonstrated PowerShell mastery across identity lifecycle  

---

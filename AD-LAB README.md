<img width="1536" height="1000" alt="image" src="https://github.com/user-attachments/assets/90f43ff0-6add-4a22-b8da-28326786660f" />


# 🛡️ IAM Lab: Active Directory Domain Controller Setup (Windows Server 2025)

## 📌 Project Overview
This project documents the complete setup of an **Identity and Access Management (IAM)** lab using **Microsoft Active Directory Domain Services (AD DS)**. The environment is hosted on an **Azure VM running Windows Server 2025 Datacenter**. This serves as my IAM project, covering the installation of AD DS, promotion to a Domain Controller, and creation of Organizational Units (OUs) and test users.

**Objective:** To understand the fundamental components of AD DS and automate identity management using PowerShell.

---

## 🖥️ Environment Specifications
| Component | Details |
| :--- | :--- |
| **Cloud Platform** | Microsoft Azure |
| **Operating System** | Windows Server 2025 Datacenter - x64 Gen2 |
| **Server Role** | Domain Controller (DC) |
| **Domain Name** | `ad.contoso.com` |
| **Tooling** | Server Manager, PowerShell, Active Directory Administrative Center |

---

## 📂 Lab Structure
- **Step 1:** Install AD DS Role
- **Step 2:** Promote Server to Domain Controller
- **Step 3:** Create Organizational Units (OUs)
- **Step 4:** Create Users (Bulk)
- **Step 5:** Verification & Management

---

Phase 1: Install AD DS Role
## Install the AD DS role and management tools
```
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```
## Verify the installation was successful
```
Get-WindowsFeature -Name AD-Domain-Services
```
<img width="1478" height="314" alt="image" src="https://github.com/user-attachments/assets/8f6a4b83-0c2b-49aa-b01e-a44f89473b1c" />

<img width="1189" height="176" alt="image" src="https://github.com/user-attachments/assets/254fa874-5d91-423e-9e6c-7fffc49474fb" />

----

Phase 2: Promote Server to Domain Controller (Create New Domain)
## Import the deployment module
```
Import-Module ADDSDeployment
```

## Promote the server to a Domain Controller and create a new forest/domain
## Replace "yourdomain.com" with your actual domain name (e.g., "adlab.local")
```
Install-ADDSForest `
    -DomainName "yourdomain.com" `
    -DomainNetbiosName "YOURDOMAIN" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -CreateDnsDelegation:$false `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
    -Force:$true
```
⚠️ The server will automatically restart after this command completes. After reboot, log in as YOURDOMAIN\Administrator.

----
Phase 3: Create Organizational Units (OUs)

## Import the Active Directory module
```
Import-Module ActiveDirectory
```
## Store your domain's Distinguished Name for reuse
```
$domainDN = (Get-ADDomain).DistinguishedName
# Example result: "DC=yourdomain,DC=com"
```
## Create the 5 required OUs
```
New-ADOrganizationalUnit -Name "IT"          -Path $domainDN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "HR"          -Path $domainDN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Finance"     -Path $domainDN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Contractors" -Path $domainDN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Admins"      -Path $domainDN -ProtectedFromAccidentalDeletion $true
```
## Verify all OUs were created
```
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```
<img width="900" height="500" alt="image" src="https://github.com/user-attachments/assets/7fb403ab-d68e-4f35-9ae6-0eba0787791e" />

----

Phase 4: Create 15 Test Users Across OUs
## Automated PowerShell Code (Creating Bulk Users)
```
Import-Module ActiveDirectory
$domainDN = (Get-ADDomain).DistinguishedName
$domain = (Get-ADDomain).DNSRoot
$password = ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force

$users = @(
    @{ First="Alice"; Last="Thompson"; OU="IT" },
    @{ First="Brian"; Last="Martinez"; OU="IT" },
    @{ First="Carol"; Last="Stevens"; OU="IT" },
    @{ First="Diana"; Last="Walker"; OU="HR" },
    @{ First="Edward"; Last="Collins"; OU="HR" },
    @{ First="Fiona"; Last="Bennett"; OU="HR" },
    @{ First="George"; Last="Harris"; OU="Finance" },
    @{ First="Hannah"; Last="Clark"; OU="Finance" },
    @{ First="Ivan"; Last="Lewis"; OU="Finance" },
    @{ First="Julia"; Last="Young"; OU="Contractors" },
    @{ First="Kevin"; Last="King"; OU="Contractors" },
    @{ First="Laura"; Last="Wright"; OU="Contractors" },
    @{ First="Michael"; Last="Scott"; OU="Admins" },
    @{ First="Nancy"; Last="Adams"; OU="Admins" },
    @{ First="Oscar"; Last="Baker"; OU="Admins" }
)

foreach ($user in $users) {
    $firstName = $user.First
    $lastName = $user.Last
    $ouName = $user.OU
    $fullName = "$firstName $lastName"
    $samAccount = ($firstName.Substring(0,1) + $lastName).ToLower()
    $upn = "$samAccount@$domain"
    $ouPath = "OU=$ouName,$domainDN"

    New-ADUser `
        -GivenName $firstName `
        -Surname $lastName `
        -Name $fullName `
        -SamAccountName $samAccount `
        -UserPrincipalName $upn `
        -Path $ouPath `
        -AccountPassword $password `
        -Enabled $true `
        -ChangePasswordAtLogon $true `
        -Department $ouName `
        -Description "Test user – $ouName OU"
    
    Write-Host "Created: $fullName ($samAccount) in OU=$ouName" -ForegroundColor Green
}

# Final Verification
Write-Host "`nVerification Table:" -ForegroundColor Yellow
Get-ADUser -Filter * -Properties Department | Where-Object { $_.Department } | Select-Object Name, SamAccountName, Department | Sort-Object Department
```
<img width="514" height="434" alt="image" src="https://github.com/user-attachments/assets/11f1c06e-8be1-44e9-9e8f-93ba333a80db" />

----

Phase 6: Create Security Groups & Assign Users
## Step 1: Create Security Groups in Each OU
```
Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName

# Define groups — one per OU, Global scope, Security type
$groups = @(
    @{ Name="GRP_IT";          OU="IT";          Description="IT Department Staff" },
    @{ Name="GRP_HR";          OU="HR";          Description="HR Department Staff" },
    @{ Name="GRP_Finance";     OU="Finance";     Description="Finance Department Staff" },
    @{ Name="GRP_Contractors"; OU="Contractors"; Description="External Contractors" },
    @{ Name="GRP_Admins";      OU="Admins";      Description="Domain Administrators Staff" }
)

foreach ($group in $groups) {
    New-ADGroup `
        -Name           $group.Name `
        -GroupScope     Global `
        -GroupCategory  Security `
        -Path           "OU=$($group.OU),$domainDN" `
        -Description    $group.Description

    Write-Host "Created group: $($group.Name) in OU=$($group.OU)" -ForegroundColor Green
}

# Verify groups were created
Get-ADGroup -Filter * | Select-Object Name, GroupScope, GroupCategory | Sort-Object Name | Format-Table -AutoSize
```

<img width="517" height="191" alt="image" src="https://github.com/user-attachments/assets/1e89a4a0-ca5c-4959-9461-488081e39b3d" />

## Step 2: Add Users to Their Respective Groups
```
Import-Module ActiveDirectory
$domainDN = (Get-ADDomain).DistinguishedName

# Map each OU's users into their group automatically
$ouGroupMap = @{
    "IT"          = "GRP_IT"
    "HR"          = "GRP_HR"
    "Finance"     = "GRP_Finance"
    "Contractors" = "GRP_Contractors"
    "Admins"      = "GRP_Admins"
}

foreach ($ou in $ouGroupMap.Keys) {
    $groupName = $ouGroupMap[$ou]
    $ouPath = "OU=$ou,$domainDN"
    
    # Get all users specifically in this OU
    $users = Get-ADUser -Filter * -SearchBase $ouPath
    
    foreach ($user in $users) {
        # Add user to the group
        Add-ADGroupMember -Identity $groupName -Members $user
        Write-Host "Added $($user.SamAccountName) → $groupName" -ForegroundColor Cyan
    }
}

# Verify membership counts
Write-Host "`nGroup Membership Summary:" -ForegroundColor Yellow
foreach ($group in $ouGroupMap.Values) {
    $count = (Get-ADGroupMember -Identity $group).Count
    Write-Host "$group has $count members" -ForegroundColor Green
}
```
<img width="1095" height="699" alt="image" src="https://github.com/user-attachments/assets/9afb5343-b4d8-4abf-8197-588021889620" />

## Step 3: Add Admins Group to Domain Admins
```
Import-Module ActiveDirectory

# Add GRP_Admins to the built-in Domain Admins group
Add-ADGroupMember -Identity "Domain Admins" -Members "GRP_Admins"

Write-Host "Success: GRP_Admins now has Domain Admin privileges!" -ForegroundColor Yellow

# Verify the membership
Get-ADGroupMember -Identity "Domain Admins" | Select-Object Name, SamAccountName | Format-Table -AutoSize
```
<img width="338" height="153" alt="image" src="https://github.com/user-attachments/assets/d606ac67-b91c-4ca0-862f-3f01ffef02f2" />

----
Phase 7: Configure Group Policy Objects (GPOs)

## Step 1: Create GPOs for Each OU
```
Import-Module GroupPolicy
Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName

# Define the GPOs to match your OU structure
$gpos = @(
    @{ Name="GPO_IT_Policy"; OU="IT" },
    @{ Name="GPO_HR_Policy"; OU="HR" },
    @{ Name="GPO_Finance_Policy"; OU="Finance" },
    @{ Name="GPO_Contractors_Policy"; OU="Contractors" },
    @{ Name="GPO_Admins_Policy"; OU="Admins" }
)

foreach ($gpo in $gpos) {
    # 1. Create the GPO
    New-GPO -Name $gpo.Name -Comment "Policy for $($gpo.OU) OU"
    
    # 2. Link the GPO to the specific OU path
    New-GPLink `
        -Name $gpo.Name `
        -Target "OU=$($gpo.OU),$domainDN" `
        -Enforced No
    
    Write-Host "Created and linked: $($gpo.Name) -> OU=$($gpo.OU)" -ForegroundColor Green
}

# Final Verification of GPOs
Write-Host "`nGPO Deployment Summary:" -ForegroundColor Yellow
Get-GPO -All | Where-Object { $_.DisplayName -like "GPO_*" } | Select-Object DisplayName, GpoStatus, CreationTime | Format-Table -AutoSize
```

<img width="784" height="215" alt="image" src="https://github.com/user-attachments/assets/9db91802-e1ee-4b4a-9345-f04bb8130151" />

## Step 2: Apply Password Policy (Domain-Wide)
```
Import-Module GroupPolicy
Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName

# 1. Create the GPO
New-GPO -Name "GPO_Domain_PasswordPolicy" -Comment "Domain-wide password policy"

# 2. Link it to the domain root
New-GPLink -Name "GPO_Domain_PasswordPolicy" -Target $domainDN

# 3. Set a registry-based policy setting (Example: Password Age)
Set-GPRegistryValue -Name "GPO_Domain_PasswordPolicy" `
    -Key "HKLM\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" `
    -ValueName "MaximumPasswordAge" `
    -Value 90 `
    -Type DWord

Write-Host "Domain password policy GPO created and linked to $domainDN" -ForegroundColor Green
```

<img width="679" height="303" alt="image" src="https://github.com/user-attachments/assets/95e4edc1-2d59-44bd-83cd-e59d27c71f89" />

## Step 3: Configure Fine-Grained Password Policies Per Group
```
Import-Module ActiveDirectory

# --- Standard users (IT, HR, Finance) ---
New-ADFineGrainedPasswordPolicy `
    -Name "PSO_StandardUsers" `
    -Precedence 20 `
    -MinPasswordLength 10 `
    -PasswordHistoryCount 12 `
    -MaxPasswordAge "90.00:00:00" `
    -MinPasswordAge "1.00:00:00" `
    -LockoutThreshold 5 `
    -LockoutObservationWindow "00:30:00" `
    -LockoutDuration "00:30:00" `
    -ComplexityEnabled $true `
    -Description "Password policy for standard domain users"

Add-ADFineGrainedPasswordPolicySubject -Identity "PSO_StandardUsers" -Subjects "GRP_IT", "GRP_HR", "GRP_Finance"
Write-Host "PSO_StandardUsers applied to GRP_IT, GRP_HR, GRP_Finance" -ForegroundColor Cyan

# --- Contractors (Stricter, 30-day expiry) ---
New-ADFineGrainedPasswordPolicy `
    -Name "PSO_Contractors" `
    -Precedence 15 `
    -MinPasswordLength 12 `
    -MaxPasswordAge "30.00:00:00" `
    -LockoutThreshold 3 `
    -ComplexityEnabled $true `
    -Description "Strict password policy for contractors"

Add-ADFineGrainedPasswordPolicySubject -Identity "PSO_Contractors" -Subjects "GRP_Contractors"
Write-Host "PSO_Contractors applied to GRP_Contractors" -ForegroundColor Cyan

# --- Admins (Strictest, 16 characters) ---
New-ADFineGrainedPasswordPolicy `
    -Name "PSO_Admins" `
    -Precedence 10 `
    -MinPasswordLength 16 `
    -MaxPasswordAge "30.00:00:00" `
    -LockoutThreshold 3 `
    -ComplexityEnabled $true `
    -Description "Strictest password policy for admin accounts"

Add-ADFineGrainedPasswordPolicySubject -Identity "PSO_Admins" -Subjects "GRP_Admins"
Write-Host "PSO_Admins applied to GRP_Admins" -ForegroundColor Cyan

# Final Verification
Write-Host "`nApplied Password Settings Objects (PSOs):" -ForegroundColor Yellow
Get-ADFineGrainedPasswordPolicy -Filter * | Select-Object Name, Precedence, MinPasswordLength | Sort-Object Precedence
```

<img width="596" height="140" alt="image" src="https://github.com/user-attachments/assets/29660c48-1166-4e0e-9e44-76a8f834fe54" />
<img width="1141" height="530" alt="image" src="https://github.com/user-attachments/assets/999191d0-76c0-4420-86f9-108a153f85b9" />

## Step 4: Configure GPO Security Settings Per OU
```
Import-Module GroupPolicy

# --- CONTRACTORS: Restrict Control Panel & CMD ---
Set-GPRegistryValue `
    -Name "GPO_Contractors_Policy" `
    -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
    -ValueName "NoControlPanel" `
    -Type DWord -Value 1

Set-GPRegistryValue `
    -Name "GPO_Contractors_Policy" `
    -Key "HKCU\Software\Policies\Microsoft\Windows\System" `
    -ValueName "DisableCMD" `
    -Type DWord -Value 1
Write-Host "Contractor restrictions applied (No Control Panel, No CMD)." -ForegroundColor Yellow

# --- HR & FINANCE: 10-Minute Screen Lock ---
foreach ($gpoName in @("GPO_HR_Policy","GPO_Finance_Policy")) {
    $desktopKey = "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop"
    Set-GPRegistryValue -Name $gpoName -Key $desktopKey -ValueName "ScreenSaveTimeOut" -Type String -Value "600"
    Set-GPRegistryValue -Name $gpoName -Key $desktopKey -ValueName "ScreenSaverIsSecure" -Type String -Value "1"
    Set-GPRegistryValue -Name $gpoName -Key $desktopKey -ValueName "SCRNSAVE.EXE" -Type String -Value "scrnsave.scr"
}
Write-Host "Screen lock policy (600s) applied to HR and Finance." -ForegroundColor Yellow

# --- IT & ADMINS: PowerShell Script Block Logging ---
foreach ($gpoName in @("GPO_IT_Policy","GPO_Admins_Policy")) {
    Set-GPRegistryValue `
        -Name $gpoName `
        -Key "HKLM\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" `
        -ValueName "EnableScriptBlockLogging" `
        -Type DWord -Value 1
}
Write-Host "PowerShell logging enabled for IT and Admins." -ForegroundColor Yellow

Run gpupdate /force in a command prompt to manually pull the latest changes
```

<img width="674" height="566" alt="image" src="https://github.com/user-attachments/assets/58279de3-0558-4974-bb23-d023ec0b9ae9" />

----
Phase 8: Role-Based Access Control (RBAC)
## Step 1: Delegate OU Management to Each Group
```
Import-Module ActiveDirectory
$domainDN = (Get-ADDomain).DistinguishedName

# Standard AD Extended Rights GUIDs
$guidMap = @{
    "Reset Password"            = [GUID]"00299570-246d-11d0-a768-00aa006e0529"
    "User Account Restrictions" = [GUID]"4c164200-20c0-11d0-a768-00aa006e0529"
    "Create User Objects"       = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
}

# The Delegation Helper Function
function Set-OUDelegation {
    param(
        [string]$OUName,
        [string]$GroupName,
        [string]$Right,
        [string]$AccessType = "Allow"
    )
    $ouPath = "OU=$OUName,$domainDN"
    $ouACL = Get-Acl "AD:\$ouPath"
    $group = Get-ADGroup $GroupName
    $groupSID = New-Object System.Security.Principal.SecurityIdentifier($group.SID)
    
    # Create the rule
    $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
        $groupSID,
        [System.DirectoryServices.ActiveDirectoryRights]$Right,
        [System.Security.AccessControl.AccessControlType]$AccessType,
        [System.DirectoryServices.ActiveDirectorySecurityInheritance]"All"
    )
    
    $ouACL.AddAccessRule($ace)
    Set-Acl "AD:\$ouPath" $ouACL
    Write-Host "Delegated [$Right] on OU=$OUName to $GroupName" -ForegroundColor Cyan
}
```
## Step 2: IT Group — Full Control Over All OUs (Helpdesk Role)
IT staff can read/write user properties and reset passwords across all OUs:
```
# Let IT Reset Passwords in HR
Set-OUDelegation -OUName "HR" -GroupName "GRP_IT" -Right "ExtendedRight"

# Let IT Create/Delete users in Finance
Set-OUDelegation -OUName "Finance" -GroupName "GRP_IT" -Right "CreateChild, DeleteChild"

Write-Host "`nRBAC Delegation Complete! IT can now manage HR and Finance." -ForegroundColor Yellow
```
<img width="767" height="46" alt="image" src="https://github.com/user-attachments/assets/97beed94-dfc6-4779-8940-89749f49de63" />

## Step 3: HR Group — Manage Only HR OU
HR can create, modify, and delete users  but only inside their own OU:
```
# 1. HR can create and delete user objects in HR OU
Set-OUDelegation -OUName "HR" -GroupName "GRP_HR" `
    -Right "CreateChild, DeleteChild" -AccessType "Allow"

# 2. HR can read and write user properties in HR OU
Set-OUDelegation -OUName "HR" -GroupName "GRP_HR" `
    -Right "GenericWrite, ReadProperty" -AccessType "Allow"

Write-Host "Success: GRP_HR now has administrative control over OU=HR." -ForegroundColor Green
```
<img width="713" height="35" alt="image" src="https://github.com/user-attachments/assets/30e6d4b1-b94c-4c68-bb87-60ca99f212c7" />

## Step 4: Finance Group — Read Only (Compliance Role)
Finance staff can view but not modify AD objects:
```
# Finance gets read-only access to Finance OU
Set-OUDelegation -OUName "Finance" -GroupName "GRP_Finance" `
    -Right "ReadProperty" -AccessType "Allow"

Write-Host "Success: GRP_Finance granted read-only access to OU=Finance" -ForegroundColor Green
```
<img width="1446" height="52" alt="image" src="https://github.com/user-attachments/assets/6473f915-5bc1-46df-a814-e70caf85d67b" />

## Step 5: Contractors — Explicitly Denied Write Access
Contractors can log in but cannot modify anything in AD

```
# 1. Deny write to Contractors on their own OU
Set-OUDelegation -OUName "Contractors" -GroupName "GRP_Contractors" `
    -Right "WriteProperty" -AccessType "Deny"

# 2. Deny creating or deleting objects
Set-OUDelegation -OUName "Contractors" -GroupName "GRP_Contractors" `
    -Right "CreateChild, DeleteChild" -AccessType "Deny"

Write-Host "GRP_Contractors explicitly denied write access on OU=Contractors" -ForegroundColor Red
```
<img width="1450" height="72" alt="image" src="https://github.com/user-attachments/assets/e293556e-f066-479d-b93d-4c3316db53af" />

## Step 6: Admins Group — Full AD Delegation
Admins get full control over all OUs explicitly (on top of Domain Admins membership):
```
$ous = @("IT", "HR", "Finance", "Contractors", "Admins")

foreach ($ou in $ous) {
    # GenericAll is the "God Mode" permission for an OU
    Set-OUDelegation -OUName $ou -GroupName "GRP_Admins" `
        -Right "GenericAll" -AccessType "Allow"
    
    Write-Host "GRP_Admins granted GenericAll on OU=$ou" -ForegroundColor Magenta
}

Write-Host "`nAll RBAC Phases Complete! Your domain structure is fully secured and delegated." -ForegroundColor Yellow
```
<img width="1438" height="97" alt="image" src="https://github.com/user-attachments/assets/34ce9d2f-b051-4c26-8bf8-d5e9cab7d82b" />


## Step 7: Use Built-in Delegation Wizard via PowerShell
This uses the official `dsacls` tool to delegate the Reset Password right — a very common real-world task:

```
$domainDN = (Get-ADDomain).DistinguishedName
$domain = (Get-ADDomain).NetBIOSName

# 1. IT can reset passwords in ALL standard OUs
$ous = @("IT","HR","Finance","Contractors")
foreach ($ou in $ous) {
    $ouDN = "OU=$ou,$domainDN"
    # /G grants permissions. CA stands for Control Access (required for Reset Password)
    dsacls $ouDN /G "$domain\GRP_IT:CA;Reset Password;user" | Out-Null
    Write-Host "GRP_IT can reset passwords in OU=$ou" -ForegroundColor Cyan
}

# 2. HR can also reset passwords specifically in their own OU
dsacls "OU=HR,$domainDN" /G "$domain\GRP_HR:CA;Reset Password;user" | Out-Null
Write-Host "GRP_HR can reset passwords in OU=HR" -ForegroundColor Cyan

# 3. Admins can reset passwords everywhere
foreach ($ou in @("IT","HR","Finance","Contractors","Admins")) {
    $ouDN = "OU=$ou,$domainDN"
    dsacls $ouDN /G "$domain\GRP_Admins:CA;Reset Password;user" | Out-Null
}
Write-Host "GRP_Admins can reset passwords in ALL OUs" -ForegroundColor Magenta

```
<img width="1257" height="56" alt="image" src="https://github.com/user-attachments/assets/ebbbaeee-522c-4b3f-881e-188f6c98b979" />

## Step 8: Verification

```
Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$ous = @("IT","HR","Finance","Contractors","Admins")

# 1. Review ACL on each OU — verifies who has delegated permissions
foreach ($ou in $ous) {
    Write-Host "`n[ACL AUDIT] OU=$ou" -ForegroundColor Yellow
    $ouPath = "AD:\OU=$ou,$domainDN"
    Get-Acl $ouPath | Select-Object -ExpandProperty Access | 
        Where-Object { $_.IdentityReference -like "*GRP_*" } | 
        Select-Object IdentityReference, ActiveDirectoryRights, AccessControlType | 
        Format-Table -AutoSize
}

# 2. Confirm departmental group memberships are still intact
$groups = @("GRP_IT","GRP_HR","GRP_Finance","GRP_Contractors","GRP_Admins")
foreach ($group in $groups) {
    Write-Host "`n[MEMBERSHIP] $group" -ForegroundColor Cyan
    Get-ADGroupMember -Identity $group | Select-Object Name, SamAccountName | Format-Table -AutoSize
}

# 3. Confirm Domain Admins still contains GRP_Admins
Write-Host "`n[PRIVILEGE CHECK] Domain Admins Group" -ForegroundColor Magenta
Get-ADGroupMember -Identity "Domain Admins" | Select-Object Name, ObjectClass | Format-Table -AutoSize

Write-Host "`n✅ LAB COMPLETE: All Users, Groups, and Delegated Permissions verified." -ForegroundColor Green
```
<img width="1466" height="298" alt="image" src="https://github.com/user-attachments/assets/e3752361-c36f-4797-b7c9-0434a9cfdad0" />
















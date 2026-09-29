# Entra ID Automation Lab — Microsoft Graph PowerShell

## Overview

This lab introduced Microsoft Graph PowerShell for automating common Entra ID identity-management tasks.

The main scenario simulated an HR onboarding workflow where employee information is supplied through a CSV file. PowerShell imports the employee data, generates account information, creates Entra ID users, and assigns each user to the appropriate security group based on department.

### Workflow

```text
HR CSV
   ↓
Import-Csv
   ↓
PowerShell objects
   ↓
Generate username / UPN
   ↓
Microsoft Graph
   ↓
Create Entra ID user
   ↓
Set user attributes
   ↓
Read employee department
   ↓
Map department to security group
   ↓
Add user to group
   ↓
Verify in Entra ID
```

---

# 1. PowerShell 7 and Microsoft Graph Setup

Verify the installed PowerShell version:

```powershell
$PSVersionTable.PSVersion
```

Install the Microsoft Graph PowerShell SDK for the current user:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
```

Connect to Microsoft Graph with permissions to manage users and groups:

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All"
```

Verify the current Graph connection:

```powershell
Get-MgContext
```

---

# 2. Query Entra ID Users

Retrieve all Entra ID users:

```powershell
Get-MgUser -All | Select-Object DisplayName, UserPrincipalName, Id
```

Microsoft Graph does not always return every user property by default. Properties such as `Department` can be explicitly requested:

```powershell
Get-MgUser -All -Property DisplayName,UserPrincipalName,Department |
    Select-Object DisplayName, UserPrincipalName, Department
```

---

# 3. Query Entra ID Groups

Find a security group by its display name:

```powershell
$salesGroup = Get-MgGroup -Filter "displayName eq 'SG-Sales'"
```

Inspect the returned group:

```powershell
$salesGroup
```

Retrieve the group's Object ID:

```powershell
$salesGroup.Id
```

---

# 4. Manual User Creation

Before automating bulk provisioning, I created an individual user manually through Microsoft Graph.

Define a temporary password profile:

```powershell
$passwordProfile = @{
    Password = "REPLACE-WITH-TEMPORARY-PASSWORD"
    ForceChangePasswordNextSignIn = $true
}
```

Create the user:

```powershell
$params = @{
    DisplayName       = "Marcus Lee"
    GivenName         = "Marcus"
    Surname           = "Lee"
    UserPrincipalName = "marcus.lee@YOURTENANT.onmicrosoft.com"
    MailNickname      = "marcus.lee"
    AccountEnabled    = $true
    PasswordProfile   = $passwordProfile
    Department        = "Sales"
}

$newUser = New-MgUser @params
```

The returned user object can then be accessed through `$newUser`.

For example:

```powershell
$newUser.Id
$newUser.DisplayName
$newUser.UserPrincipalName
```

---

# 5. Add a User to a Security Group

Retrieve the target group:

```powershell
$salesGroup = Get-MgGroup -Filter "displayName eq 'SG-Sales'"
```

Add the user using the Entra Object IDs of the group and user:

```powershell
New-MgGroupMember -GroupId $salesGroup.Id -DirectoryObjectId $newUser.Id
```

This demonstrated an important Graph concept:

```text
User Object
   │
   └── User.Id
          ↓
New-MgGroupMember
          ↑
   ┌── Group.Id
   │
Group Object
```

Graph operations frequently work with Object IDs rather than only human-readable names.

---

# 6. Create the HR CSV

The bulk provisioning scenario used the following CSV structure:

```csv
FirstName,LastName,Department
Daniel,Kim,Sales
Sofia,Martinez,HR
Ethan,Williams,Sales
Priya,Patel,IT
```

Example project location:

```text
C:\Labs\EntraAutomation\
```

The CSV can also be generated directly from PowerShell:

```powershell
@"
FirstName,LastName,Department
Daniel,Kim,Sales
Sofia,Martinez,HR
Ethan,Williams,Sales
Priya,Patel,IT
"@ | Set-Content ".\newhires.csv"
```

Verify the file:

```powershell
Get-Content ".\newhires.csv"
```

---

# 7. Import CSV Data into PowerShell

Import the employee records:

```powershell
$employees = Import-Csv ".\newhires.csv"
```

View the imported objects:

```powershell
$employees
```

Each CSV row becomes a PowerShell object.

For example:

```powershell
$employee.FirstName
$employee.LastName
$employee.Department
```

The CSV exists permanently on disk, while `$employees` is a PowerShell variable populated when `Import-Csv` reads the file.

---

# 8. Generate User Principal Names

Define the tenant domain:

```powershell
$domain = "YOURTENANT.onmicrosoft.com"
```

Generate UPNs from the CSV:

```powershell
foreach ($employee in $employees) {

    $mailNickname = "$($employee.FirstName).$($employee.LastName)".ToLower()
    $upn = "$mailNickname@$domain"

    Write-Host $upn
}
```

Example result:

```text
daniel.kim@YOURTENANT.onmicrosoft.com
sofia.martinez@YOURTENANT.onmicrosoft.com
ethan.williams@YOURTENANT.onmicrosoft.com
priya.patel@YOURTENANT.onmicrosoft.com
```

---

# 9. Department-to-Security-Group Mapping

A PowerShell hash table maps HR department names to Entra security groups:

```powershell
$departmentGroups = @{
    "IT"    = "SG-IT"
    "Sales" = "SG-Sales"
    "HR"    = "SG-HR"
}
```

Example lookup:

```powershell
$departmentGroups["Sales"]
```

Result:

```text
SG-Sales
```

This allows the provisioning workflow to determine group membership automatically from the employee's department.

---

# 10. Bulk User Creation

The following loop generates account information and creates each employee in Entra ID:

```powershell
foreach ($employee in $employees) {

    $displayName = "$($employee.FirstName) $($employee.LastName)"
    $mailNickname = "$($employee.FirstName).$($employee.LastName)".ToLower()
    $upn = "$mailNickname@$domain"

    Write-Host "Creating $displayName..."

    $params = @{
        DisplayName       = $displayName
        GivenName         = $employee.FirstName
        Surname           = $employee.LastName
        UserPrincipalName = $upn
        MailNickname      = $mailNickname
        AccountEnabled    = $true
        PasswordProfile   = $passwordProfile
        Department        = $employee.Department
    }

    $newUser = New-MgUser @params

    Write-Host "Created: $($newUser.UserPrincipalName)"
}
```

---

# 11. Automated Security-Group Assignment

After the users existed, the CSV was processed again to find each user and assign the correct security group.

```powershell
foreach ($employee in $employees) {

    $mailNickname = "$($employee.FirstName).$($employee.LastName)".ToLower()
    $upn = "$mailNickname@$domain"

    $user = Get-MgUser -Filter "userPrincipalName eq '$upn'"

    $groupName = $departmentGroups[$employee.Department]
    $group = Get-MgGroup -Filter "displayName eq '$groupName'"

    Write-Host "Adding $($user.DisplayName) to $groupName..."

    New-MgGroupMember -GroupId $group.Id -DirectoryObjectId $user.Id

    Write-Host "SUCCESS: $($user.DisplayName) -> $groupName"
}
```

Example:

```text
Daniel Kim
   ↓
Department = Sales
   ↓
$departmentGroups["Sales"]
   ↓
SG-Sales
   ↓
Get SG-Sales Object ID
   ↓
Get Daniel's Object ID
   ↓
Add Daniel to SG-Sales
```

---

# 12. Verify Group Membership

Retrieve the Sales group:

```powershell
$salesGroup = Get-MgGroup -Filter "displayName eq 'SG-Sales'"
```

Retrieve its members and resolve each member as an Entra user:

```powershell
Get-MgGroupMember -GroupId $salesGroup.Id -All |
    ForEach-Object {
        Get-MgUser -UserId $_.Id -Property DisplayName,UserPrincipalName,Department
    } |
    Select-Object DisplayName, UserPrincipalName, Department
```

The results can also be verified manually through:

```text
Entra Admin Center
→ Groups
→ SG-Sales
→ Members
```

---

# 13. Reusable Provisioning Script

The individual exercises can be combined into a reusable provisioning script.

Example filename:

```text
New-EntraUsers.ps1
```

```powershell
# ==========================================
# Entra ID Bulk User Provisioning
# ==========================================

# Configuration

$domain = "YOURTENANT.onmicrosoft.com"

$departmentGroups = @{
    "IT"    = "SG-IT"
    "Sales" = "SG-Sales"
    "HR"    = "SG-HR"
}

$passwordProfile = @{
    Password = "REPLACE-WITH-TEMPORARY-PASSWORD"
    ForceChangePasswordNextSignIn = $true
}

# Import HR data

$employees = Import-Csv ".\newhires.csv"

# Process employees

foreach ($employee in $employees) {

    $displayName = "$($employee.FirstName) $($employee.LastName)"
    $mailNickname = "$($employee.FirstName).$($employee.LastName)".ToLower()
    $upn = "$mailNickname@$domain"

    Write-Host "Processing $displayName..."

    # Check whether the user already exists

    $existingUser = Get-MgUser -Filter "userPrincipalName eq '$upn'"

    if ($existingUser) {
        Write-Host "SKIPPED: $upn already exists."
        continue
    }

    # Build user properties

    $params = @{
        DisplayName       = $displayName
        GivenName         = $employee.FirstName
        Surname           = $employee.LastName
        UserPrincipalName = $upn
        MailNickname      = $mailNickname
        AccountEnabled    = $true
        PasswordProfile   = $passwordProfile
        Department        = $employee.Department
    }

    try {

        # Create user

        $newUser = New-MgUser @params

        Write-Host "CREATED: $upn"

        # Determine department security group

        $groupName = $departmentGroups[$employee.Department]

        if ($groupName) {

            $group = Get-MgGroup -Filter "displayName eq '$groupName'"

            if ($group) {

                New-MgGroupMember `
                    -GroupId $group.Id `
                    -DirectoryObjectId $newUser.Id

                Write-Host "GROUP: $displayName -> $groupName"
            }
            else {
                Write-Host "WARNING: Group $groupName was not found."
            }
        }
        else {
            Write-Host "WARNING: No group mapping exists for $($employee.Department)."
        }

    }
    catch {

        Write-Host "ERROR: Failed to provision $displayName"
        Write-Host $_
    }

    Write-Host "-----------------------------"
}
```

---

# 14. Safe Workflow Before Running Automation

Verify the current directory:

```powershell
Get-Location
```

List the files:

```powershell
Get-ChildItem
```

Inspect the incoming employee CSV:

```powershell
Import-Csv ".\newhires.csv" | Format-Table
```

Then execute the provisioning script:

```powershell
.\New-EntraUsers.ps1
```

The script checks Entra ID for an existing UPN before attempting account creation.

If the account already exists:

```text
Processing Daniel Kim...
SKIPPED: daniel.kim@YOURTENANT.onmicrosoft.com already exists.
```

This prevents the script from blindly attempting to recreate existing users.

---

# Key Concepts Learned

### PowerShell Objects

Commands such as:

```powershell
$user = Get-MgUser ...
```

return objects containing properties.

Properties can be accessed using dot notation:

```powershell
$user.Id
$user.DisplayName
$user.UserPrincipalName
```

### Variables

Variables store values or objects for later use:

```powershell
$domain
$upn
$user
$group
$newUser
```

### `foreach`

Repeats an operation for every object in a collection:

```powershell
foreach ($employee in $employees) {
    # Perform operation
}
```

### `if`

Controls what happens based on a condition:

```powershell
if ($existingUser) {
    # User already exists
}
```

### `continue`

Stops processing the current loop iteration and moves to the next employee:

```powershell
continue
```

### Hash Tables

Store key/value mappings:

```powershell
$departmentGroups = @{
    "Sales" = "SG-Sales"
}
```

### Splatting

Instead of passing many parameters directly to a command, parameters can be stored in a hash table:

```powershell
$params = @{
    DisplayName = $displayName
    GivenName   = $employee.FirstName
}
```

Then passed to the command:

```powershell
New-MgUser @params
```

### `Write-Host`

Displays status information in the PowerShell console:

```powershell
Write-Host "Creating user..."
```

It does not modify Entra ID.

### `try` / `catch`

Allows the script to handle errors and display controlled failure information rather than stopping without useful feedback.

---

# Main Takeaways

This lab demonstrated that PowerShell can automate repetitive Entra ID administration rather than requiring every user to be manually created through the Entra Admin Center.

The important relationship is:

```text
Employee data
     ↓
PowerShell object
     ↓
Entra user object
     ↓
Object ID
     ↓
Security group membership
```

The PowerShell syntax itself relies on familiar programming concepts such as variables, objects, properties, loops, conditionals, and dictionaries/hash tables.

The Microsoft-specific knowledge is understanding:

- Entra users
- User Principal Names
- Entra Object IDs
- Security groups
- Group membership
- User attributes
- Microsoft Graph permissions
- How these objects relate to each other

## Future Improvements

Future versions of this automation could add:

- License assignment
- User updates
- Offboarding / account disabling
- Group removal
- More robust input validation
- Logging to CSV
- Secure temporary-password generation
- Script parameters
- Functions
- Additional error handling
- Automated verification/reporting
- App-based authentication for unattended automation

> **Security note:** Real credentials, passwords, secrets, tenant-specific sensitive information, and authentication tokens should never be committed to a public GitHub repository.

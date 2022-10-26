---
tags: active-directory azure-ad dynamic groups powershell
---

One of my favorite features in Azure AD is dynamic groups. We can simply manage members of a group by defining filter rules based on user attributes. We can then go ahead and assign Azure AD resources to those groups, whether that's apps, licenses or memberships / ownerships for other groups. This comes in very handy if we synchronize employee data from an HR system. The only thing we need to worry about is the sync. Group membership and access to resources is completely automated. Nice! :blush: However, many organizations still use Active Directory to manage their users and resources. Wouldn't it be great to have the same functionality there? Say no more! :sunglasses:

## The Scenario

Let's assume we have the following users in our AD which are also synced to Azure AD.

| UserPrincipalName       | Department | Title             |
| ----------------------- | ---------- | ----------------- |
| john.doe@contoso.com    | Marketing  | UI/UX Designer    |
| sam.smith@contoso.com   | Marketing  | Visual Designer   |
| tom.tonkins@contoso.com | Marketing  | Head of Marketing |

Now we would like to dynamically add them to the following security groups based on their attributes.
* `role-department-marketing`: All members of the marketing department
* `role-title-designer`: All designers of the org

Let's see how we can achieve that in Azure AD and AD.

## Azure AD
We simply create the two security groups with dynamic filter rules set in parameter `MembershipRule`. I prefer using the `Microsoft.Graph.Groups` PowerShell module. However, you can achieve the same thing with the `AzureAD` module or via the web GUI.

```powershell
# role-department-marketing
$Params = @{
    DisplayName                   = "role-department-marketing"
    MailNickname                  = "role-department-marketing"
    Description                   = "Dynamically managed, containing all members of the marketing department"
    MailEnabled                   = $false
    SecurityEnabled               = $true
    GroupTypes                    = "DynamicMembership"
    MembershipRule                = '(user.department -eq "Marketing")'
    MembershipRuleProcessingState = "On"
}
New-MgGroup @Params

# role-title-designer
$Params = @{
    DisplayName                   = "role-title-designer"
    MailNickname                  = "role-title-designer"
    Description                   = "Dynamically managed, containing all designers of the org"
    MailEnabled                   = $false
    SecurityEnabled               = $true
    GroupTypes                    = "DynamicMembership"
    MembershipRule                = '(user.jobTitle -contains "Designer")'
    MembershipRuleProcessingState = "On"
}
New-MgGroup @Params
```

Let's verify the group members.

> :information_source: **Info**  
> Keep in mind that processing of the rules can take up to 30 minutes depending on the size of your Azure AD, as stated in this [Microsoft article](https://learn.microsoft.com/en-gb/azure/active-directory/enterprise-users/groups-troubleshooting).  

```powershell
# role-department-marketing
Get-AzureADGroup -Filter "DisplayName eq 'role-department-marketing'" `
    | Get-AzureADGroupMember `
    | Select-Object UserPrincipalName, Department, JobTitle

UserPrincipalName           Department     JobTitle
-----------------           ----------     --------
john.doe@contoso.com        Marketing      UI/UX Designer
sam.smith@contoso.com       Marketing      Visual Designer
tom.tonkins@contoso.com     Marketing      Head of Marketing
```

```powershell
# role-title-designer
Get-AzureADGroup -Filter "DisplayName eq 'role-title-designer'" `
    | Get-AzureADGroupMember `
    | Select-Object UserPrincipalName, Department, JobTitle

UserPrincipalName           Department     JobTitle
-----------------           ----------     --------
john.doe@contoso.com        Marketing      UI/UX Designer
sam.smith@contoso.com       Marketing      Visual Designer
```
Look's about right :thumbsup:


## Active Directory
Now let's do the same in AD. As there is no built-in functionality for dynamic groups, I have created the script [Sync-DynamicAdGroupMember.ps1](https://github.com/dominikduennebacke/Sync-DynamicAdGroupMember). Let's see what it does.

> **.DESCRIPTION**  
> The Sync-DynamicAdGroupMember.ps1 loops thru all AD groups that have a Get-ADUser filter query defined on a speficied extensionAttribute.
> The script then fetches all AD users that match the query and syncs them with the group's members.
> This means missing members are added and obsolete members are removed.
> Manual changes to the members of the group are overwritten.

Right on. So, first we create our groups.
```powershell
# role-department-marketing
$Params = @{
    Name          = "role-department-marketing"
    Description   = "Dynamically managed, containing all members of the marketing department"
    GroupCategory = "Security"
    GroupScope    = "Universal"
    Path          = "OU=groups,DC=contoso,DC=com"
}
New-ADGroup @Params

# role-title-designer
$Params = @{
    Name          = "role-title-designer"
    Description   = "Dynamically managed, containing all designers of the org"
    GroupCategory = "Security"
    GroupScope    = "Universal"
    Path          = "OU=groups,DC=contoso,DC=com"
}
New-ADGroup @Params
```

Then we set a Get-ADUser filter query on `extensionAttribute10` of the groups.
```powershell
# role-department-marketing
Set-ADGroup -Identity "role-department-marketing" -Replace @{
    extensionAttribute10 = "department -eq 'Marketing'"
}

# role-title-designer
Set-ADGroup -Identity "role-title-designer" -Replace @{
    extensionAttribute10 = "title -like '*Designer*'"
}
```

Now we download the script.
```powershell
$Params = @{
    Uri     = "https://raw.githubusercontent.com/dominikduennebacke/Sync-DynamicAdGroupMember/main/Sync-DynamicAdGroupMember.ps1"
    OutFile = "Sync-DynamicAdGroupMember.ps1"
}
Invoke-WebRequest @Params
```
And run it - providing integer `10` for parameter `ExtensionAttribute`. This tells the script that the Get-ADUser filter can be found on this attribute of our AD groups. You can interchange that to a different number in case you already use `extensionAttribute10` for a different purpose.
> :warning: **Warning**  
> * When you run the script make sure you comply with the [requirements](https://github.com/dominikduennebacke/Sync-DynamicAdGroupMember#REQUIREMENTS)
> * As with any script from the internet, use it at your own risk and inspect the source code before execution

```powershell
./Sync-DynamicAdGroupMember.ps1 -ExtensionAttribute 10 -VERBOSE

VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching AD groups with a value in extensionAttribute10
VERBOSE: Syncing group members
VERBOSE: role-department-marketing: department -eq 'Marketing'
VERBOSE: role-department-marketing: (+) john.doe
VERBOSE: role-department-marketing: (+) sam.smith
VERBOSE: role-department-marketing: (+) tom.tonkins
VERBOSE: role-title-designer: title -like '*Designer*'
VERBOSE: role-title-designer: (+) john.doe
VERBOSE: role-title-designer: (+) sam.smith
```

Let's verify the group members.
```powershell
# role-department-marketing
Get-ADGroupMember -Identity "role-department-marketing" `
    | Get-ADUser -Properties * `
    | Select-Object UserPrincipalName, Department, Title

UserPrincipalName           Department     Title
-----------------           ----------     -----
john.doe@contoso.com        Marketing      UI/UX Designer
sam.smith@contoso.com       Marketing      Visual Designer
tom.tonkins@contoso.com     Marketing      Head of Marketing
```
```powershell
# role-title-designer
Get-ADGroupMember -Identity "role-title-designer" `
    | Get-ADUser -Properties * `
    | Select-Object UserPrincipalName, Department, Title

UserPrincipalName           Department     Title
-----------------           ----------     -----
john.doe@contoso.com        Marketing      UI/UX Designer
sam.smith@contoso.com       Marketing      Visual Designer
```

Et voilà, we achieved the same result as in Azure AD and hence have successfully replicated the dynamic group feature in AD :muscle:

## Scheduling
What's missing? Continuous integration. Just like in Azure AD the script needs to run on a regular basis, ideally every 5-10 minutes :calendar: For that either utilize the task scheduler which is present on every Windows machine or use the CI/CD environment of your choice, given the runners / workers use Windows. In any case make sure the script is run with a user account that has sufficient permissions to modify group members in your AD, ideally a system user.

## Scaling
In our example we have configured two dynamic AD groups. The script theoretically allows an infinite number of those groups. However, keep an eye on the execution time of the script which should not be larger than the scheduling interval to avoid concurrent runs. Also check the CPU / RAM load on the execution server and your domain controllers. I have run it without issues in environments of ~1000 users and 20 dynamic groups with a scheduling interval of 5 minutes.

## Filter Query
You may have noticed that the queries between AD and Azure AD differ, even though they achieve exactly the same thing.
```powershell
# Azure AD
user.jobTitle -contains "Designer"

# AD
title -like '*Designer*'
```
That's because AD and Azure AD are two separate systems with their own attributes and comparison logic. For example, while Azure AD does not offer a `like` operator, AD does offer a `contain` operator which however has a completely different meaning. While AD accepts wildcard characters such as `*`, Azure AD does not. And there are many more differences. So make sure to check the [documentation](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser?view=windowsserver2022-ps#parameters) on Get-ADUser filter syntax before jumping in.

## Parameters
The script offers some additional features you can utilize with parameters. Let's take a look.

### GroupSearchBase
You can speed up execution by providing an OU for the `GroupSearchBase` parameter. The script will then only consider groups within this OU (recursively).
```powershell
.\Sync-DynamicAdGroupMember.ps1 -GroupSearchBase "OU=groups,DC=contoso,DC=com"
```

### UserSearchBase
You can speed up execution but also limit your Get-ADUser query results by providing an OU for the `UserSearchBase` parameter. The script will then only consider users within this OU (recursively). This is particularly useful if you move users to an archive OU during offboarding (which is outside the OU you provide in the parameter) and hence automatically remove them from your dynamic groups.
```powershell
.\Sync-DynamicAdGroupMember.ps1 -UserSearchBase "OU=users,DC=contoso,DC=com"
```

### WhatIf
You are hesitant to run the script in your production environment? :weary: Try it out first with the `WhatIf` switch. The script will not perform any changes but provide output about them.
```powershell
.\Sync-DynamicAdGroupMember.ps1 -WhatIf

What if: role-title-designer: (+) john.doe
What if: role-title-designer: (+) sam.smith
```

`WhatIf` can also be combined with `VERBOSE` to receive additional output.
```powershell
.\Sync-DynamicAdGroupMember.ps1 -WhatIf -VERBOSE

VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching AD groups with a value in extensionAttribute10
VERBOSE: Syncing group members
VERBOSE: role-title-designer: title -like '*Designer*'
What if: role-title-designer: (+) john.doe
What if: role-title-designer: (+) sam.smith
```

### PassThru
By adding the `PassThru` switch the script will return pipeable output for all changes that were made which you could potentially write to a log file. If no changes were made, no output is generated.
```powershell
.\Sync-DynamicAdGroupMember.ps1 -PassThru | Out-File -FilePath .\Log.txt

Group                 Query                      User          Action
-----                 -----                      ----          ------
role-title-designer   title -like '*Designer*'   john.doe      Add
role-title-designer   title -like '*Designer*'   sam.smith     Add
```

## Conclusion
We dit it! Dynamic groups in Active Directory. This enables a few things:
* We can assign resources to these groups in AD by assigning it to other groups
* We can use these groups within applications that use AD as user base (e.g. we could assign a group to a role in a Jira project)
* We can even assign resources in Azure AD, given that the groups are in the Azure AD Connect sync scope

Fantastic! :blush:
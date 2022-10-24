---
tags: active-directory azure-ad dynamic groups powershell
---

One of my favorite features in Azure AD is dynamic groups. You can simply manage users of a group by defining filter rules. Since many organizations still use Active Directory to manage their users and resources wouldn't it be great to have the same functionality there? Say no more :sunglasses:

## Azure AD
For quite some time now Azure AD offers [dynamic groups](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/groups-dynamic-membership). This means membership of a user in a group is determined by filter rules based on the user's attributes. Let's look at an example:  

![Filter rule of an Azure AD group](/images/aad.png)

You may have noticed the filter rule.
```
(user.jobTitle -contains "Developer")
```

This means all users in our AAD which have the word `Developer` in their title, are member of this group. Let's check out our group members.
```powershell
Get-AzureADGroup -Filter "DisplayName eq 'role-title-developer'" | Get-AzureADGroupMember | Select-Object UserPrincipalName,JobTitle
```
```
UserPrincipalName           JobTitle
-----------------           --------
john.doe@contoso.com        Senior Frontend Developer
sam.smith@contoso.com       Expert Software Developer
tom.tokins@contoso.com      Software Developer
```


Look's about right. We can now go ahead and assign Azure AD resources to those groups, whether that's an app, a license or memberships or ownership for other groups. This comes in very handy if we synchronize our employee data from an HR system. The only thing we need to worry about is the sync. Group membership and access to resources is completely automated. Nice! :blush:  

But what if some or even all of your resources are managed in Active Directory? Keep on reading.

## Active Directory

I have built the script [Sync-DynamicAdGroupMember.ps1](https://github.com/dominikduennebacke/Sync-DynamicAdGroupMember) which basically mimics the functionality of Azure AD dynamic groups in Active Directory.  

> **.SYNOPSIS**  
> Manages AD group members based on Get-ADUser filter query defined in an extensionAttribute.

Ok let's try it out. First we create a new AD group.
```powershell
$Params = @{
    Name          = "role-title-developer"
    Description   = "Dynamically managed containing all developers of the org"
    GroupCategory = "Security"
    GroupScope    = "Universal"
    Path          = "OU=groups,DC=contoso,DC=com"
}
New-ADGroup @Params
```
Then we set a Get-ADUser filter query on `extensionAttribute10`.
```powershell
Set-ADGroup -Identity "role-title-developer" -Replace @{
    extensionAttribute10 = "title -like '*Developer*'"
}
```

Now we download the script.
```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/dominikduennebacke/Sync-DynamicAdGroupMember/main/Sync-DynamicAdGroupMember.ps1" -OutFile "Sync-DynamicAdGroupMember.ps1"
```
And run it providing number `10` for parameter `ExtensionAttribute`. This tells the script that the Get-ADUser filter can be found on this attribute. When you run the script make sure you comply with the [requirements](https://github.com/dominikduennebacke/Sync-DynamicAdGroupMember#REQUIREMENTS).
> :warning: **Warning**  
> As with any script from the internet, use it at your own risk and inspect the source code before running it.

```powershell
./Sync-DynamicAdGroupMember.ps1 -ExtensionAttribute 10 -VERBOSE
```
```
VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching AD groups with a value in extensionAttribute10
VERBOSE: Syncing group members
VERBOSE: role-title-developer: department -eq 'Sales'
VERBOSE: role-title-developer: (+) john.doe
VERBOSE: role-title-developer: (+) sam.smith
VERBOSE: role-title-developer: (+) tom.tonkins
```

Let's verify the group members.
```powershell
Get-ADGroupMember -Identity "role-title-developer" | Get-ADUser -Properties title | Select-Object UserPrincipalName, Title
```
```
UserPrincipalName           Title
-----------------           -----
john.doe@contoso.com        Senior Frontend Developer
sam.smith@contoso.com       Expert Software Developer
tom.tokins@contoso.com      Software Developer
```

Et voilà, all users in our AD which have the string `Developer` in their title attribute are now member of the group `role-title-developer` :muscle: So we have successfully replicated the dynamic group feature from AAD to AD. This enables a few things for us:
* We can assign resources to this group in AD by assigning it to other groups
* We can use the group within applications that use AD as user base (e.g. you could assign the group to a role in a Jira project)
* We can even assign resources in AAD, given that the group is in the Azure AD Connect sync scope

Sounds like the jack of all trades. Fantastic! :blush:

## Scheduling
In order to fully replicate the AAD feature we need to set up scheduling. I recommend running the script every 5-10 minutes. For that either utilize the task scheduler which is present on each Windows machine or use the CI/CD environment of your choice, given the runners / workers use Windows. In any case make sure the script is run with a user account that has sufficient permissions to modify group members in your AD, ideally a system user.

## Scaling
One dynamic AD group is great, but how about ten? No problem, the script theoretically allows an infinite number of dynamic groups. However, keep an eye on the execution time of the script which should not be larger than the scheduling interval to avoid concurrent runs. Also check the CPU / RAM load on the execution server and your domain controllers. I have run it without issues in environments of ~1000 users and 30 dynamic groups with a scheduling interval of 5 minutes.

## Filter query
You may have noticed that the queries between the AD and AAD differ even though they achieve the same thing.
```
# AD
"title -like '*Developer*'"

# AAD
(user.jobTitle -contains "Developer")
```
That's because AD and AAD are two seperate systems with their own attributes and comparison logic. For example, while AAD does not offer a `like` operator, AD does offer a `contain` operator which however has a completely different meaning. While AD accepts wildcards `*`, AAD does not. And there are many more differences. So make sure to check out the [documentation](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser?view=windowsserver2022-ps#parameters) on Get-ADUser filter syntax before jumping in. To get your creativity going here are some examples.

```powershell
# Group: role-department-marketing
department -eq 'Marketing'

# Group: role-department-marketingandsales
(department -eq 'Marketing') or (department -eq 'Sales')

# Group: role-type-employee
(employeeType -eq 'Employee') -and (Enabled -eq $true)

# Group: role-office-nyc
office -eq "New York
```

## Parameters
Oh, so you're a pro user? Cool, :sunglasses: here are some extra features the script offers.

### SearchBase
You can speed up execution by providing an OU for the `SearchBase` parameter. The script will then only consider groups within this OU (recursively).
```powershell
.\Sync-NestedAdGroupMember.ps1 -SearchBase "OU=groups,DC=contoso,DC=com"
```

### LegacyPair
Karl is the application owner of GrayLog and he doesn't like change. Approaching him to set up `app-graylog-access-UNNESTED` instead of `app-graylog-access` will cause lenghty discussions and it will just take forever. Say no more - been there, done that. In that case you can provide additional pairs as hashtable using the parameter `LegacyPair`. Duh Karl! :stuck_out_tongue_winking_eye:
```powershell
.\Sync-NestedAdGroupMember.ps1 -LegacyPair @{"app-graylog-access-NESTED" = "app-graylog-access"; "app-kibana-access-NESTED" = "app-kibana-access"} -VERBOSE
```
```
VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching NESTED AD groups
VERBOSE: Syncing group members recursively from NESTED group(s) to UNNESTED group(s)
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) john.doe
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) sam.smith
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) tom.tonkins
VERBOSE: app-graylog-access-NESTED > app-graylog-access
VERBOSE: app-graylog-access-NESTED > app-graylog-access: (+) john.doe
VERBOSE: app-graylog-access-NESTED > app-graylog-access: (+) sam.smith
VERBOSE: app-graylog-access-NESTED > app-graylog-access: (+) tom.tonkins
VERBOSE: app-kibana-access-NESTED > app-kibana-access
VERBOSE: app-kibana-access-NESTED > app-kibana-access: (+) john.doe
VERBOSE: app-kibana-access-NESTED > app-kibana-access: (+) sam.smith
VERBOSE: app-kibana-access-NESTED > app-kibana-access: (+) tom.tonkins
```

### WhatIf
You are hesitant to run the script in your production environment? :weary: Try it out first with the `WhatIf` switch. The script will not perform any changes but provide output about them.
```powershell
.\Sync-NestedAdGroupMember.ps1 -WhatIf
```
```
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) john.doe
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) sam.smith
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) tom.tonkins
```

`WhatIf` can also be combined with `VERBOSE` to receive additional output.
```powershell
.\Sync-NestedAdGroupMember.ps1 -WhatIf -VERBOSE
```
```
VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching NESTED AD groups
VERBOSE: Syncing group members recursively from NESTED group(s) to UNNESTED group(s)
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) john.doe
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) sam.smith
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) tom.tonkins
```

### PassThru
You have set up a scheduled task to run the script and demand output that you want to pipe to a log file? By adding the `PassThru` switch the script will return pipeable output for all changes that were made. If no changes were made, no output is generated.
```powershell
.\Sync-NestedAdGroupMember.ps1 -PassThru | Out-File -FilePath .\Log.txt
```
```
NestedGroup               UnnestedGroup               User        Action
-----------               -------------               ----        ------
app-graylog-access-NESTED app-graylog-access-UNNESTED john.doe    Add
app-graylog-access-NESTED app-graylog-access-UNNESTED sam.smith   Add
app-graylog-access-NESTED app-graylog-access-UNNESTED tom.tonkins Add
```

### NestedSuffix / UnnestedSuffix
You tried, you really tried, but you cannot deal with the pre-defined suffixes `-NESTED` and `-UNNESTED` that determine group pairs? Alright alright, calm down. The script has two parameters hidden from IntelliSense which allow you to override the suffixes. Make sure they are unique, so groups are not accidentally considered as pair by the script.
```powershell
.\Sync-NestedAdGroupMember.ps1 -NestedSuffix "-nest" -UnnestedSuffix "-unnest"
```

## Conclusion
So there you have it, a simple PowerShell script that can save you lots of time and allows you to utilize RBAC or other access management methods based on user groups.

## Disclaimer
GrayLog supports nested group membership since 2020 and more and more applications do so too. Additionally many of them offer modern authentication procotols such as OAUTH or SAML that you can utilize with your identity provider (e.g. Azure AD). So keep an eye - you might not even need this script.
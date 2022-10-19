---
tags: powershell active-directory
---

Don't you hate it? You have spent countless hours on setting up a role-based access control model. You created role groups for your Active Directory users which you assigned to resource groups, but now this one application put's up a fight - it does not support nested group membership. No problem, you assign users directly to the resource group. However, next time Sam changes the department you need to manually remove him from that group. Ugh! Can't we automate that? Sure, PowerShell to the rescue.

## Problem
Let's look at a typical access management approach. We have configured the resource group `app-graylog-access` in GrayLog to provide base access to the application. Nested within that resource group we added two role groups `role-department-devops` and `role-department-sysops` that contain our users. Now all those users can access GrayLog and the only thing we need to worry about is managing our role groups.
```
├── app-graylog-access
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
```
This works great, provided that GrayLog supports nested group membership. But what if it doesn't?

## Solution
Well, we use our handy script [Sync-NestedAdGroupMember](https://github.com/dominikduennebacke/Sync-NestedAdGroupMember). First we create a group pair in Active Directory.
* `app-graylog-access-NESTED`: Here we manage the users.
* `app-graylog-access-UNNESTED`: This group is configured within GrayLog to allow base access.

Then we nest our role groups into `app-graylog-access-NESTED`.
```
├── app-graylog-access-NESTED
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
├── app-graylog-access-UNNESTED
```

Now we download the script and run it. Make sure you comply with the [requirements](https://github.com/dominikduennebacke/Sync-NestedAdGroupMember#REQUIREMENTS) when doing so.
```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/dominikduennebacke/Sync-NestedAdGroupMember/main/Sync-NestedAdGroupMember.ps1" -OutFile "Sync-NestedAdGroupMember.ps1"
./Sync-NestedAdGroupMember.ps1 -VERBOSE
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
```

Let's look at our group structure one more time:
```
├── app-graylog-access-NESTED
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
├── app-graylog-access-UNNESTED
│   ├── john.doe
│   ├── sam.smith
│   ├── tom.tonkins
```

Et voilà, all users that are a (nested) member of the group `app-graylog-access-NESTED` are now direct member of the group `app-graylog-access-UNNESTED`. This works two ways: Missing members are added, and obsolete members are removed. So if Sam makes that department change, he will automatically be removed. Yay!

## Scheduling
So how does this help me? I would need to run this script every time something in my role groups changes. Not quite! Scheduling allows you to sit back and let automation do its job. Let the script run every 5-10 minutes. This ensures that changes made to the `NESTED` group are reflected in the `UNNESTED` group in a timely manner. I won't go into depth how to set up scheduling for now, but in short: Either utilize the task scheduler which is present on each Windows machine or use the CI/CD environment of your choice, given the runners/workers use Windows. In any case make sure the script is run with a user account that has sufficient permissions to modify group members in your AD.

## Scaling
One pair is great, but how about 10? No problem, the script theoretically allows an infinite number of pairs. However, keep an eye on the execution time of the script which should not be larger than the scheduling interval you choose. I have run it in environments of roughly 1000 users and 6-7 pairs with a scheduling interval of 5 minutes, without issues.

## The suffixes
The script dictates to use the suffixes `-NESTED` and `-UNNESTED` for your group pairs. Does this make you feel uncomfortable? All caps is not your cup of tea? I hear you, but please bear with me. In my experience it is very valuable to quickly identify groups which are managed by the script, whether that is in an application, in an OU or the AD user's "Member Of" tab. Maybe you even run monitoring on your AD users, to prevent direct group memberships. With the naming convention you can easily exclude `-UNNESTED` groups from that check.

## Parameters
Oh, so you're a pro user? Cool, here are some extra features the script offers.

### SearchBase
You can speed up execution by providing an OU for the `SearchBase` parameter. The script will then only consider groups within this OU (recursively).
```powershell
.\Sync-NestedAdGroupMember.ps1 -SearchBase "OU=groups,DC=contoso,DC=com"
```

### LegacyPair
Karl is the application owner of GrayLog and he doesn't like change. Approaching him to set up `app-graylog-access-UNNESTED` instead of `app-graylog-access` will cause some nasty discussions and it will take forever. Say no more. In that case you can provide additional pairs as hashtable using the parameter `LegacyPair`. Duh Karl!
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
VERBOSE: app-kibana-access-NESTED > app-kibana-access:  (+) john.doe
```

### WhatIf
You are a bit hesitant to run the script in your production environment? Try out the parameter `WhatIf`. The script will not perform any changes but provide output about them.
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
You have set up a scheduled task to run the script and demand output that you want to pipe to a log file? No problem. By adding the `PassThru` switch the script will return an output for all changes that were made.
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

## Conclusion
So there you have it, a simple PowerShell script that can save you lots of time and allows you to utilize RBAC or other access management methods based on user groups.

## Disclaimer
By now GrayLog supports nested group membership and more and more applications do so too. So make sure to keep your applications up to date - you might not even need this script.
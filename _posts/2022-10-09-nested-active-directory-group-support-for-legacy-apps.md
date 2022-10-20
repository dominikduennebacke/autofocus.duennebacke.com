---
tags: powershell active-directory groups
---

Don't you hate it? You have spent countless hours on setting up a role-based access model in your Active Directory. You created role groups for your users which you assigned to resource groups, but now this one application put's up a fight - it does not support nested group membership. No problem you say. Just assign users directly to the resource group. However, next time Sam changes the department you need to manually remove him from that group. Ugh! Can't we automate that? Sure, PowerShell to the rescue!

## Problem
Let's look at a typical access management approach. We have configured the resource group `app-graylog-access` in GrayLog to provide base access to the application. To that resource group we added two role groups (`role-department-devops` and `role-department-sysops`) as members which contain our users as direct members. Now all those users can access GrayLog and the only thing we need to worry about is managing our role groups.
```
├── app-graylog-access
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
```
This works great, provided that GrayLog supports nested group membership. But what if it doesn't?

## Workaround
We use our handy script [Sync-NestedAdGroupMember.ps1](https://github.com/dominikduennebacke/Sync-NestedAdGroupMember)! Let's take a look what it does.
```
.SYNOPSIS
Fetches members of AD groups with name suffix -NESTED recursively and syncs them to their -UNNESTED counterpart.
```

Great! Sounds like that's exactly what we need. Let's try it out. So, first we create a pair of AD groups. Make sure the names before the suffixes are identical.
* `app-graylog-access-NESTED`: This is where we manage our users.
* `app-graylog-access-UNNESTED`: This group is configured within GrayLog to allow base access.

Then we add our role groups as members to `app-graylog-access-NESTED`. The structure looks as follows.
```
├── app-graylog-access-NESTED
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
├── app-graylog-access-UNNESTED
```

Now we download the script.
```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/dominikduennebacke/Sync-NestedAdGroupMember/main/Sync-NestedAdGroupMember.ps1" -OutFile "Sync-NestedAdGroupMember.ps1"
```
And run it. Make sure you comply with the [requirements](https://github.com/dominikduennebacke/Sync-NestedAdGroupMember#REQUIREMENTS) when doing so.
```powershell
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

Et voilà, all users that are a (nested) member of the group `app-graylog-access-NESTED` are now direct member of the group `app-graylog-access-UNNESTED` and can access GrayLog. Now let's look at Sam. The next day he changes to the sales department (who does that?!). On the next run the script automatically removes him from the `-UNNESTED` group. Yay!
```
├── app-graylog-access-NESTED
│   ├── role-department-devops
│   │   ├── john.doe
│   ├── role-department-sysops
│   │   ├── tom.tonkins
├── app-graylog-access-UNNESTED
│   ├── john.doe
│   ├── tom.tonkins
```

## Scheduling
So how can you call that automation? I would need to run the script every time something in my role groups changes. Well, not quite! Scheduling allows you to sit back and relax. Let the script run every 5-10 minutes. This ensures that changes made in `NESTED` group are reflected in the `UNNESTED` group in a timely manner. I won't go into depth how to set up scheduling for now, but in short: Either utilize the task scheduler which is present on each Windows machine or use the CI/CD environment of your choice, given the runners / workers use Windows. In any case make sure the script is run with a user account that has sufficient permissions to modify group members in your AD, ideally a system user.

## Scaling
One group pair is great, but how about ten? No problem, the script theoretically allows an infinite number of pairs. However, keep an eye on the execution time of the script which should not be larger than the scheduling interval to avoid concurrent runs. I have run it in environments of ~1000 users and 6-7 pairs with a scheduling interval of 5 minutes, without issues.

## Suffixes
The script dictates to use the suffixes `-NESTED` and `-UNNESTED` for your group pairs. Does this make you feel uncomfortable? All caps is not your cup of tea? I hear you, but please bear with me. In my experience it is very valuable to quickly identify groups which are managed by the script. This could be in:
* An application
* AD Users and Computers (ADUC) Snap-in
* AD role group's 'Member Of' tab
* AD user's 'Member Of' tab

Maybe you even run monitoring on your AD users, to report (and remove) direct group memberships. With the naming convention you can easily exclude `-UNNESTED` groups from that check. Also the suffixes need to be unique so the script does not accidentally consider other groups as pairs. You are still not convinced and insist on other suffixes? I got you, take a look at the parameter section.

## Parameters
Oh, so you're a pro user? Cool, here are some extra features the script offers.

### SearchBase
You can speed up execution by providing an OU for the `SearchBase` parameter. The script will then only consider groups within this OU (recursively).
```powershell
.\Sync-NestedAdGroupMember.ps1 -SearchBase "OU=groups,DC=contoso,DC=com"
```

### LegacyPair
Karl is the application owner of GrayLog and he doesn't like change. Approaching him to set up `app-graylog-access-UNNESTED` instead of `app-graylog-access` will cause lenghty discussions and it will just take forever. Say no more - been there, done that. In that case you can provide additional pairs as hashtable using the parameter `LegacyPair`. Duh Karl!
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
You are hesitant to run the script in your production environment? Try it out first with the `WhatIf` switch. The script will not perform any changes but provide output about them.
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
You have set up a scheduled task to run the script and demand output that you want to pipe to a log file? By adding the `PassThru` switch the script will return an output for all changes that were made.
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
You tried, you really tried, but you cannot deal with the pre-defined suffixes `-NESTED` and `-UNNESTED` that determine group pairs? Alright alright, calm down. The script has two parameters hidden from IntelliSense which allow you to override the suffixes.
```powershell
.\Sync-NestedAdGroupMember.ps1 -NestedSuffix "-nest" -UnnestedSuffix "-unnest"
```

## Conclusion
So there you have it, a simple PowerShell script that can save you lots of time and allows you to utilize RBAC or other access management methods based on user groups.

## Disclaimer
GrayLog supports nested group membership since 2020 and more and more applications do so too. Additionally many of them offer modern authentication procotols such as OAUTH or SAML that you can utilize with your identity provider (e.g. Azure AD). So keep an eye - you might not even need this script.

> **Note:**
> As with any script you get from the internet, use them at your own risk.
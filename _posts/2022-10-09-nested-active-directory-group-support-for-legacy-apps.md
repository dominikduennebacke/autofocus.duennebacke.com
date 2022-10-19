---
tags: active-directory
---

Don't you hate it? You have spent countless hours on creating a role-based access control model. You created role groups for your Active Directory users which you assign to resource groups, but now this one application put's up a fight - it does not support nested group membership. No problem, you assign users directly to the access group. However next time Sam changes the department you need to manually remove him from that group. Ugh! Can't we automate that? Sure, PowerShell to the rescue. I created the script [Sync-NestedAdGroupMember](https://github.com/dominikduennebacke/Sync-NestedAdGroupMember) that serves as a workaround.

## Problem
Let's look at a typical access management approach. We have the resource group `app-graylog-access` which is configured in GrayLog to provide base access to the application. Nested within that resource group we find two role groups `role-department-devops` and `role-department-sysops` that contain our users. This works great, given that GrayLog supports nested group membership. But what if it doesn't?
```
├── app-graylog-access
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
```

## Solution
Well, we just use our handy script. First we create a group pair:
* `app-graylog-access-NESTED`: Here we manage the users which can be nested in groups
* `app-graylog-access-UNNESTED`: This group is configured within GrayLog to allow base access

The structure looks like this.
```
├── app-graylog-access-NESTED
│   ├── role-department-devops
│   │   ├── john.doe
│   │   ├── sam.smith
│   ├── role-department-sysops
│   │   ├── tom.tonkins
├── app-graylog-access-UNNESTED
```

Now we run the script.
```powershell
Sync-NestedAdGroupMember -VERBOSE
```
```
VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching NESTED AD groups
VERBOSE: Syncing group members recursively from NESTED group to UNNESTED group
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
So how does this help me? I would need to run this script every time something in my role groups changes. Not quite! Scheduling allows you to sit back and let automation do its job. Let the script run every 5-10 minutes. This ensures that changes made to the `NESTED` group are reflected in the `UNNESTED` group in a timely manner. I won't go into depth how to set up scheduling, but in short: Either utilize the task scheduler which is present on each Windows machine or use the CI/CD environment of your choice, given the runners/workers use Windows. In any case make sure the script is run with a user account that has sufficient permissions to modify group members in your AD.

## Scaling
One pair is great, but how about 10? No problem, the script theoretically allows an infinite amount of pairs. However, keep an eye on the execution time of the script which should not be larger than the scheduling interval you choose. I have run it in environments of roughly 1000 users and 6-7 pairs with a scheduling interval of 5 minutes without issues.

## The suffixes
The script dictates to use the suffixes `-NESTED` and `UNNESTED`. I don't like it, it's ugly and capitalized, wtf man? I hear you, but in my experience it is valuable to be able to quickly identify group pairs, whether that is in an application or the AD users memberOf tab. Maybe you even run a monitoring on your users, to prevent direct group membership. By the naming convention you could exclude `UNNESTED` groups from that check.

## Parameters
Oh, so you're a pro user? Cool, here are some extra features the script offers.

### SearchBase
You can speed up execution by providing an OU for the `SearchBase` parameter. The script will then only consider groups within this OU (recursively).
```powershell
Resolve-NestedAdGroup -SearchBase "OU=Groups,DC=contoso,DC=com"
```

### LegacyPair
Karl is the application owner of GrayLog and he doesn't like change. Approaching him to set up `app-graylog-access-UNNESTED` instead of `app-graylog-access` will cause some nasty discussions and it will take forever. Say no more. In that case you can provide additional pairs as hashtable using the parameter `LegacyPair`. Duh Karl!
```powershell
Sync-NestedAdGroupMember -LegacyPair @{"app-graylog-access-NESTED" = "app-graylog-access"} -VERBOSE
```
```
VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching NESTED AD groups
VERBOSE: Syncing group members recursively from NESTED group to UNNESTED group
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) john.doe
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) sam.smith
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) tom.tonkins
VERBOSE: app-graylog-access-NESTED > app-graylog-access: (+) john.doe
VERBOSE: app-graylog-access-NESTED > app-graylog-access: (+) sam.smith
VERBOSE: app-graylog-access-NESTED > app-graylog-access: (+) tom.tonkins
```

### WhatIf
You are a bit hesitant to run the script in your production environment? No problem, adding the parameter `WhatIf` will not perform any changes but provide output about them.
```powershell
Sync-NestedAdGroupMember -WhatIf
```
```
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) john.doe
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) sam.smith
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) tom.tonkins
```

`WhatIf` can also be combined with `-VERBOSE` to receive additional output.
```powershell
Sync-NestedAdGroupMember -WhatIf -VERBOSE
```
```
VERBOSE: Checking dependencies
VERBOSE: The secure channel between the local computer and the domain is in good condition.
VERBOSE: Fetching NESTED AD groups
VERBOSE: Syncing group members recursively from NESTED group to UNNESTED group
VERBOSE: app-graylog-access-NESTED > app-graylog-access-UNNESTED
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) john.doe
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) sam.smith
What if: app-graylog-access-NESTED > app-graylog-access-UNNESTED: (+) tom.tonkins
```

## Conclusion
So there you have it, a simple PowerShell script that can save you lots of hassle and allows you to utilize access management thru role groups or similar setups.

## Disclaimer
By now GrayLog allows nested group membership and more and more applications do too. So make sure to keep your applications up to date, so you might not even need this script.
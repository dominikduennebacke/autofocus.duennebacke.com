---
tags: active-directory
published: false
---

One of my favorite features in Azure AD is the dynamic group features. You can simply add users to a group by defining a query based on a users attribute. Wouln't it be great to have the same feature in Active Directory? I got you covered.

## Download
I have published the script on the PowerShell Gallery. You can simply install it on any Windows server with internet connection:
```powershell
Install-Script Resolve-NestedAdGroup
```

## Usage
Let's assume you have an old Graylog instance running (that does not yet support nested groups) and you want to assign access to the AD group `role-department-devops` which contains all users of your DevOps team: `john.doe`, `sam.smith` and `tom.tonkins`.
* Create two AD groups (as a pair for the script)
  * `app-graylog-access-NESTED`
  * `app-graylog-access-UNNESTED`
* Add the group `role-department-devops` to `app-graylog-access-NESTED`
* Configure `app-graylog-access-UNNESTED` within Graylog to allow access to the application
* Execute the script with a user account that has permissions to manage users of those groups (typically this would be a system user with domain admin rights or targeted permissions)

The script will add `john.doe`, `sam.smith` and `tom.tonkins` as direct members to the group `app-graylog-access-UNNESTED`.

Now let's assume `john.doe` switched to the sales team and is no longer member of `role-department-devops`. The script will then remove the user `john.doe` from `app-graylog-access-UNNESTED` on the next run.

You can create an infinite number of group pairs. Pairs are detected automatically by an identical group name before the suffixes `-NESTED` and `-UNNESTED`.

Depending on your environment, you can now set up a scheduled task on a Windows server within your AD domain, or configure a CI/CD pipeline that runs the script every 5-10 minutes. Make sure the executing user has sufficient permissions.

## Parameters

### SearchBase
You can speed up execution of the script by providing an OU for the `SearchBase` parameter. The script will then only consider groups within this OU (recursively).
```powershell
Resolve-NestedAdGroup -SearchBase "OU=Groups,DC=contoso,DC=com"
```

### LegacyPair
Sometimes it is a bit tricky to switch existing groups on applications. Hence the script has a parameter `LegacyPair` to which you can provide sync pairs as a hashtable. The script will then treat them as a NESTED/UNNESTED pair.
```powershell
Resolve-NestedAdGroup -LegacyPair @{
    "app-graylog-access-NESTED" = "graylog-access"
    "rdp-exchange01-NESTED"     = "rdp-exchange01"
}
```
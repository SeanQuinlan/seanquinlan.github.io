---
layout: post
title: "Getting the NTSecurityDescriptor AD attribute via ADSI"
date: 2018-11-23 19:13
tags: adsi
description: A post showing how to obtain the NTSecurityDescriptor attribute for an Active Directory user, via ADSI.
comments: true
---

Recently I have been creating my own module to try and replicate the functionality of Microsoft's "ActiveDirectory" PowerShell module, but using only the .NET System.DirectoryServices namespace (often referred to as ADSI). I realise that there are already a couple of modules that try and provide the same functionality ([AdsiPS](https://github.com/lazywinadmin/AdsiPS/) and [PSAD](https://github.com/zloeber/PSAD)), but this was partly a learning exercise for myself and neither of the modules provided everything I required.

So I set about creating my own module, and one of the first problems I encountered was trying to replicate all the user attributes that are returned via Get-ADUser. Many of these attributes are not regular LDAP attributes retrieved from Active Directory, but rather are specially constructed (or calculated) based off a different attribute.

Originally I set out to find how to get the **CannotChangePassword** attribute, but I discovered that this is calculated from the **NTSecurityDescriptor** attribute, which itself requires a little work to format correctly. So let's start with **NTSecurityDescriptor** and we can look at **CannotChangePassword** in the next post.

The module I am writing will focus on searching AD via ADSI, so I wrote a small script to search AD for a specific username and return just the **DistinguishedName** and **NTSecurityDescriptor** attributes.

```powershell
$User = 'TestUser'
$Properties = @('distinguishedname','ntsecuritydescriptor')

$Searcher = [adsisearcher]"(samaccountname=$User)"
$Searcher.PropertiesToLoad.AddRange($Properties)
$Results = $Searcher.FindAll()
$Results | Foreach-Object {
    $UserObject = [pscustomobject]@{
        'DistinguishedName' = $($_.Properties.distinguishedname)
        'NTSecurityDescriptor' = $($_.Properties.ntsecuritydescriptor)
    }
    $UserObject
}
```

Running it returns an object with the 2 properties I was looking for:

```plaintext
PS C:\Scripts> .\Get-TestUser.ps1

DistinguishedName                    NTSecurityDescriptor
-----------------                    --------------------
CN=Test User,CN=Users,DC=test,DC=lab {1, 0, 20, 140...}

```

However when I compared the value of **NTSecurityDescriptor** returned via ADSI and the same value returned via Get-ADUser, I can see that they are different. ADSI returns it as `System.Byte[]` and Get-ADUser returns it as `System.DirectoryServices.ActiveDirectorySecurity`. So I went back to searching how to convert the ADSI value to the Get-ADUser one.

I didn't actually find my answer, but during the search I stumbled upon [this post from Richard Siddaway](https://blogs.msmvps.com/richardsiddaway/2012/03/12/display-ad-object-s-security-settings-by-identity/) and in it he details a number of ways of getting security settings from an AD object. One of the methods is via ADSI, and I can see that if he binds directly to an object, he can access the ObjectSecurity property, which looks like the thing I'm looking for.

Since I'm performing a search (and potentially getting more than one result), I actually get a `System.DirectoryServices.SearchResult` object returned instead. But the directory object itself is included in that SearchResult object, and can be retrieved via the `GetDirectoryEntry()` method on the search result.

So I modified the script to use the `GetDirectoryEntry()` method, and then return the ObjectSecurity property from that, rather than returning the LDAP byte array that it returned before.

```powershell
$User = 'TestUser'
$Properties = @('distinguishedname','ntsecuritydescriptor')

$Searcher = [adsisearcher]"(samaccountname=$User)"
$Searcher.PropertiesToLoad.AddRange($Properties)
$Results = $Searcher.FindAll()
$Results | Foreach-Object {
    $UserObject = [pscustomobject]@{
        'DistinguishedName' = $($_.Properties.distinguishedname)
        'NTSecurityDescriptor' = $($_.GetDirectoryEntry().ObjectSecurity)
    }
    $UserObject
}
```

Running this updated script, it appears to return something similar to Get-ADUser.

```plaintext
PS C:\Scripts> .\Get-TestUser.ps1

DistinguishedName                    NTSecurityDescriptor
-----------------                    --------------------
CN=Test User,CN=Users,DC=test,DC=lab System.DirectoryServices.ActiveDirectorySecurity


PS C:\Scripts> Get-ADUser TestUser -Properties NTSecurityDescriptor | Select-Object DistinguishedName,NTSecurityDescriptor

DistinguishedName                    NTSecurityDescriptor
-----------------                    --------------------
CN=Test User,CN=Users,DC=test,DC=lab System.DirectoryServices.ActiveDirectorySecurity
```

Looking good. I accessed the first 3 entries under the "Access" sub-property using both methods, to confirm I was accessing the same data.

```plaintext
PS C:\Scripts> (.\Get-TestUser.ps1).NTSecurityDescriptor.Access | Select -First 3


ActiveDirectoryRights : ExtendedRight
InheritanceType       : None
ObjectType            : ab721a53-1e2f-11d0-9819-00aa0040529b
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Deny
IdentityReference     : Everyone
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : ExtendedRight
InheritanceType       : None
ObjectType            : ab721a53-1e2f-11d0-9819-00aa0040529b
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Deny
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : GenericRead
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None



PS C:\Scripts> (Get-ADUser TestUser -Properties NTSecurityDescriptor | Select-Object DistinguishedName,NTSecurityDescriptor).NTSecurityDescriptor.Access | Select -First 3


ActiveDirectoryRights : ExtendedRight
InheritanceType       : None
ObjectType            : ab721a53-1e2f-11d0-9819-00aa0040529b
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Deny
IdentityReference     : Everyone
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : ExtendedRight
InheritanceType       : None
ObjectType            : ab721a53-1e2f-11d0-9819-00aa0040529b
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Deny
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : GenericRead
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None
```

This will be very helpful for my next task, which is calculating the **CannotChangePassword** attribute using **NTSecurityDescriptor**.


---
layout: post
title: Read Passwords in PowerShell
---

Working with passwords doesn't always have to be scary.

PowerShell has made it incredibly easy and secure to specify and use credentials with the Get-Credential command.
```powershell
$credential=Get-Credential
```
![_config.yml]({{ site.baseurl }}/images/getCredential.PNG)

<br>
The `Get-Credential` cmdlet returns a PSCredential object.

You may not know this but there is a method for this object type that will allow you to the stored password as plain text.

```powershell
$credential=Get-Credential
$credential.GetNetworkCredential()

$credential.GetNetworkCredential() | FL
```

![_config.yml]({{ site.baseurl }}/images/getNetworkCredentials.PNG)




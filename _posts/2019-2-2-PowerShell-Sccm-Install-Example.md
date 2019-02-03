---
layout: post
title: PowerShell - Sccm Install Example
---

After creating 5 posts about automating Sccm primary site installs, 
I think an example bringing everything together is useful. 

Sccm Automation Posts:
- [Post 1 ](http://codeandkeep.com/PowerShell-ActiveDirectory-Exchange-Part1/) 
  - Download Prereqs
- [Post 2 ](http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites-Install/) 
  - Install Prereqs
- [Post 3 ](http://codeandkeep.com/PowerShell-Sccm-PreRequisites/) 
  - AD Prereqs
- [Post 4 ](http://codeandkeep.com/PowerShell-Sccm-AD-PreRequisites-SQL/) 
  - Sql Install
- [Post 5 ](http://codeandkeep.com/PowerShell-Sccm-Primary-Site-Install/) 
  - Sccm Primary Site Install

### The Environment
----

<p>
  This is a lab environment with the following servers:
    <ul>
      <li>dc1</li>
        <ul>
          <li>DomainController</li>
        </ul>
      <li>cm1</li>
        <ul>
          <li>Will be the Sccm Primary Site Server</li>
        </ul>
      <li>sql</li>
        <ul>
          <li>Will be the Sccm Sql server</li>
        </ul>
    </ul>
    All servers are Windows Server 2016. 
</p>
<p>
  NOTE: You can install Sql on the cm1 server to save building another VM.
</p>

### Download
----

<p>
  In my environment, none of my servers have internet access, 
  so all my downloads will done on my host machine and copied to the VMs.
</p>

First, I have downloaded the latest ADK and Windows PE Add-on for the ADK.
Both can be found [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install).

<p>
  I have mounted the Sccm Installation ISO image to my host machine. 
  It is mounted on Drive E:.
</p>

Lastly, I have copied my script from [Post 1](http://codeandkeep.com/PowerShell-ActiveDirectory-Exchange-Part1/) and saved it as SccmPreReqDl.ps1 on my host machine.

```powershell
.\SccmPreReqDl.ps1 -sccmInstallSource E:\ -prereqTargetPath C:\temp\Sccm -Verbose
```

<p>
  It should look something like this while running the script:
</p>

 ![_config.yml]({{ site.baseurl }}/images/SccmPreReqDl.png)

<p>
  You should see something like this folder structure after it has completed:
</p>

```powershell
ls C:\temp\Sccm

    Directory: C:\temp\Sccm

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         2/2/2019   4:45 PM                adk
d-----         2/2/2019   5:08 PM                adkPeAddon
d-----         2/2/2019   4:57 PM                prereq
```

##### Copy files to Hyper-V VM from Host

<p>
  You only need to follow this section if you are downloading on your host, 
  and are running Hyper-V.
</p>

<p>
  Now I need to copy the folder C:\temp\Sccm to my cm1 server.
  If your host is running Hyper-V and you want to copy this via PowerShell, 
  you can do something like the following.
</p>

```powershell
# On Hyper-V host, enable 'Guest Service Interface' on target VM
Enable-VMIntegrationService -VMName cm1 -Name 'Guest*'

# Compress-Archive only available on PowerShell v5 and up
Compress-Archive -Path C:\temp\Sccm\ -DestinationPath C:\temp\SccmPreReq.zip

# Copy zip file to VM
Copy-VMFile -VMName cm1 `
  -FileSource Host `
  -SourcePath 'C:\temp\SccmPreReq.zip' `
  -DestinationPath 'C:\temp\Sccm\SccmPreReq.zip' `
  -CreateFullPath
```

<p>
Now on the VM:
</p>

```powershell
# Expand-Archive only available on PowerShell v5 and up
Expand-Archive -Path C:\temp\Sccm\SccmPreReq.zip -Destination C:\temp\Sccm
```

##### End of Hyper-V file copy

-----

### Sccm PreReq Install
----

I have mounted the Windows Server 2016 ISO to the D:\ drive on cm1, 
and saved the script from 
[Post 2](http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites-Install/) 
to the cm1 server as SccmPreReqInstall.ps1.

```powershell
.\SccmPreReqInstall.ps1 -prereqSourcePath C:\temp\Sccm -windowsMediaSourcePath D:\sources\sxs -Verbose
```

<p>
  After this script has finished, 
  you should have all the necessary Windows Features, 
  and all the required ADK components for the Sccm install.
</p>

### Sccm AD PreReqs
----

I have saved the script from 
[Post 3](http://codeandkeep.com/PowerShell-Sccm-PreRequisites/) 
on the cm1 server as SccmADPreReq.ps1. 
I have mounted the Sccm Installation ISO to the D:\ drive of the cm1 server.
<p>
  I am logged into the cm1 server as a user with membership to 
  the Schema Admins, and the Domain Admins security groups.
</p>

```powershell
.\SccmADPrereq.ps1 -computerName $ENV:COMPUTERNAME -Verbose

# Extend the Active Directory schema
D:\SMSSETUP\BIN\X64\ExtADSch.exe
```

### Sql Install
----

<p>
  The Sql Server installation ISO image has been mounted to the D:\ drive. 
  I have created a service account in AD, 
  that I will use for the Sql Agent and Sql Server services. 
  Those services are not permitted to run as a local account in an 
  Sccm installation.
</p>

<p>
  Since I have opted to use Desired State Configuration (DSC) 
  to automate the Sql install, 
  you will need to download the SqlServerDsc resource from 
  the PowerShell Gallery. 
  I have also setup a certificate on the Sql server that the 
  Dsc local configuration manager will use to encrypt my passwords. 
</p>

<p>
  If your Sql server has internet access:
</p>

```powershell
Install-Module -Name SqlServerDsc
```

<p>
  If your Sql server does not have internet access, 
  run the following command on a computer with internet access: 
</p>

```powershell
Save-Module -Name SqlSeverDsc -Path C:\temp\

# Copy the folder to the Sql server to the PS Modules folder
Copy-Item -Path C:\temp\SqlServerDsc `
  -Recurse `
  -Destination "\\Sql\c$\Program Files\WindowsPowerShell\Modules"
```

#### DSC Configuration
----

Copy the 'Configuration' from 
[Part 4](http://codeandkeep.com/PowerShell-Sccm-PreRequisites-SQL/) 
Directly into an Admin PowerShell window on your Sql server.

Create your DSC configuration data.
Reference the  
[Sql post](http://codeandkeep.com/PowerShell-Sccm-PreRequisites-SQL/) 
 for more information on this if required:


```powershell
$config = @{
  AllNodes = @(
    @{ 
      NodeName = $ENV:COMPUTERNAME 
      # Path to your certificate to use for encryption
      CertificateFile = 'C:\cert\cert.cer';
      
      # This will need to match your Certificate
      Thumbprint = '4B1EF9E6C194098257E93120A4A39DA853F23434'
    }
  )
}
```
<p>
  Now to generate your Sql Mof file:
</p>

```powershell
# Service account created for running Sql server services
$svcCred=Get-Credential

$sccmAdmin='codeAndKeep\cmAdmin'

$installDir='E:\Program Files\Microsoft SQL Server'
$installx86Dir='E:\Program Files (x86)\Microsoft SQL Server'


SccmSqlInstallation -OutputPath C:\temp\sqlInstall `
  -ConfigurationData $config `
  -computerName $ENV:COMPUTERNAME `
  -sqlSourceFiles 'D:\' `
  -features "SQLENGINE,RS" `
  -sqlSvcCredential $svcCred `
  -agentSvcCredential $svcCred `
  -sysAdminAccounts $sccmAdmin `
  -instanceDir $installDir `
  -dataDir $installDir `
  -sharedDir $installDir `
  -sharedWOWDir $installx86Dir
```

<p>
  Now start the configuration:
</p>

```powershell
Set-DscLocalConfigurationManager -Path C:\temp\sqlInstall

Start-DscConfiguration -Path C:\temp\sqlInstall -Wait -Force -Verbose
```

<p>
  Lastly, if you are installing Sql on a separate server, 
  you will need to add the Sccm server as an admin on the Sql server.
</p>

```powershell
Add-LocalGroupMember -Name Administrators -Member 'codeAndKeep\cm1$'
```

### The Sccm Install


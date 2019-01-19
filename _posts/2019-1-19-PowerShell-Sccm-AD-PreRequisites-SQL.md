---
layout: post
title: PowerShell - Automate SQL Install - SCCM AD PreReqs
---

This is a continuation of automating Sccm prerequisites [part 1](
http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites/), 
[part 2](
http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites-Install/) 
and [part 3](
http://codeandkeep.com/PowerShell-Sccm-AD-PreRequisites/
)

<p>
  In this post I will be covering how you can automate the 
  installation of Microsoft Sql Server. 
  We will only install the Sccm required Sql components, 
  Reporting Services and the Sql Engine.
</p>

<p>
  Automatically installing Sql server can be accomplished in a few 
  different ways.
  You can use the command line supported options of the setup.exe 
  file that comes with the Sql installation media. 
  You can also create a Sql install configuration file, 
  and run the same setup.exe file pointing to that configuration file. 
</p>
<p>
  The alternative to these options 
  (which will ultimately use the above approaches under the hood), 
  is to use Desired State Configuration (DSC). 
  If you are not familiar with this product I would highly recommend 
  looking into it. 
  In essence, DSC is a technology that can automate infrastructure. 
  Using DSC, you will be writing code that can configure, build, and 
  maintain your environment (it can also manage Linux systems). 
  Think infrastructure as code. 
  I will be posting a lot more on DSC, 
  and hopefully this has peaked your interest if you are not familiar. 
</p>

### What You Need
----

<p>
  <ol>
    <li>PowerShell v5</li>
      <ul>
        <li>Default on Win10 and Server 2016</li>
        <li></li>
      </ul>
    <li>SqlServerDsc PowerShell Dsc Module</li>
      <ul>
        <li>Download from PowerShell Gallery, directly from PowerShell</li>
      </ul>
    <li>Sql Installation Media</li>
  </ol>
</p>

```powershell
configuration SccmSqlInstallation {
  Param(
    [Parameter(Position=0)]
    [String]$computerName=$ENV:COMPUTERName,

    [Parameter(Position=1)]
    [String]$sqlSourceFiles='C:\sql2016',
    
    [Parameter(Position=2)]
    [String]$sqlInstanceName='MSSQLSERVER',
    
    [Parameter(Position=3)]
    [PSCredential]$agentSvcCredential,

    [Parameter(Position=4)]
    [PSCredential]$sqlSvcCredential,

    [Parameter(Position=5)]
    [String[]]$sysAdminAccounts=(whoami),

    [Parameter(Position=6)]
    [String]$features='SQLENGINE',

    [Parameter(Position=7)]
    [String]$instanceDir="$ENV:ProgramFiles\Microsoft SQL Server",

    [Parameter(Position=8)]
    [String]$dataDir,

    [Parameter(Position=9)]
    [String]$sharedDir="$ENV:ProgramFiles\Microsoft SQL Server",

    [Parameter(Position=10)]
    [String]$sharedWOWDir="${ENV:ProgramFiles(x86)}\Program Files (x86)\Microsoft SQL Server"
  )
  Import-DscResource -ModuleName SqlServerDsc
  Import-DscResource -ModuleName PSDesiredStateConfiguration

  node $computerName {

    WindowsFeature 'NetFramework' {
      Name = 'Net-Framework-45-Core';
      Ensure = 'Present';
    }

    SqlSetup 'SqlInstall' {
      InstanceName = $sqlInstanceName;
      SourcePath = $sqlSourceFiles;
      Action = 'Install';
      Features = $features;
      InstanceDir = $instanceDir;
      InstallSQLDataDir = $dataDir;
      InstallSharedDir=$sharedDir;
      InstallSharedWOWDir=$sharedWOWDir;
      SQLSvcStartupType = 'Automatic';
      AgtSvcStartupType = 'Automatic';
      AgtSvcAccount = $agentSvcCredential;
      SQLSysAdminAccounts = $sysAdminAccounts;
      SQLSvcAccount = $sqlSvcCredential;
      SQLCollation = 'Latin1_General_CI_AS';
      DependsOn = '[WindowsFeature]NetFramework';
    }
    LocalConfigurationManager {
      CertificateId = $node.Thumbprint
    }
  }
}

$config = @{
  AllNodes = @(
    @{ 
      NodeName = $ENV:COMPUTERNAME 
      #PsDscAllowPlainTextPassword = $true;
      #PsDscAllowDomainUser = $true;
      CertificateFile = 'C:\cert\cert.cer';
      Thumbprint = '4B1EF9E6C194098257E93120A4A39DA853F23434'
    }
  )
}


SccmSqlInstallation -OutputPath C:\temp\sqlInstall `
  -computer sql `
  -ConfigurationData $config `
  -sqlInstanceName cm `
  -sqlSourceFiles 'D:\' `
  -sqlSvcCredential $svc `
  -sysAdminAccounts 'codeAndKeep\cmAdmin' `
  -agentSvcCredential $svc `
  -features "SQLENGINE,RS"


Set-DscLocalConfigurationManager -Path C:\temp\sqlInstall

Start-DscConfiguration -Path C:\temp\sqlInstall -Force -verbose -wait
```

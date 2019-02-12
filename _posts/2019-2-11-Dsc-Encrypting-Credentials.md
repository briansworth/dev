---
layout: post
title: Dsc - Encrypting Credentials
---

<p>
  Using credentials in a DSC configuration is almost unavoidable. 
  Storing your credentials in plain text is absolutely avoidable, 
  and should be a requirement for you, in even a lab environment.
</p>

<p>
  To accomplish credential encryption in a DSC capactity, 
  you will need to use certificates. 
  I will go over 2 different ways to do so.
</p>
1. Self Signed Certificate
2. Using PKI (Active Directory Certificate Services)


### Self Signed Certificate
----

<p>
  Starting in Windows 10 / Server 2016, 
  you can create self signed certificates using the 
  New-SelfSignedCertificate cmdlet. 
  If you are on a prior version of Operating System, 
  you can download this cmdlet from the PSGallery using 
  'Install-Module -Name SelfSignedCertificate'.
</p>

<p>
  Since you will be puttin certificates into the local computer 
  certificate store, you will need to be running as admin.
</p>

```powershell
$certFolder='C:\dsc\cert'
$certStore='Cert:\LocalMachine\My'
$pubCertPath=Join-Path -Path $certFolder -ChildPath DscPubKey.cer
$expiryDate=(Get-Date).AddYears(2)

# You may want to delete this file after completing
$privateKeyPath=Join-Path -Path $ENV:TEMP -ChildPath DscPrivKey.pfx

$privateKeyPass=Read-Host -AsSecureString -Prompt "Private Key Password"

if(!(Test-Path -Path $certFolder)){
  New-Item -Path $certFolder -Type Directory | Out-Null
}

$cert=New-SelfSignedCertificate -Type DocumentEncryptionCertLegacyCsp `
  -DnsName 'DscEncryption' `
  -HashAlgorithm SHA512 `
  -NotAfter $expiryDate `
  -KeyLength 4096 `
  -CertStoreLocation $certStore

$cert | Export-PfxCertificate -FilePath $privateKeyPath `
  -Password $privateKeyPass `
  -Force

$cert | Export-Certificate -FilePath $pubCertPath 

Import-Certificate -FilePath $pubCertPath `
  -CertStoreLocation $certStore

Import-PfxCertificate -FilePath $privateKeyPath `
  -CertStoreLocation $certStore `
  -Password $privateKeyPass | Out-Null
```

<p>
  Now have a certificate 'DscEncryption' in your Computer Personal store. 
  You should also have a X.509 certificate file C:\dsc\cert\DscPubKey.cer. 
  You can now utilize this certificate for 
  Dsc configurations on your local machine.
</p>

### ADCS Certificate Template
----

<p>
  As you may have guessed, 
  you will need to have AD Certificate Services setup 
  before you can follow along. 
  I will not cover how to do so here, but maybe in a future post.
</p>

#### Prerequisites
----

<p>
  To utilize a Dsc Certificate Template, 
  you must have a PKI environment setup using ADCS.
</p>

<p>
  You will also want to grab the ADCSTemplateForPSEncryption 
  module from the PowerShell gallery. 
  With that, you will be able to run the command 
  New-ADCSTemplateForPSEncryption. 
  This command will require that the Active Directory PowerShell module 
  is available.
</p>

<p>
  Lastly, the computer you are requesting the certificate on must be on 
  the same domain as the Certificate Authority used. 
</p>

```powershell
# -Publish will publish the certificate to AD, you may not want to do so
New-ADCSTemplateForPSEncryption -DisplayName 'Dsc Template' `
  -AutoEnroll `
  -Publish `
  -Verbose 
```

<p>
  You should now have your certificate template setup. 
  You can check for it by running Get-CATemplate. 
</p>

<p>
  Now you will be able to request a certificate from your domain 
  Certificate Authority server using the newly created Template:
</p>

```powershell
$certFolder='C:\dsc\cert'
$dnsName="$(([Net.Dns]::GetHostByName($ENV:COMPUTERNAME)).HostName)"
$subjectName="CN=$ENV:COMPUTERNAME"

if(!(Test-Path -Path $certFolder)){
  New-Item -Path $certFolder -Type Directory | Out-Null
}

$certReq=Get-Certificate -Template 'Dsc Template' `
  -CertStoreLocation 'Cert:\LocalMachine\My' `
  -Url ldap: `
  -SubjectName $subjectName `
  -DnsName $dnsName

if($certReq.Status -eq 'Issued'){
  $certReq.Certificate | Export-Certificate -FilePath "$certFolder\DscPubKey.cer"
  Write-Output $certReq.Certificate
}elseif($certReq){
  Write-Output $certReq
}
```

<p>
  If you followed the Self Signed Certificate walkthrough, 
  you should be at a similar end point.
  You should have a certificate in your Computer Personal store 
  (under the name of your computer). 
  You should also have a X.509 certificate file C:\dsc\cert\DscPubKey.cer. 
  You can now utilize this certificate for 
  Dsc configurations on your local machine.
</p>

### Credential Encryption Example
----

<p>
  First, verify all the certificate info. 
  We will need the certificate Thumprint, and certificate file location 
  on the local computer.
</p>

#### Verify certificate

```
# If you followed the same path used above
$certPath='C:\dsc\cert\DscPubKey.cer'

$certFile=[Security.Cryptography.X509Certificates.X509Certificate]::CreateFromCertFile($certPath)
$certThumb=$certFile.GetHashString()

$cert=Get-Item -Path "Cert:\LocalMachine\My\$certThumb"
if($cert -and $cert.PrivateKey){
  Write-Host "Certificate looks good. Use the following info"
  New-Object -TypeName PSObject -Property @{
    'CertificateID/Thumbprint' = $certThumb;
    CertPath = $certPath;
  }
}else{
  $warn="Something isn't right. Verify certificate is in correct Store and exported to $certPath"
  Write-Warning $warn
}
```

#### Configure LCM 

```powershell
[DSCLocalConfigurationManager()]
configuration LCMCertConfig {
  Node $ENV:COMPUTERNAME {
    Settings {
      RefreshMode = 'Push';
      CertificateID = $cert.Thumbprint;
    }
  }
}

LCMCertConfig -OutputPath 'C:\dsc\lcm'
Set-DscLocalConfigurationManager -Path 'C:\dsc\lcm' -Force -Verbose
```


#### Push test credential configuration

```powershell
configuration UserInGroup {
  Param(
    [Parameter(Position=0)]
    [String]$GroupName='Administrators',

    [Parameter(Position=1,Mandatory=$true)]
    [String]$UserName, 

    [Parameter(Position=2,Mandatory=$true)]
    [PSCredential]$Credential
  )
  Import-DscResource -ModuleName PSDesiredStateConfiguration

  Node $ENV:COMPUTERNAME {
    Group 'BrianIsAdmin' {
      GroupName = $GroupName;
      UserName = $UserName;
      Credential = $Credential;
      Ensure = 'Present';
    }
  }
}

$configData=@{
  AllNodes=@(
    @{
      NodeName=$ENV:COMPUTERNAME;
      CertificateFile=$certPath;
      Thumbprint=$cert.Thumbprint;
    }
  )
}

$cred=Get-Credential
UserInGroup -UserName 'codeAndKeep\brian' -Credential $cred -OutputPath 'C:\dsc'

Start-DscConfiguration -Path C:\dsc -Force -Wait -Verbose
```

---
layout: post
title: PowerShell - Networking with Windows Routing
---

<p>
  In this post, I will be deploying RRAS (Routing and Remote Access Server). 
  More specifically, I will utilize the 'routing' component of RRAS. 
</p>

### Prelude
----

<p>
  Networking seems kind of like black magic. 
  I have never been too comfortable with, 
  it so recently I've decided to just dive in and figure it out. 
  I want to start by using basic built-in features in Windows, 
  then branch off from there.
</p>

A couple posts back,
([here](http://codeandkeep.com/PowerShell-Get-Subnet-NetworkID/)) I wrote 
a function (Get-IPv4Subnet) that can calculate network subnets based 
on certain parameters you provide. 

<p>
  Armed with this new found power to identify networks, 
  connecting networks together seems like the next logical step.
</p>

### Prerequisites
----

You should have the following server setup:
  - Active Directory Forest
    - 1 or more Domain Controller(s)
  - Test server on separate network
    - Can be a base workstation or server
  - **Server (core or gui) for RRAS**
    - We will be configuring this one

On your Hypervisor, you should have at least to network adapters. 
You can quickly create an internal switch in Hyper-V like so:

```powershell
New-VMSwitch -Name 'Internal1' -SwitchType Internal

# list all your VM Switches
Get-VMSwitch
```

<p>
  I am assuming you have an operating sytem on the VM. 
  Other than that, we will configure this from scratch.
</p>

### VM Configuration
---

I am using my lightweight base template for this server:
  - Server 2016 Core
  - 1 CPU
  - 1GB memory
  - 30GB Disk
  - 2 NICs

<p>
  Verify your VM has 2 NICs in Hyper-V like so:
</p>

```powershell
Get-VMNetworkAdapter -VMName route

Name            IsManagementOs VMName SwitchName MacAddress   Status IPAddresses
----            -------------- ------ ---------- ----------   ------ -----------
Network Adapter False          route  internal1  000000000000        {}
```

<p>
  In this case, I only have 1 NIC, so I need to add my second one. 
  To add or remove VM network adapters, 
  the machine will need to be powered off. 
  With the VM shutdown, add your second VM network adapter in Hyper-V like so:
</p>

```powershell
Add-VMNetworkAdapter -VMName route -SwitchName 'internal2'

# Validate you have 2 now
Get-VMNetworkAdapter -VMName route

Name            IsManagementOs VMName SwitchName MacAddress   Status IPAddresses
----            -------------- ------ ---------- ----------   ------ -----------
Network Adapter False          route  internal1  000000000000        {}
Network Adapter False          route  internal2  000000000000        {}
```

<p>
  One more thing to take note of, 
  is which NIC is assigned to your Domain Controller. 
  You will need to specify the IP configuration with this information.
</p>

```powershell
Get-VMNetworkAdapter -VMName dc1

Name            IsManagementOs VMName SwitchName MacAddress   Status IPAddresses
----            -------------- ------ ---------- ----------   ------ -----------
Network Adapter False          dc1    internal1  00155D38012A        {}
```


### Network Configuration
----

<p>
  Power on your RRAS server and we can setup the network interfaces. 
  You will need to know the IP Address of your domain controller. 
  Since you have 2 NICs on your VM, 
  you will need to know which NIC is on the same VM Switch as your DC. 
  You can use the MAC address to determine which is which:
</p>

```powershell
# On your Host machine
Get-VMNetworkAdapter -VMName route

Name            IsManagementOs VMName SwitchName MacAddress   Status IPAddresses
----            -------------- ------ ---------- ----------   ------ -----------
Network Adapter False          route  internal1  00155D38012D {Ok}   {}
Network Adapter False          route  internal2  00155D38012E {Ok}   {}
```

<p>
  Cross reference your MAC Addresses inside the VM:
</p>

```powershell
# In your RRAS server
Get-NetAdapter

Name        InterfaceDescription                  ifIndex Status   MacAddress
----        --------------------                  ------- ------   ----------
Ethernet    Microsoft Hyper-V Network Adapter #2        2 Up       00-15-5D-38-01-2D
Ethernet 2  Microsoft Hyper-V Network Adapter #3        8 Up       00-15-5D-38-01-2E
```

<p>
  In this example, I can see that 'Ethernet' is my 'internal1' NIC in Hyper-V. 
  This is the same switch that my DC is using. 
</p>

<p>
  Now that I know which Interface can contact my DC, 
  I can setup my IP configuration: 
</p>

```powershell
# My DC's IP is 10.0.0.5 and SubnetMask is 255.255.255.0
# Ethernet will need to be on this same network
New-NetIPAddress -InterfaceAlias Ethernet `
  -IPAddress 10.0.0.1 `
  -PrefixLength 24

# Use my DC as my DNS server
Set-DnsClientServerAddress -InterfaceAlias Ethernet `
  -ServerAddresses 10.0.0.5

# Setting up my second network on 10.0.1.1 with subnetmask 255.255.255.0
New-NetIPAddress -InterfaceAlias 'Ethernet 2' `
  -IPAddress 10.0.1.1 `
  -PrefixLength 24
```

<p>
  Now we can join the domain:
</p>

```powershell
$domainCred=Get-Credential

# Remove the -NewName param if you computer is already named correctly
Add-Computer -DomainCredential $domainCred `
  -DomainName codeAndKeep.com `
  -Restart `
  -NewName 'route'
```

#### Install RRAS roles

<p>
  Now to install the require Windows Features for the RRAS role, 
  and install the routing service 
</p>

```powershell
### Windows Features
$features=@(
  'RemoteAccess',
  'DirectAccess-VPN',
  'Routing',
  'Web-Server',
  'Web-WebServer',
  'Web-Common-Http',
  'Web-Default-Doc',
  'Web-Dir-Browsing',
  'Web-Http-Errors',
  'Web-Static-Content',
  'Web-Health',
  'Web-Http-Logging',
  'Web-Performance',
  'Web-Stat-Compression',
  'Web-Security',
  'Web-Filtering',
  'Web-IP-Security',
  'Web-Mgmt-Tools',
  'Web-Scripting-Tools',
  'Windows-Internal-Database',
  'GPMC',
  'RSAT',
  'RSAT-Role-Tools',
  'RSAT-RemoteAccess',
  'RSAT-RemoteAccess-Powershell'
)
Install-WindowsFeature -Name $features

Install-RemoteAccess -VpnType RoutingOnly
```

<p>
  Believe it or not, that is all we need to do. 
  You can jump onto your test machine on your secondary network to verify:
</p>

```powershell
# On your test machine on your secondary network
$nic=Get-NetAdapter
New-NetIPAddress -InterfaceIndex $nic.ifIndex `
  -IPAddress 10.0.1.50 `
  -PrefixLength 24 `
  -DefaultGateway 10.0.1.1

# Use the DC as your DNS server for further integration
Set-DnsClientServerAddress -InterfaceIndex $nic.ifIndex `
  -ServerAddresses 10.0.0.5

# Now to test
ping 10.0.0.5

nslookup codeAndKeep.com
```

<p>
  Most environments are probably going to use 
  hardware routers for this kind of thing. 
  For a small lab, it might be possible to install a more feature rich 
  router / firewall as a VM to get a more realistic environment.
</p>

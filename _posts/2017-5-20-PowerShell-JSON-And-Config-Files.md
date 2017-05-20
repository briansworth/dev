---
layout: post
title: PowerShell: Configuration Files in JSON 
---

If you run a lot of code in any environment, or have multiple environment that you manage, configuration files are incredibly useful.

In the Windows world, it seems that these files are typically created in xml.  Xml will work, and if you are stuck with PowerShell version 2.0 or earlier, than you may run have to stick with xml.

For those of us living in a more modern world, JSON is the way to go.  It is much easier to read, in my opinion, and takes up much less space than xml would.  This makes JSON much simpler and efficient to work with.

Example:
**XML**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<hyperv>
  <vmpath>V:\VMs</vmpath>
  <templatepath>V:\Templates</templatepath>
  <vmswitch> 
    <external>External vSwitch</external>
    <internal>Internal1 vSwitch</internal>
    <internal>Internal2 vSwitch</internal>
    <private>>Private vSwitch</private>
  </vmswitch>
  <support>
    <email>brian@codeAndKeep.com</email>
    <phone>555-555-5555</phone>
  </support>
</hyperv>
```
Here's a simple xml example with some info for a Hyper-V config file.  
<br>
Now let's see the same info in JSON.

**JSON**
```
{
  "hyperv": {
    "vmpath": "V:\\VMs",
    "templatepath": "V:\\Templates",
    "vmswitch": {
      "external": "External vSwitch",
      "internal": [
        "Internal1 vSwitch",
        "Internal2 vSwitch"
      ],
      "private": "Private vSwitch"
    },
    "support": {
      "email": "brian@codeAndKeep.com",
      "phone": "555-555-5555"
    }
  }
}
```
The way I have formatted this makes it 3 lines longer in this example.
That being said; I find this much easier to read, as there is much less clutter.  In JSON it is much easier to arrays than it is in xml (vmswitch -> internal).  


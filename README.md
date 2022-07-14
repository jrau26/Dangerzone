# Dangerzone Domain
## Table of contents  

* Introduction  
* Creating the Lab Environment  
* Our First Domain Controller  
* Joining dangerzone  

### Introduction  

The Dangerzone domain was created to follow [John Hammond's Active Directory series on youtube](https://www.youtube.com/watch?v=pKtDQtsubio).  
It's purpose is to actively learn about AD, VMware, and Powershell for the purposes of System administration with a focus on networking.    
 
What follows are the steps taken in creating my own Virtual machines  
and domain with the aim of creating clear and concise documentation.  

### Creating the Lab Environment

In order to streamline the proccess of creating multiple VMs template VMs are made.  
The server template has a fresh install of Windows server core 2022 that is updated with sconfig at time of writing.  
The client template has a fresh install of Windows 11 with [TPM bypassed](https://www.bleepingcomputer.com/news/microsoft/how-to-bypass-the-windows-11-tpm-20-requirement/), and a local user account that has admin privilege called "local_admin"  
After installing VMware tools and rebooting a snapshot is taken. Template mode enabled so that linked clones can be made.  

[It is worth noting that linked clones require the source image in order to run, and that sebsequent clones of that source image cannot be made while the 
linked clones are running.](https://docs.vmware.com/en/vCenter-Converter-Standalone/6.2/com.vmware.convsa.guide/GUID-F09DE391-C05C-48F4-BDB2-CB12F12B434A.html)  

### The First Domain Controller

To create the domain controller  clone the server template, and call it DC1.  
Change the hostname to DC1 with ``rename-computer``.  
The templates were created with a NAT network adapter. [By default VMware will put the interface within the scope of a DHCP address range.](https://docs.vmware.com/en/VMware-Workstation-Pro/16.0/com.vmware.ws.using.doc/GUID-9831F49E-1A83-4881-BB8A-D4573F2C6D91.html#GUID-9831F49E-1A83-4881-BB8A-D4573F2C6D91)  
We want the DC to have a static IP with it's DNS settings pointing to itself.  
Using the ``new-netipaddress`` cmdlet create an IP address within the static range of VMwares NAT adapter specifying the interfaceindex, prefix length, and default gateway.  
Using the ``Set-DnsClientServerAddress`` cmdlet set the DNS to point to IP address previously set, specifying the correct interfaceindex.  
Confirm the changes with ``get-netipconfiguration``.  
It is possible to make all of these changes with ``sconfig``.  

While it is possible to directly admin the server in this manner, we may want the option to remotely manage it through PSremoting.  
In order to do this we need to configure winRM to allow connections. This is done with the command ``winRM quickconfig``  

To create the management client, clone the client template and call it management client.  
For PSremoting to work this machine needs to either be added to the domain of the host it's remoting to (the domain controller in this case), or, [added to winRM's Trusted Hosts.](https://techdirectarchive.com/2020/03/23/how-to-add-trusted-host-for-the-winrm-client/)  
Once the DC is added to the trusted hosts, we can remote in with ``new-pssession`` specifying the computername and credentials.  
Now that the session is created, we can enter and exit it with ``enter-pssession`` and ``exit-pssession`` without ending it.  

I remote in to DC1 and now is time to [install Active directory.](https://xpertstec.com/how-to-install-active-directory-windows-server-core-2022/)  
Using the ``install-windowsfeature AD-Domain-Services -includemanagmenttools`` cmdlet install AD services with managment tools.  
Confirm install with ``get-windowsfeature ad*``  
``import-Module ADDSDeployment``  
the cmdlet ``install-ADDSForest`` will create our domain. this will take some time and the server will restart.  
During the installtion, the DNS setting is set to the loopback address 127.0.0.1, which we can set back to the static IP with the ``DNS-clientserveraddress`` cmdlet.  

Now the Domain controller, and Domain itself has been created. At this point we can take a snapshot of the DC  

### Joining dangerzone  


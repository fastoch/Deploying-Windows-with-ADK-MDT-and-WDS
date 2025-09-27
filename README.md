# Introduction

ADK, MDT, and WDS each play distinct roles in the Microsoft **Windows deployment ecosystem**.  

## ADK

ADK (Windows **Assessment and Deployment Kit**) is a set of tools and technologies that help IT professionals customize, assess, and deploy Windows operating systems.  

It includes **utilities** to capture and apply Windows **images** and is foundational for deployment automation, often leveraged by MDT.  

## MDT 

MDT (**Microsoft Deployment Toolkit**) is a unified collection of tools, processes, and guidance **built on top of ADK** to automate the deployment of Windows desktops and servers.  

It helps create standardized Windows images and manages deployment through **task sequences** that orchestrate OS installation, driver injection, application installation, and configuration.  

MDT is designed for medium to large enterprises to reduce deployment time, ensure consistency, and facilitate upgrade and migration scenarios.

## WDS

WDS (**Windows Deployment Services**) is a **network-based** deployment server role in Windows Server.  

It enables the deployment of Windows OS images over the network using **PXE boot** technology.  
PXE stands for **Preboot Execution Environment**, and relies on DHCP for IP addressing.  

**Reminder**:  
DHCP is what assigns an IP address to the clients and provides them with the information needed to locate the PXE server.  

WDS is primarily responsible for hosting and delivering boot and install images to client machines.  
It is simpler than MDT and requires pre-built images, often created via **MDT** or other tools.  

WDS can deploy images to multiple machines simultaneously but lacks MDT’s extensive automation and customization features.

## How they work together

- ADK provides the deployment tools and image management foundation
- MDT automates and orchestrates deployment workflows using ADK tools
- WDS handles network-based image delivery to client devices.

Often, **MDT** and **WDS** are **used together** for full deployment solutions, where:
- MDT creates and customizes the deployment images and task sequences,
- while WDS is responsible for delivering them over the network.

---

# Configuring an automated deployment ecosystem

- setting up the VMs via VMware Workstation or a similar tool
- setting up the WDS server 
- setting up the DHCP server (to allow PXE boot in both UEFI and BIOS modes)
- setting up ADK and MDT

## Setting up our Virtual Windows servers

We can use a tool like VMware Workstation to create our VMs.
- Once we have set up the ADDS (domain controller) server, we need to add the DHCP role to it.  
- After that, we need to set up a dedicated Windows Server VM with the WDS role
- The WDS server then needs to be registered to our AD domain
- It's **recommended** to create an **NTFS partition** on our WDS server.
  - That partition will host our images, PXE boot files and other WDS-related files.
- We also need 2 PXE clients (W10 and W11) to test our PXE boot
  - For the Windows 10 client, we can modify the VM settings to test both BIOS and UEFI (with secure boot enabled or not)
  - For the Windows 11 client, we don't have a choice but to use UEFI (with secure boot enabled or not)
 
**Note**:  
Of course, our client VMs don't have any OS installed yet, we've just specified the future OS when creating the VMs via VMware Workstation.

## Networking side note

As an example, the IP addresses of our VMs can be: 
- 192.168.16.10 for the "ADDS DC/DNS/DHCP" server
- 192.168.16.11 for the WDS server
- and 192.168.16.12 and 192.168.16.13 for the W10 and W11 clients 
They just need to be part of the same network.  

And if in real life, our servers are in a different network from the machines to be deployed (different VLANs, for example), we must declare the IP address of 
the DHCP server and the WDS server in the DHCP relay options of our gateway (router/firewall).

## Setting up the WDS server

- WDS needs a **DHCP** server so it can deliver images via **PXE boot**
- A WDS server can either be independent (workgroup) or registered in an Active Directory domain
- WDS can provide 2 types of images: boot images (such as boot.wim) and installation images (such as installation.wim)

When a machine runs a PXE boot, its firmware uses one of these 2 modes:
- BIOS mode (also called "Legacy mode")
- UEFI mode (associated with secure boot)

**Note**:  
This is true for deploying Windows 10 images, but **Windows 11** requires to use **UEFI** to align with the **TPM 2.0** prerequisites.  

### Installing the WDS role on our WDS VM

- The VM is running Windows Server 2022 and has been joined to the AD domain provided by our ADDS/DHCP VM
- Open "Server Manager"
- Click on "Manage" and select "Add Roles and Features"
- Choose "Role-based or feature-based installation"
- Select the server where you want to install the role 
- In the list of roles, check "Windows Deployment Services"
- Click on Next until the Role Services selection
- Ensure both role services are selected: Deployment Server and Transport Server
- Click on "Install"

### Configuring WDS

- once the WDS role installed, open the Windows Deployment Services console
- in the left pane, open the Servers section and right-click on the WDS server > Configure server
- we get reminded that we need a DHCP server and a DNS server active on our network, along with an NTFS partition to store images
- in our example, the WDS is not independent and is part of an AD DS domain
- then we need to specify the path to the remote installation folder (inside the NTFS partition, this is where we'll store our images)
  - we can name this folder "RemoteInstall", in which case the path will look like "W:\RemoteInstall"
- We need to specify which clients our WDS server will respond: choose "respond to all clients"
- the WDS config is over, we can uncheck "add images to the server now" before clicking on "Finish"

In the left pane, we now have new "folders": installation images, boot images, pending devices, drivers, etc.

### Adding boot images & installation images

- we need to load an ISO image of W10 or W11
- we can deploy W10 images via WDS, so we just need to load a W10 ISO in our WDS VM (removable devices > CD/DVD > Connect)
- but keep in mind that we cannot deploy W11 images only via WDS, we need to combine WDS with MDT (more on that in video #2)
- once we've loaded a W10 ISO in our WDS VM, we need to go back to the WDS console and right-click on "boot images" to add a boot image
  - browse to the virtual DVD drive > "sources" folder > boot.wim
  - name the image as you wish
- same process to add an installation image with a right-click on "installation images"
  - this time, select the install.wim file
  - specify which OS versions you want to add to the server

Once we've added the installation images to our WDS server, we'll be able to PXE boot from our future Windows PC clients:
- The client boots from the network (PXE boot)
- It loads up the boot image (boot.wim)
- It then starts installing the Windows OS of our choice (install.wim)

### Important note about installation images

When we'll use **MDT**, we won't need to use these installation images anymore.  
Instead, we'll be relying on the MDT server to deliver the installation images.  

## Setting up the DHCP server

We need to add the DHCP role and configure it so it can handle both UEFI and BIOS PXE boot.  
- https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/quickstart-install-configure-dhcp-server?tabs=powershell
- https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/quickstart-install-configure-dhcp-server?tabs=gui

### Prerequisite: ADDS + DNS

-  This VM will be running Windows Server 2022
-  We need to add the AD DS role to this server and then promote it into a domain controller (DC)
-  Promoting the server to a DC will allow us to install and configure the DNS role 

### DNS side note

When creating an Active Directory Domain Services (ADDS) server, the DNS Server role is not automatically installed by default.  
But the AD DS installation wizard offers the option to automatically install and configure the DNS server.  
This automatic addition and configuration happens during the **promotion** of the server to a **domain controller**.  
The DNS zones created during this process are integrated with the AD DS domain namespace.  

**Documentation**:  
https://learn.microsoft.com/en-us/windows-server/networking/dns/quickstart-install-configure-dns-server?tabs=gui

### Configuring the DHCP role for PXE boot in BIOS mode

- We now have an Active Directory (AD) Domain Controller (DC) to which we must add the DHCP role.  
- Once the DHCP role has been added, we need to create a new IPv4 scope
- For that, open the "DHCP" console > right-click on IPv4 > new IP scope
  - name this IP scope "Deployments"
  - the scope could go from 192.168.16.100 to 192.168.16.128 (with a /24 subnet mask, which is 255.255.255.0)
  - no need to add any exclusion or delay
  - the lease term can be set to 4 hours (enough to deploy the OS on the client machine)
  - say Yes to configure the DHCP settings right away
  - specify the default gateway address: for example, 192.168.16.1 in a network with the subnet 192.168.16.0/24
  - the domain name and the IP address for our DNS server should be automatically detected
  - no need to use WINS servers
  - select Yes to enable the new IP scope
- After creating our new "Deployments" IP scope, we need to go to the "Scope options" section
- https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/quickstart-install-configure-dhcp-server?tabs=gui#manage-scope-options
  - from there, right-click > configure options
  - options 3, 6 and 15 should already be configured (router, DNS servers and DNS domain name)
  - option 66 needs to be checked and set with the IP of our WDS server (192.168.16.11 in our example)
  - option 67 needs to be checked and set with the boot file name: boot\x64\wdsnbp.com (WDS network boot program)

Our DHCP server is now ready for running PXE boots in BIOS mode.  
We can easily test that by:
- going to our W10 client VM,
- checking the VM advanced settings > Firmware type must be set to BIOS
- starting the VM and making sure that it sees the WDS server and can load the boot.wim image
- once the W10 installation wizard starts, we can consider that the PXE boot is working 

### Configuring the DHCP role for PXE boot in both UEFI & BIOS modes

Option 60 is known as the Vendor Class Identifier (VCI). It is used mainly in PXE (Preboot Execution Environment) boot scenarios to identify the client type to the DHCP server.

- in the DHCP console, let's go back to the "Deployments" scope options and remove options 66 and 67
- to add the option 60, we need to run the following cmd in powershell:
  ```
  Add-DhcpServerv4OptionDefinition -ComputerName SRV-ADDS-01 -Name PXEClient -Description "PXE Support" -OptionId 060 -Type String
  ```
  Of course, replace SRV-ADDS-01 with your actual ADDS/DHCP server name
- go back to the DHCP console, click on Scope options under the "Deployments" scope
- and you should now see option 60

After that, we need to declare 3 vendor classes.  
These vendor classes will allow our DHCP server to identify the PXE client and to determine whether it's using BIOS or UEFI.  

To define our vendor classes, we will run the following powershell script:
```
# Let's start by defining 3 variables that we will use multiple times in our script:

# DHCP server hostname
$DhcpServerName = "SRV-ADDS-01"
# WDS (PXE) server IP address
$PxeServerIp = "192.168.16.11"
# Network address of the targeted DHCP scope
$Scope = "192.168.16.0"

# Defining the 3 DHCP vendor classes we'll need to cover all use cases:

Add-DhcpServerv4Class -ComputerName $DhcpServerName -Name "PXEClient - UEFI x64" -Type Vendor -Data "PXEClient:Arch:00007" -Description "PXEClient:Arch:00007"
Add-DhcpServerv4Class -ComputerName $DhcpServerName -Name "PXEClient - UEFI x86" -Type Vendor -Data "PXEClient:Arch:00006" -Description "PXEClient:Arch:00006"
Add-DhcpServerv4Class -ComputerName $DhcpServerName -Name "PXEClient - BIOS x86 et x64" -Type Vendor -Data "PXEClient:Arch:00000" -Description "PXEClient:Arch:00000"

# Now, we need to define policies for each of our 3 vendor classes, which we'll also do via our powershell script:

# First policy
$PolicyNameBIOS = "PXEClient - BIOS x86 et x64"
Add-DhcpServerv4Policy -Computername $DhcpServerName -ScopeId $Scope -Name $PolicyNameBIOS -Description "Options DHCP pour boot BIOS x86 et x64" -Condition Or -VendorClass EQ, "PXEClient - BIOS x86 et x64*"
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 066 -Value $PxeServerIp -PolicyName $PolicyNameBIOS
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 067 -Value boot\x64\wdsnbp.com -PolicyName $PolicyNameBIOS

# Second policy
$PolicyNameUEFIx86 = "PXEClient - UEFI x86"
Add-DhcpServerv4Policy -Computername $DhcpServerName -ScopeId $Scope -Name $PolicyNameUEFIx86 -Description "Options DHCP pour boot UEFI x86" -Condition Or -VendorClass EQ, "PXEClient - UEFI x86*"
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 060 -Value PXEClient -PolicyName $PolicyNameUEFIx86
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 066 -Value $PxeServerIp -PolicyName $PolicyNameUEFIx86
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 067 -Value boot\x86\wdsmgfw.efi -PolicyName $PolicyNameUEFIx86

# Third policy
$PolicyNameUEFIx64 = "PXEClient - UEFI x64"
Add-DhcpServerv4Policy -Computername $DhcpServerName -ScopeId $Scope -Name $PolicyNameUEFIx64 -Description "Options DHCP pour boot UEFI x64" -Condition Or -VendorClass EQ, "PXEClient - UEFI x64*"
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 060 -Value PXEClient -PolicyName $PolicyNameUEFIx64
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 066 -Value $PxeServerIp -PolicyName $PolicyNameUEFIx64
Set-DhcpServerv4OptionValue -ComputerName $DhcpServerName -ScopeId $Scope -OptionId 067 -Value boot\x64\wdsmgfw.efi -PolicyName $PolicyNameUEFIx64
```

In the DHCP console, if we right-click on IPv4 and click on "define vendor classes", we should now see our 3 classes.  
Same thing if we go to our IPv4 scope and click on "Policies": we should see the 3 policies we've just defined.  

## Testing PXE boot on BIOS and UEFI VMs

We can now test PXE boot in different modes by: 
- modifying the Firmware type in our VM client settings (W10 can be BIOS or UEFI, while W11 has to be set to UEFI)
- and then try to run a PXE boot for each VM to confirm that it is working for both BIOS and UEFI modes

## Installing MDT on Windows Server 2022 to deploy Windows 11

The objective is to combine WDS with MDT to improve our deployment process by adding task sequences to:
- define the client hostname
- join it to our Active Directory (AD) domain
- handle partitioning of the client disks
- handle BitLocker encryption
- install drivers that match the targeted hardware
- install applications on the client
- execute custom scripts for various purposes
- etc.

With MDT, we'll be doing "**Lite Touch**" deployment, which means there's a bit of human intervention.  


---
**sources**:  
- video #1: https://youtu.be/ILon8Quv924?si=NWygllLZPJ2hJXi4
- tuto #1: https://www.it-connect.fr/serveurs-dhcp-wds-boot-pxe-bios-et-uefi/
- video #2: https://youtu.be/bx374BP8I6A?si=IxrKPmQkhy1Bw3Qg
- tuto #2: https://www.it-connect.fr/installer-mdt-sur-windows-server-2022-pour-deployer-windows-11-22h2/

@22/22 (video 1/2)  
@2/37 (video 2/2)

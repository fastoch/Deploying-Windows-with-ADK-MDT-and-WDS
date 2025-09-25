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

WDS is primarily responsible for hosting and delivering boot and install images to client machines.  
It is simpler than MDT and requires pre-built images, often created via **MDT** or other tools.  

WDS can deploy images to multiple machines simultaneously but lacks MDT’s extensive automation and customization features.

## How they work together

- ADK provides the deployment tools and image management foundation
- MDT automates and orchestrates deployment workflows using ADK tools
- WDS handles network-based image delivery to client devices.

Often, MDT and WDS are **used together** for full deployment solutions, where MDT creates and customizes the deployment images and task sequences, while WDS delivers them over the network.

---

# Configuring an automated deployment ecosystem

- setting up the WDS server 
- setting up the DHCP server (to allow PXE boot in both UEFI and BIOS modes)
- setting up ADK and MDT

## The WDS server

- WDS needs a **DHCP** server so it can deliver images via **PXE boot**
- A WDS server can either be independent (workgroup) or registered in an Active Directory domain
- WDS can provide 2 types of images: boot images (such as boot.wim) and installation images (such as installation.wim)

When a machine runs a PXE boot, its firmware uses one of these 2 modes:
- BIOS mode (also called "Legacy mode")
- UEFI mode (associated with secure boot)

**Note**:  
This is true for deploying Windows 10 images, but **Windows 11** requires to use **UEFI** to align with the **TPM 2.0** prerequisites.  

## The DHCP server

This will be an Active Directory (AD) Domain Controller (DC) to which we'll add the DHCP role.  

## Setting up our Virtual Windows servers

We can use a tool like VMware Workstation to create our VMs.
- Once we have set up the ADDS (domain controller) server, we need to add the DHCP role to it.  
- After that, we need to set up a dedicated Windows Server VM with the WDS role
- The WDS server then needs to be joined to our AD domain
- We also need 2 PXE clients (W10 and W11) to test our PXE boot
  - For the Windows 10 client, we can modify the VM settings to test both BIOS and UEFI (with secure boot enabled or not)
  - For the Windows 11 client, we don't have a choice but to use UEFI (with secure boot enabled or not)

The IP addresses of our VMs can be 192.168.16.10 and 16.11 and 16.12, for example. They just need to be part of the same network.  

**Note**:  
Of course, our client VMs don't have any OS installed yet, we've just specified the future OS when creating the VM via VMware Workstation.

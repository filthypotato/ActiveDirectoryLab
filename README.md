## Introduction

In this lab, I set up an Active Directory Domain Controller using Windows Server 2025 and connected a Windows 10 client to simulate a real-world enterprise environment.
By us connecting a Windows 10 client machine, and then using a PowerShell script to generate over 1000 user account, allows us for
practical simulation and hand on learning with Active Directory, Group Policies, user management, and domain based configuration.

This homelab allows me to practice:

- Domain configuration
- Group Policy
- User management
- Attack simulations
- Blue team logging & monitoring

## Network Overview

- Windows Server (Domain Controller)
- Windows 10 Client
- Internal NAT network
- Static IP for DC
- DNS pointed to DC

## Prerequisites

Files needed to follow along:

  > **Note:** This lab was built using virt-manager on a Linux host (KVM/QEMU)
  > If you are using VMWare or VirtualBox, steps may differ.

### Virtualization Platform
  - [virt-manager (KVM/QEMU)](https://virt-manager.org/)
  - Virtualization must be enabled. (Intel VT-x or AMD-V)

Terminal command to install rerequisites.

```bash
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat qemu-full virt-manager libvirt edk2-ovmf dnsmasq bridge-utils
```
Enables and start libvirtd

```bash
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```
Checks if running

```bash
systemctl status libvirtd
```
Adds yourself to the libvirt group

```bash
sudo usermod -aG libvirt $USER
```
Then **log out** and log back
 (or reboot)


### Installations Files

 - [Windows Server 2025 ISO](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)
 - [Windows 10 ISO](https://www.microsoft.com/en-us/software-download/windows10)

### Verify Virtualization Support

Run:

```bash
lscpu | grep Virtualization
```

You should see **`VT-x`** or **`AMD-V`**


# Procedures

## Setting up Windows Server 2025

Download Virt Manager by using the steps above if you have not already.

Open Virt Manager in Terminal:
```bash
virt-manager
```

Click `Create a new virtual machine`

- `Local install media (ISO Image, or CDROM)` > Forward

- `Browse` for ISO Image

- Select ISO Image for Windows Server 2025

- Make sure on Chose the operating system you are installing -> `Microsoft Windows Server 2025` > Foward

- Click `yes` on the next window

These are the setup options that I used:
- Hardware
  - Memory: 4096MiB
  - CPUs: 2
- Virtual Hard Disk:
  - 60.00 GiB
    - My boot drive did not have enough space, so `Select or create custom storage`

- Network selection
  - Virtual Network 'default': NAT
    - This prevents exposing your IP and Host system to the VM.

- Now `Finish!`

**Great! You now have the Virtual Machine setup, the next steps will involve the Windows Setup Wizard inside of the VM**

# DC: Windows Server Setup Wizard

Select Options as they appear:
  - Select the operating system you would like installed

    - Windows Server 2025 Standard Evaluation (Desktop Experience)

  - Which type of installation do you want?

    - Custom: Install Windows only (advanced)

    - Click next to the following screem to select your default partition.

  - Customize Settings:

    - Password: Password1

        - (Easy password makes it easy for lab setup)

# Configuring NIC's (Internal/External Networks)

Check Default NAT Network
  - In Arch termnial:

  ```bash
  virsh net-list --all
  ```
You should see:

  - default active

If this unfortunately does not work, try these commands:

```bash
  sudo virsh net-start default
  sudo virsh net-autostart default
```
***This is the INTERNET network!***


# Create Internal Network (INTNET)

Open virt-manager:

  - Edit -> Connection Details

  - Virtual Networks

  - Click + (Add Network)

Configure:

  - Name: `intnet`

  - Mode: `Isolated`

  - IPv4:

    - Address: 172.16.0.1

    - Netmask: 255.255.255.0

  - Disable DHCP

*Finish*

**Now you should have:**

  - `default` -> internet

  - `intnet` -> internal lab


# Inside Your Windows Server VM

Now we will rename the Network Adapters

1: Open:
  - Control Panel

  - Network and Sharing Center

  - Change Adapter Settings

You should see and rename:

  - Ethernet -> _INTERNET_

  - Ethernet 2 -> _INTNET_

Right click the `_INTNET_` -> Properties
Internet Protocol Version 4 -> Properties

Set:

  ```
  IP: 172.16.0.1
  Mask: 255.255.255.0
  Gateway: (blank)
  DNS: 127.0.0.1
  ```
Now we can rename the Server to DC

- Start

  - System

  - Rename this PC

Rename to:

  `DC`

***Now Restart***

Now lets attach the Networks to the Windows Server VM

Go inside `virt-manager`

  - Right click Windows Server VM

  - Open

  - Show Hardware Details

Add:

  NIC 1:

  - Source: `default`

  - Device mode: `e1000` (you can use virtio if you have the drivers installed, but for this I do not)

  NIC 2:

  - Source: `intnet`

  - Device model: `e1000`

  **Apply**

  *Now run VM!*

Open Command Prompt inside of your Windows VM

```ngnix
ipconfig
```

You will see:

```nginx
_INTERNET_ -> DHCP address 192.168.x.x (or close too)
_INTNET_ -> 172.16.0.1
```
This means it is working!

# Configuring Active Directory Domain Services

Now that we have the configured NIC's, we can setup our Domain (Active Directory Domain Server).

Open `Server Manager` -> select `Add role and features`
Click next until you get to `Select destination server/Server Selection` tab.

Wait until `Server Roles` window pops up.

  - Select Active Directory Domain Services, click Add Features, click Next.
  - Click Next through the rest of the setup, click Install.
  - Click Close after install completes.

Now on Server Manager, there will be a Notification icon in top right (will be a yellow caution symbol).
Select it and select "Promote this server to a domain controller".

This will open the Active Directory Services Configuration Window, you can choose these options as follows.

  - Deployment Configuration:
    - Add new forest.
    - Root domain name: mydomain.com

  - Domain Controller Options:
    - Wait for window to load..
      - Password & Confirm Password: Password1 

  - Click Next through the rest of the settings, click Install

# Creating dedicated domain admin user account

Open Active Directory Users and Computers.

Right-click on `mydomain.com` > Select New > Organizational Unit

  - Name: "_ADMINS"
    - Right-click "_ADMINS" group > New > User

  - For this part setup the Admin account with your information using this format:
    - First name: Peter
    - Last name: Smith
    - User logon name: p-smith@mydomain.com

  - Next
    
    - Password & Confirm Password: Password1
    - Uncheck "User must change password at next logon"
    - Check "Password never expires"

  - Next > Finish

In order to set your New Account as a Domain Admin, select the user account, right-click > Properties.

  - Member Of > Add
  - In "Enter the object names to select (examples):"
    - Type "domain admins".
    - Check Names
    - Click OK, Apply, then OK.

Now we are able to sign out and select the new user account we created.

Go ahead and sign out, then on the Sign-In screen, select Other Users, and enter in the credentials for your user you created.
  - Example:
    - Username: p-smith
    - Password: Password1

# Network Configuration

## Configure Remote Access Server (RAS)/NAT

Now it is time to setup a RAS/NAT to allow our client virtual machines to be on a private virtual network,
but we still want to be able to access the Internet through the DC.

Open Server Manager > select "Add roles and features".

Click Next through the Wizard, and select these options as they appear:

  - [Server Selection] Select the DC
  - [Sever Roles] Check Remote Access
  - [Role Services] Check DirectAccess and VPN (RAS)
    - Click Add Features
    - Check Routing 
  - Click Next through the rest of the Wizard, click Install
  - When done, click Close

Navigate to Server Manager > Tools > Routing and Remote Access

  - Right-click on 'DC (local)' and select Configure and Enable Routing and Remote Access
  - Click nect through the Wizard, and select these options as they appear:
    - [Configuration] Select Network address tranlation (NAT)
    - [NAT Internet Connection] Select "Use this public interface to connect to the Internet"
      - Select _INTERNET_

      > **NOTE:**
      > If these options are unavaliable, try closing and reopening Routing and Remote Access

  - Continue clicking Next through the Wizard then click Finish.

Now there will be a green symbol next to "DC (local)" in Routing and Remote Access,
the setup has been configured ***correctly!*** Now our client VMs should be able to connect
once we setep DCHP for them!


# Configuring DCHP Server

Open Server Manager > select "Add roles and features".

  - Click next through the Wizard, choosing these options as they appear:
    - [Server Selection] Select DC.mydomain.com
    - [Server Roles] Select DHCP Server > Add Features
    - Click Next through the rest of the settings, click Install.
  - Navigate to Server Manager > Tools > DHCP
    - Expand the dc.mydomain.com drop-down menu.
    - Right-click on IPv4 and select New Scope…
  - Click next through the Wizard, choosing these options on the following tabs:
    - [Scope Name] Name: 172.16.0.100-200
    - [IP Address Range]
      - Start IP Address: 172.16.0.100
      - End IP Address: 172.16.0.200
      - Length: 24
      - Subnet mask: 255.255.255.0
    - [Router (Default Gateway)] IP address: 172.16.0.1 (and click Add!)
  - Click Next through the rest of the settings, and click Finish.
    - Right-click on dc.mydomain.com and select Authorize.
    - Right-click on dc.mydomain.com and select Refresh.
    - A green symbol should appear next to IPv4 and IPv6 to show that DHCP is now properly configured!

# How to enable internet access from this virtual machine
 > **Note:**
 > In a real production environment, we do not want to allow the domain server to access the Internet.

  - Inside of Server Manager:
    - Select the Local Server tab on the left-hand side.
    - Find the properties set for “IE Enhanced Security Configuration: On”.
    - Change the setting to Off for both Administrators and Users.




**This is currently a work in progress..**

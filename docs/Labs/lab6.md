---
id: lab6
title: Lab 6 - Active Directory
sidebar_position: 6
description: Lab 6 - Active Directory
--- 

# Lab 6 - Active Directory

## Lab Preparation

### Purpose of Lab 6

In this lab, you’ll redesign and configure your environment to a more realistic, low-footprint, on-premises design. You’ll move core network services off **srv1** and into **Active Directory** by promoting **srv2** to **Domain Controller (DC1)** with integrated **DNS** and **DHCP**, manage everything remotely from **srv1** using **RSAT**, and prove end-to-end client functionality by joining **laptop1** to the domain.

You’ll also prepare a clean clone (**srv3**) and promote it to **DC2** for redundancy.

The lightweight **router** VM you'll create replaces RRAS/NAT, mirroring how real sites use a dedicated edge device.

### Objectives

By the end of this lab, you will be able to:

- Build a minimal **VyOS router** VM and configure LAN/WAN, default route, and NAT for Internet access.

- Configure a consistent **IP plan** (gateway 10.0.UID.254, statics for servers, DHCP range for clients).

- Promote **srv2** to **AD DS** as the first domain controller for **yourSenecaUsername.com** with integrated **DNS**.

- Configure **DNS forwarders** and create a **reverse lookup zone** with **Secure only** dynamic updates.

- Convert **srv1** into a **management server** (RSAT only) and manage **srv2** remotely via **Server Manager**.

- Join **srv1** and **laptop1** to the domain and verify **Kerberos + name resolution** end-to-end.

- Validate DNS registration: confirm **A** and **PTR** records for servers and client populate correctly.

- Use GUI tools (**DNS**, **DHCP**, **ADUC/AD DS**) to administer services from **srv1**.

- Clone **srv3**, join it to the domain, promote it to **DC2 (GC + DNS)**, and verify DNS replication.

### Minimum Progression Requirements

Before beginning, you must have:

1. **Successfully graded *Labs 1-5* and *Assignment 1*.**
2. Attended the **Week 8 - Active Directory** lecture.
3. Your external SSD (or personal computer) with your VMs.
4. Your OSM620 Lab Logbook.
5. Optional, but recommended: Caffeine delivery system.

### Before You Begin

**Time to Backup!** *You will be making major changes to all virtual machines.* This is a good time to backup these VMs so you can restore from them if something catastrophic occurs.

With all VMs turned off, copy the entire `Virtual Machines` folder on your SSD to a separate location. This can be a backup directory on your SSD, or even a separate physical drive. Leave the copy be and use the original going forward. (If you need space, compress the backup folder.)

**Do not continue until the copy has fully completed.**

## Investigation 1: Adding the *router* VM

In this investigation, we'll be adding a Linux-based router to our setup. This low-footprint virtual machine will take the place of the RRAS NAT work you did in previous labs. Most on-premises installations (offices, warehouses, schools) will have a physical router that does this instead of relying on Windows Server to do that work.

Setting this up is extremely easy and should only take a few minutes.

After you're done, you will need to have this VM powered on during all lab work. I'll remind you of this often.

### VM Specifications

| Setting    | Value                       |
| ---------- | --------------------------- |
| VM Name    | router                      |
| Role       | Router / NAT (edge)         |
| OS / Image | VyOS (current stable ISO)   |
| vCPU       | 1                           |
| RAM        | 512 MB                      |
| Disk       | 20 GB                       |
| NICs       | 2                           |
| NIC1       | VMware NAT                  |
| NIC2       | VMnet10                     |
| NIC2 IPv4  | **10.0.`UID`.254/24** (static) |

Instructions to set this up are provided below.

### Part 1: Create *router* VM

Here we'll create the VM hardware and install the VyOS operating system. This is very quick process.

> **Reminder**: Remember to create a new *router* folder on your SSD for the VM files!

1. Download the VyOS ISO: [https://community-downloads.vyos.dev/stream/1.5-stream-2025-Q2/vyos-1.5-stream-2025-Q2-generic-amd64.iso](https://community-downloads.vyos.dev/stream/1.5-stream-2025-Q2/vyos-1.5-stream-2025-Q2-generic-amd64.iso)
2. Create a new VMware VM with the following:
    1. ISO: **VyOS ISO image**
    2. Type: **Debian 12.x 64-bit**
    3. Boot firmware: **UEFI**
    4. Name: **router**
    5. CPU: **1 core**
    6. RAM: **512 MB**
    7. CPU > IOMMU: **Enable**
    8. CPU > Enable hypervisor: **Disabled**
    9. Storage: **20 GB**
    10. NIC1: **NAT**
    11. NIC2: **VMnet10**
3. Power on VM.
4. In the boot menu, select the first option: **Live system (vyos)**
5. Login with these credentials:
    1. Username: **vyos**
    2. Password: **vyos**
6. Install VyOS on the disk using this command: `install image`
    1. Would you like to continue?: **y**
    2. What would you like to name this image?: **Just hit Enter to select the default**
    3. Please enter a password for the "vyos" user: *yourNormalAdminPassword*
    4. What console should be used as default?: **Hit Enter to select default**
    5. Which one should be used for installation?: **Enter to select default**
    6. Installation will delete all data on the drive. Continue?: **y**
    7. Would you like to use all the free space on this drive:? **y**
    8. What would you like to use as boot config?: **Enter to select default**
7. The installation proceeds (it's very fast). When finished, reboot: `sudo reboot`
8. Wait for the machine OS to boot up again.
9. Use the following credentials for your new VM:
    1. Username: ***vyos***
    2. Password: ***yourNormalAdminPassword***

### Part 2: Configuring Routing

Now that our new virtual machine is installed and running, we need to configure it to do our routing. This is a one-time process.

1. Power on router VM and login. (If you haven't already.)
2. Run the following commands ONE AT A TIME (**Replace UID!**):

```bash
configure
set interface ethernet eth0 address dhcp
set system name-server eth0
set interface ethernet eth1 address 10.0.UID.254/24
set nat source rule 20 outbound-interface name eth0
set nat source rule 20 source address 10.0.UID.0/24
set nat source rule 20 translation address 'masquerade'
set system ipv6 disable 
commit
save
exit
```

> *Figure 1. Example of VyOS commands*

3. Check your router can access the Internet: `ping eff.org`

> *Figure 2. Proper ping completion*

4. If it receives proper ping, move on to the next step. If not, ask for help!
5. Log out: `exit`
6. Keep this VM up! Minimize and move on to the next Investigation.

Congratulations! You now have a NAT router VM that uses minimal resources.

> Note: After this lab, whenever you have other Windows VMs turned on, turn this one on first. (You don't need to log into it, just let it do its thing.)

## Investigation 2: Create srv3 VM

In this investigation, we're going to clone *srv2*, reset it to defaults, then configure the clone with *srv3* information.

**Do not create the VM below from scratch!** This is just for quick reference.

| Setting   | Value                  |
| --------- | ---------------------- |
| VM Name   | srv3                   |
| Role      | Windows Server Core #2 |
| vCPU      | 2                      |
| RAM       | 4 GB                   |
| Disk      | 250 GB                 |
| NICs      | 1                      |
| NIC1      | *Removed*              |
| NIC2      | VMnet10                |
| NIC2 IPv4 | **10.0.`UID`.3** (static) |

### Part 1: Cloning in VMware

1. Shut down ALL VMs except the router.
2. Create a new folder inside Virtual Machines on your SSD and call it: *srv3-yourSenecaUsername*
3. In VMware Workstation, right-click on *srv2* and select: **Create Full Clone**
4. Rename when asked to: **srv3-yourSenecaUsername**
5. Save it inside: **Virtual Machines/srv3-yourSenecaUsername/**
6. This may take several minutes to complete.
7. Once complete, change the following hardware (do not power on):
    1. Remove NIC1 (External Network)
    2. Confirm hardware:
        1. CPU: **2 cores**
        2. RAM: **4 GB**

### Part 2: Cleaning the Clone (srv3)

1. Power on and login.
2. Exit to PowerShell.
3. Run the following command: `C:\Windows\System32\Sysprep\Sysprep.exe /oobe /generalize /shutdown`
4. This will take a *very* long time, around 10 minutes. When complete, the computer will shut down.
5. Power on *srv3* again.
6. It will ask you to change the login password. Just use the same you've been using before. (Tab to go to the next field, then Enter to send both fields.)
7. When asked about diagnostics, select option 1 (Required only)
8. Your scrubbed *srv3* will now start up.

### Part 3: Configuring srv3

1. Login to *srv3*.
2. Stay in *sconfig*.
3. Change the computer name to to *srv3*. (***not srv3-yourSenecaUsername!***)
4. The computer restarts.
5. After restart, log back in.
6. Change your network adapter settings to the following:
    1. Name: **Internal Network**
    2. IP address: **10.0.`UID`.3**
    3. Subnet: **255.255.255.0**
    4. Gateway: **10.0.`UID`.254**
    5. DNS1: **10.0.`UID`.2**
    6. DNS2: ***Blank***
    7. DNS3: ***Blank***
7. Exit to PowerShell
8. Check Internet connection by running: `ping 173.239.79.200`

    > **Note:** DNS won't work yet, that's okay.

9. If Step 8 worked, go back to sconfig and shutdown the system.
10. Keep this VM powered off until told otherwise.

## Investigation 3: Convert srv1 to a Management Server

In this investigation, we'll remove all our network services from Labs 1-5 and turn *srv1* into a **Management Server**. Essentially, a GUI that lets us control other servers remotely from the **Server Manager** application.

This is a very normal on-prem setup, and you'll see why later on in the lab. It makes everything far easier and it's more secure.

### Part 1: Removing Old Roles

1. Turn off all VMs.
2. Power on *srv1* and login.
3. In *Server Manager > Remove Roles*:
    1. Roles > Select these roles (Remove features when asked):
          1. DHCP
          2. DNS
          3. Remote Access
          4. Hyper-V
          5. IIS
4. **Do NOT select restart this server.**
5. This may take a while!
6. When complete, shut down *srv1*.

### Part 2: Hardware Changes for srv1

1. Go into srv1's hardware properties in VMware (do not power on!)
2. Remove NIC1 (External Network)
3. Reduce hardware:
    1. CPU: **4 cores**
    2. RAM: **8 GB**

### Part 3: Network Configuration (srv1)

1. Turn on *router* VM.
2. Power on *srv1* and login.
3. Set static network configuration for Internal Network NIC:
    1. IP address: **10.0.`UID`.1**
    2. Subnet: **255.255.255.0**
    3. Gateway: **10.0.`UID`.254**
    4. DNS: **149.112.121.20**
4. Confirm Internet connection.
    1. In Command Prompt, run: `ping eff.org`
    2. Confirm it works. If not, ask for help!
5. Shut down *srv1*.

## Investigation 4: Promote srv2 to AD DC1

In this investigation, we're going to promote *srv2* to an **Active Directory Domain Controller**. As the first Core machine, we have to do this purely from the command line with PowerShell.

### Part 1: Network Reconfiguration (srv2)

Let's configure the server to get it ready for Active Directory.

1. Remove NIC1 (External Network)
2. Confirm hardware:
    1. CPU: **2 cores**
    2. RAM: **4 GB**
3. Power on.
4. Change computer name from *srv2-yourSenecaUsername* to *srv2*. **Restart**.
5. Log back in.
6. Set static network configuration for Internal Network NIC:
    1. IP address: **10.0.`UID`.2**
    2. Subnet: **255.255.255.0**
    3. Gateway: **10.0.`UID`.254**
    4. DNS: **10.0.`UID`.2**
        1. Note: If DHCP won't be removed, go to PowerShell and run: `Set-NetIPInterface -InterfaceAlias "Internal Network" -Dhcp disabled`
        2. Go back to *sconfig* and rerun Step 6.
7. Exit to PowerShell.
8. Check Internet connection by running: `ping 173.239.79.200`
9. If ping is successful, move on to the next Investigation. If not, ask for help!

### Part 2: Installing AD and DC Promotion

Here, we install the Active Directory server role and then promote *srv2* to be an AD Domain Controller.

1. Install the Active Directory role:

```powershell
Install-WindowsFeature AD-Domain-Services
```

2. **This can take a while!** Be patient.
3. Promote srv2 to a domain controller:

```powershell
Install-ADDSForest -DomainName 'yourSenecaUsername.com' -DomainNetbiosName 'YOURSENECAUSERNAME' -InstallDNS -Force
```

4. When asked for a password, use the same one you've used for your Administrator accounts.
5. **This will take several minutes!** Your VM will restart and take a few more minutes. Be patient.
6. Once you can, login with: `YOURSENECAUSERNAME\Administrator`
7. In sconfig, confirm you can see:
    1. Computer name: **srv2**
    2. Domain: **yourSenecaUsername.com**
8. Minimize srv2 VM but keep it on!

## Investigation 5: Manage srv2 from srv1 GUI

Let's use *srv1*'s **Server Manager** to manage and configure *srv2*!

### Part 1: Join srv1 to the new domain

We now have an Active Directory domain, but we need to join *srv1* to it.

1. Double-check the following VMs are already powered on:
    1. router
    2. srv2
2. Power on *srv1* and login.
3. Change "Internal Network" NIC DNS to: `10.0.`UID`.2`
4. In *Computer Properties*, change the computer name from *srv1-yourSenecaUsername* to *srv1*.
5. Restart and log back in.
6. Open *Computer Properties* again, and click on **Workgroup**.
7. Select *Domain* and enter: *yourSenecaUsername.com*
8. When asked for credentials, use your Administrator username and password.
9. When it adds properly, you'll get a welcome message.
10. The computer will now restart.
11. When the computer is up again, login using: **Other > YOURSENECAUSERNAME\Administrator**

### Part 2: Setup srv1 as a GUI Management Server for srv2

This is where we add *srv2* to *Server Manager* for remote control. We also install a few features that help with that remote control.

1. In *Server Manager*, go to **All Servers**.
2. Go to *Manage > Add Servers*
3. Click on **Find Now**.
4. Select *srv2*.
5. Click **OK**.
6. It may take a few moments to appear properly. **Ensure it says "Online - Performance counters not started."**
7. Go to *Manage > Add Roles and Features*
8. Select **srv1** from the list.
9. Skip *Roles*, and select the following *Features*:
10. *Remote Server Administration Tools > Role Administration Tools:*
    1. **AD DS and AD LDS Tools**
    2. **DHCP Server Tools**
    3. **DNS Server Tools**
11. Click **Next** through the rest of the wizard and then **Install**.
12. Any other message, ask for help!

### Part 3: Configure DNS on srv2 with srv1

We're now going to set up Active Directory DNS on *srv2*. This is much more advanced than the previous standalone DNS service we installed in a previous lab, and you'll see that shortly.

1. Go to *Server Manager > Tools > DNS*
2. When asked to connect to a DNS server, select "The following computer" and enter: `srv2`
3. You will now be connected to the DNS server on *srv2* remotely, using the GUI on *srv1*.
4. Find *srv2* in the list, right-click, and select **Properties**.
5. Add forwarders: **149.112.121.20**
6. Right-click on **Reverse Lookup** and select **New Zone**.
7. Stick with the wizard defaults until *Network ID*.
8. Enter Network ID: `10.0.`UID``
9. Click **Next** and then **Finish** through the rest of the wizard.
10. Go to *Reverse Lookup Zones > `UID`.0.10.in-addr.arpa*
11. Notice the list is currently empty.
12. Run `ipconfig /registerdns` on both *srv1* and *srv2*.
13. Refresh the view in reverse lookup and confirm the two server entries.
14. Check the *Forward Lookup Zone > yourSenecaUsername.com*. Your two servers are already there!
15. On *srv1*, open **Command Prompt** and run:

```powershell
ipconfig /flushdns
nslookup srv2
nslookup srv1
nslookup eff.org
ping srv2
ping eff.org
```

16. Open Firefox and go to: **reddit.com**
17. If those two steps worked, you now have full Internet using the DNS service on *srv2*!

Notice how you didn't have to create any A or PTR records! The servers self-registered. *How cool is that?*

### Part 4: Configure DHCP on *srv2* with *srv1*

We're now going to set up Active Directory DHCP on *srv2*. As with DNS, this is much more advanced than the standalone we did earlier.

Unlike DNS, which was installed along with the AD Domain Controller, the DHCP role isn't currently on *srv2*. We have to install it and then configure.

1. Go to *Server Manager > Manage > Add Roles and Features*
2. Select **srv2** from the list.

  > **WARNING:** Make sure you select *srv2*! Not *srv1*!

3. Select the DHCP role.
4. Go through the DHCP wizard and finish.
5. When installation is complete, click on **Complete DHCP configuration**.
6. Click **Next** and then **Commit**.
7. Close the window.
8. Go to *Server Manager > Tools > DHCP*
9. Right-click the DHCP entry in the left and select **Manage authorized servers...**
10. Select *srv2*, then click **OK**.
11. In the new *srv2* entry, create a new scope.
12. Configure it with the following:
    1. Name: **OSM620 Fall 2025 - AD DHCP**
    2. Range: **10.0.`UID`.1 - 10.0.`UID`.254**
    3. Exclude:
        1. **10.0.`UID`.1-10.0.`UID`.9**
        2. **10.0.`UID`.254**
    4. Router: **10.0.`UID`.254**
    5. DNS:
        1. DNS1: **10.0.`UID`.2**
        2. DNS2: **10.0.`UID`.3**
    6. WINS: **Leave defaults**
    7. Select **Activate now**.
13. Back in the main DHCP window, go to the *IPv4* folder, right-click, and select **Properties**.
14. Click the DNS tab.
15. Select **Always dynamically update DNS records**
16. Click **OK**
17. That's it, you're done!

## Investigation 6: Add laptop1 to Domain

Now that we've set up our AD Domain Controller on *srv2* and remote control on *srv1*, it's time to add a normal client machine to our Active Directory domain. We'll use *laptop1* for this.

The initial "join to AD" is typically done by an administrator such as yourself. After the laptop is part of the AD domain, you would then give it to the employee.

### Part 1: Setting Proper Network Configuration

1. Have the following VMs powered on:
    1. router
    2. srv2
    3. srv1 (if you have the hardware resources)
2. Before powering on *laptop1*, check NIC1 in VMware. Make sure it's set to **VMnet10**.
3. Power on *laptop1* and login.
4. If you named this machine *laptop1-yourSenecaUsername*, change it to *laptop1* and restart.
5. Check Windows NIC1 network settings. They should be set to **DHCP**.
    1. If it was static until now, open **Command Prompt** and run to check it has an Internet connection:
        1. `ipconfig /release`
        2. `ipconfig /renew`
6. In Command Prompt, run: `ipconfig /all`
7. Check the information. You should have a 10.0.`UID`.x address, DNS as *srv2*, and gateway as 10.0.`UID`.254.
8. In Command Prompt, run:

```powershell
ipconfig /flushdns
nslookup srv1
nslookup srv2
nslookup eff.org
ping srv1
ping srv2
ping eff.org
```

9. Back on *srv1*, go to *Server Manager > Tools > DHCP*
10. In the DHCP window, go to: *srv2 > IPv4 > Scope > Leases*
11. You should see your *laptop1* in the leases section.
12. If all this works, continue to the next section!

*Laptop1* now has a stable Internet connection using DHCP and DNS from *srv2*, and is routing to the Internet through the *router* VM.

### Part 2: Domain Join for laptop1

Now, it's time to join *laptop1* to our AD domain.

1. Open *Settings > System > About* (at the very bottom)
2. Click on the **Domain or workgroup** link.
3. In the new *System Properties > Computer Name* tab, click on **Change**.
4. Change from workgroup to domain: **yourSenecaUsername.com**
5. Authenticate the same as you did with *srv1*.
6. After the congratulations message, allow the machine to restart.
7. Do not log in with your normal account.
8. Click on Other, and use: **YOURSENECAUSERNAME\Administrator**
9. Your profile may take a minute to setup. That's normal.
10. Once you see the desktop, minimize this VM.

### Part 3: Confirm laptop1 is in DNS

Let's check that *laptop1* auto-registered to DNS after the AD join.

1. On *srv1*, open the DNS tool.
2. Go to *srv2 > Forward Lookup Zones* > *yourSenecaUsername.com*
3. Do you see laptop1? If yes, congrats! It auto-registered, just as we wanted.
4. If not, ask for help. Do not continue.

## Investigation 7: Promote *srv3* to AD DC2

In this investigation, we'll create a second AD Domain Controller by using *srv3*. This is a much simpler process than *srv2*, as we can use *srv1* for part of it.

### Part 1: Domain Join for *srv3*

Time to add *srv3* to our new AD domain.

1. Power on *srv2* and *router* (if not already on). Leave all other VMs off.
2. Power on *srv3* and login.
3. Change workgroup to domain: **yourSenecaUsername.com**
4. Authenticate with your Administrator credentials when asked.
5. The computer will restart. This will take some time.
6. Once you can, login with `YOURSENECAUSERNAME\Administrator` by hitting the ESC key on your keyboard. (Twice.)
7. When it says *Select a user*, select **Other user**.
8. In the next page, use the following credentials to login (tab on your keyboard to go to the next field):
    1. User name: **YOURSENECAUSERNAME\Administrator**
    2. Password: ***yournormalpassword***
9. If it asks you about diagnostics again, select **1**.
10. In sconfig, confirm you can see:
    1. Computer name: **srv3**
    2. Domain: **yourSenecaUsername.com**
11. Minimize *srv3* VM but keep it on!

### Part 2: Install AD and DC Promotion (srv3)

Now, let's use *srv1*'s **Server Manager** to install the Active Directory role on *srv3* and promote *srv3* to a secondary Domain Controller. All remotely through the GUI.

1. Power on srv1 and login using: **YOURSENECAUSERNAME\Administrator**

> **NOTE:** You **must** login as the domain administrator. Otherwise, none of this will work!

2. In Server Manager, go to All Servers.
3. Click on *Manage > Add Servers*
4. Add *srv3*.
5. This will take a few minutes to process. **Wait until the SRV3 line says: "Online - Performance counters not started."**
6. In Server Manager, add a new role. (*Manage > Add Roles and Features*)
    1. Select: **srv3**
    2. Select: **Active Directory Domain Services**
    3. Go through wizard and select defaults until you get to the *Confirmation* page.
    4. On the *Confirmation* page, select the option **"Restart the destination server automatically if required"**, then click **Install**
    5. Active Directory will now install onto *srv3*.
    6. When complete, the **Promote this server to a domain controller** link is active on the *Results* page. **Click it**.  
7. The *Active Directory Domain Services Configuration Wizard* now opens.  

    1. Keep the defaults on the first page, but click on the **Change...** button. We need to authorize this connection.
    2. The credentials window opens. Enter the following:
        1. User name: **Administrator**
        2. Password: ***yournormalpassword***
    3. Back in the main wizard, click **Next**,
    4. On the next page, its asking for a *DSRM password*. Enter the same password you've been using for all your admin accounts.
    5. Keep all other defaults and click **Next**.
    6. The DNS page gives you a warning. Don't worry about it and click **Next**.
    7. On the *Additional options* page, for the *Replicate from:* field, select *srv2* from the drop down. Click **Next**.
    8. Click **Next** on the rest of the pages to accept defaults until you arrive on *Prerequisites Check* page.
    9. Look for the green check mark that says: ***"All prerequisites checks passed successfully. Click Install to begin installation."***

        > **NOTE: If you do NOT see the green check mark, do not continue! Ask for help!**

    10. Click **Install**.
    11. It will now install. The DNS warning will come up again. Continue to ignore it.
    12. Once it finishes, click **Close**.
8. Give it a few minutes as it processes.
9. When the *srv3* VM is back to the login screen, go to *srv1*. (This will take a few minutes.)
10. Open **Server Manager** on *srv1*.
11. Go to **AD DS** and confirm *srv3* is there.
12. It may take a few moments to appear properly. **Ensure it says "Online - Performance counters not started."**
13. Any other message, ask for help!

You have now converted *srv3* into our second AD Domain Controller, mostly using the GUI remotely!

### Part 3: Confirm DNS replication

Always confirm your work. Let's check on DNS.

1. In *srv1*, open the DNS tool.
2. When asked, tell it to connect to **srv3**.
3. In the new window, right-click on DNS and select **Connect to DNS Server**.
4. Add *srv2* to your list of servers in DNS.
5. Open *IPv4 > Forward Lookup Zones* in both *srv2* and *srv3*.
6. They should look identical.
7. If they do, then DNS replication is working.
8. Find *srv3* in the list, right-click, and select **Properties**.
9. Add forwarders: **149.112.121.20**
10. Congratulations, you now have two Domain Controllers!

This confirms *srv2* and *srv3* are communicating and syncing information, including DNS records.

## Investigation 8: Test Without srv1

In this investigation, we'll explore the consequences of what we've set up in this lab and how much more efficient on hardware resources it is.

1. Shut down *srv1* and *srv3*.
2. Have the following VMs powered on:
    1. router
    2. srv2
3. Power on *laptop1*.
4. Login to *laptop1* with: **YOURSENECAUSERNAME\Administrator**
5. Use Command Prompt and run: `ipconfig /all`
6. Does it look correct? If so, move to the next step.
7. Open *Firefox* and go to: **reddit.com**
8. Did that work? If so, move to the next step. **Otherwise, ask for help.**
9. In Command Prompt, run:

```powershell
ipconfig /flushdns
nslookup srv1
nslookup srv2
nslookup srv3
nslookup eff.org
ping srv1
ping srv2
ping srv3
ping eff.org
```

10. Why did some of these work and not the others?

Congrats! You now have a low profile Domain Controller. Resource usage is at a minimum.

> **A note about Hyper-V:** In a real production environment, we could use *srv1* to install Hyper-V VMs on *srv2*, allowing us to run these VMs on a lower resource Core machine. That's beyond the scope of this course, but something cool to know.

## Lab 6 Sign-Off

It’s essential to complete Lab 6 correctly. All later labs assume working Active Directory Domain Controllers and its services.

When you finish Lab 6, ask your instructor for a sign-off.

### Sign-Off Checklist

Please have the following on screen and ready to show. You will need to power on ALL VMs for the check.

#### On srv1

1. *Server Manager > All Servers*: Shows *srv1*, *srv2*, and *srv3*
2. *Server Manager > AD DS*: Shows *srv2* and *srv3*
3. `ipconfig /all` shows the correct information.
4. Firefox loads: **eff.org**

#### On srv2

1. Sconfig shows:
    1. Name: **srv2**
    2. Domain: **yourSenecaUsername.com**
2. In PowerShell, `ipconfig /all` shows the correct information.
3. You can ping:
    1. srv1
    2. srv3
    3. laptop1
    4. eff.org
    5. reddit.com

#### On srv3

1. Sconfig shows:
    1. Name: **srv3**
    2. Domain: **yourSenecaUsername.com**
2. In PowerShell, `ipconfig /all` shows the correct information.
3. You can ping:
    1. srv1
    2. srv3
    3. laptop1
    4. eff.org
    5. reddit.com

#### On laptop1

1. System Properties shows:
    1. Name: **laptop1**
    2. Domain: **yourSenecaUsername.com**
2. In Command Prompt, `ipconfig /all` shows the correct information.
3. You can ping:
    1. srv1
    2. srv2
    3. srv3
    4. eff.org
    5. reddit.com
4. In Firefox: `http://www.reddit.com` loads the Reddit main page.

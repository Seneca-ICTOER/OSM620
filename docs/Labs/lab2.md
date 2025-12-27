---
id: lab2
title: Lab 2 - Hyper-V, NAT, and Windows Clients
sidebar_position: 2
description: Lab 2 - Hyper-V, NAT, and Windows Clients
--- 
# Lab 2 - Hyper-V, NAT, and Windows Clients

## Lab Preparations

### Purpose of Lab 2

In this lab, you’ll move beyond basic VM deployment and learn to create nested virtualization environments. This is a crucial skill for modern sysadmins and cloud architects. You’ll enable the Hyper-V role on your Windows Server, build a secure internal network using Hyper-V switches, and deploy Windows 11 client VMs within your existing server VM. You’ll also set up NAT and routing, allowing your nested clients to access the internet safely through your server.

By the end, you’ll have a working mini-enterprise: one physical computer running a server VM, which itself is hosting client VMs. Just like real-world cloud and enterprise setups! You’ll gain hands-on experience with Hyper-V, RRAS, and NAT, and build a technical foundation for future labs in DHCP, DNS, and security.

### Objectives

By the end of this lab, you will be able to:
  * Enable the Hyper-V role on a Windows Server 2025 VM and configure virtual networking.
  * Build and configure a Hyper-V External Virtual Switch bound to a specific NIC.
  * Set up Routing and Remote Access (RRAS) for network address translation (NAT) between internal and external networks.
  * Deploy Windows 11 client VMs inside Hyper-V, configuring hardware and storage as needed.
  * Assign static IPs and ensure that internal client VMs can reach the internet via NAT on your server.
  * Practice real-world sysadmin tasks: managing nested VMs, routing, and internal networks.

### Minimum Requirements

Before beginning, you must have:
  1.	Completed Lab 1 in full, with both srv1 and srv2 working, activated, and updated.
  2.	Attended the Week 2 lecture, or carefully read through the Week 2 slides.
  3.	Downloaded the Windows 11 Education ISO and your product key (from Lab 1).
  4.	Access to VMware Workstation and your external SSD with at least 200 GB of free space remaining.
  5.	Your printed OSM620 Lab Logbook for notetaking and command recording.
  6.	A basic understanding of server roles, virtual networking, and Windows installation processes.
  7.	Caffeine or snack of your choice: nested virtualization can be a long process!

## Investigation 1: Hyper-V Role Installation and Switch Setup *(srv1)*

In this investigation, you will install the Hyper-V role on srv1 and create a Hyper-V virtual switch that uses your existing Internal Network adapter.

Your Hyper-V switch will be named Internal Network (Hyper-V).

### Part 1: Install the Hyper-V Role

 1. Open the **Server Manager** application and go to **Add Roles and Features**.
 1. **Installation Type:** Keep Role-based or feature-based installation selected.
 1. **Server Selection:** Ensure your local server (*srv1*) is selected.
 1. **Server Roles:** Check ***Hyper-V***.
    1. When prompted to "*Add features that are required for Hyper-V*", click **Add Features**.
 1. **Features:** Leave defaults (the *Hyper-V Management Tools* are already included).
 1. **Hyper-V page**:
    1. When asked about Virtual Switches, select **Internal Network** only.
    1. Migration for live migrations: leave unchecked.
    1. Default Stores: leave defaults.
 1. Confirmation: Check "*Restart the destination server automatically if required*".
 1. Click **Install**.
 1. Wait for installation to complete. If prompted, allow the reboot. Log back in when finished.

> **Note:** It’s normal for *Server Manager* to take a minute or two after reboot to show the Hyper-V tile.

### Part 2: Rename the Hyper-V Virtual Switch

As part of our Hyper-V role installation, we selected the *Internal Network* (Part 1, Step 6.1).

This has created a virtual switch on the server. For easy management, we'll rename it.

Our new virtual switch name will be: `HQ Network`

 1. In Server Manager, click Tools (top-right) and select Hyper-V Manager.
 1. In the left pane, click your server name (e.g., srv1-…).
 1. In the right Actions pane, click Virtual Switch Manager….
 1. Under the Virtual Switches column on the left, look for the switch that's already been created. It will likely have a name similar to: `Intel(R) 82574L Gigabit Network Connection - Virtual Switch`
 1. In the Virtual Switch Properties window:
    1. Name: Change to `HQ Network`
    1. Ensure the **External network** radio button is selected.

        > **Reminder:** This is the adapter you renamed earlier to Internal Network and set to 10.0.`UID`.1.

    1. Ensure "*Allow management operating system to share this network adapter*" is checked.
    1. Click OK. If you see a warning about temporary network disruption, click Yes.

    > **Naming note:** You are creating an External Hyper-V switch, but you’re binding it to the Windows NIC you named Internal Network.
    >
    > That’s correct.
    >
    > You will end up with a third network adapter called: `vEthernet (HQ Network)`

 1. Go back to *Server Manager > Local Server* and refresh. You should now see the following two networks:
    * **External Network**: Assigned by DHCP
    * **vEthernet (HQ Network)**: IPv4 addess assigned by DHCP, IPv6 enabled

### Part 3: Configure Network Setting for Hyper-V Switch

Now that we've created the Hyper-V network adapter, we have to give it an IP address. This will let our Hyper-V VMs talk to the server, and eventually, bridge our Internet connection.

 1. In *Server Manager > Local Server*, click on the entry next to **vEthernet (HQ Network)**. This opens the *Network Connections* window.
 1. Verify you now see a third network adapter: `vEthernet (HQ Network)`
 1. Right-click it → Properties and do the following:
    1. Internet Protocol Version 6 (TCP/IPv6): **Unchecked**
    1. Internet Protocol Version 4 (TCP/IPv4): Select and click **Properties**
        1. IPv4 Address: **10.0.`UID`.1**
        1. Subnet Mask: **255.255.255.0**
        1. Leave all other fields blank and click OK.
 4. Back in *Server Manager > Local Server*, refresh. You should now see the following two networks:
     * **External Network**: Assigned by DHCP
     * **vEthernet (HQ Network)**: 10.0.`UID`.1

## Investigation 2: RRAS and NAT

Our server now has access to two networks:

* **External Network**: Connection to the Internet
* **vEthernet (HQ Network)**: Internal network only between *srv1* and the Hyper-V VMs we'll create in Investigation 3. This network *does not* have access to the Internet.

If we want to give our Hyper-V machines Internet access, we need to route network traffic between the two networks using *srv1* as a bridge. This keeps our Hyper-V machines on a secure network, and Internet access is handled and monitored by our Windows Server.

This will involve setting up routing and network address translation (NAT).

Welcome to Microsoft's **Routing and Remote Server (RRAS)**.

### Part 1: Installing the Remote Access Role

 1. Open the **Server Manager** application and go to **Add Roles and Features**.
 1. **Installation Type:** Keep Role-based or feature-based installation selected.
 1. **Server Selection:** Ensure your local server (*srv1*) is selected.
 1. **Server Roles:** Check ***Remote Access***.
 1. **Features:** Leave defaults (the *Hyper-V Management Tools* are already included).
 1. **Remote Access page:** Click Next.
     1. The **Add Roles and Features Wizard** dialog box appears. Leave defaults and select *Add Features*.
 1. **Remote Access > Role Services page:** Check the following and click *Next* when ready:
     * Direct Access and VPN (RAS)
     * Routing
 1. Keep defaults and select *Next* on the following pages:
     1. Web Server Role (IIS)
     1. Web Server Role (IIS) > Role Services
 1. Confirmation: Click **Install**.
 1. In the **Results** page: When installation has completed, click on the "*Open the Getting Started Wizard*" link.
 1. The wizard opens. Make no changes and close it. (This just removes the system nagging.)
 1. Back in **Results**, click Close.

### Part 2: Enabling NAT with RRAS

You will now install the tools for routing and NAT bridging discussed at the top of this investigation using the **Routing and Remote Access Server** (RRAS) we've just installed.

1. Open the **Server Manager** application and go to **Tools > Routing and Remote Access**.
1. This opens the *Routing and Remote Access* application.
1. In the right-hand column, find SRV1, right-click it, and select **Configure and Enable Routing and Remote Access**.
1. The setup wizard appears. Click *Next*.
1. **Configuration:** Select the *Custom configuration* radio button.
1. **Custom Configuration:** Select (checkbox) only the following:
    * NAT
    * LAN routing
1. **Summary window**: Click *Finish*.
1. The **Routing and Remote Access** service status dialog box appears. Click *Start service*.
1. It may take a few moments for the service to fully start. Be patient.
1. Once the *Routing and Remote Access* application reappears, move to the next part.

### Part 3: Configuring NAT with RRAS

It's now time to configure the routing and NAT bridging between the two networks.

1. Open the **Routing and Remote Access** application.
1. In the right-hand column, expand *SRV1*, then expand *IPv4*.
1. Right-click on *NAT* and select *New Interface*.
1. Select the **External Network** entry and click *OK*.
1. In the *Network Address Translation Properties* window, select the following and click OK when ready:
    * Public interface connected to the Internet
    * Enable NAT on this interface
1. Right-click on *NAT* and select *New Interface* again.
1. Select the **vEthernet (HQ Network)** entry and click OK.
1. In the *Network Address Translation Propertie*s window, select the following and click OK when ready:
    * Private interface connected to private network
1. Back in the *NAT* window, you should see two created interfaces:
    * External Network
    * vEthernet (HQ Network)
1. Keep the application open on *NAT* and minimize. We'll come back to it later.
1. That's it! Move on to the next Investigation.

## Investigation 3: Installing Windows Client 1 with Hyper-V *(client1)*

In this investigation, you'll create your first Hyper-V based virtual machines.

These will be Windows 11 client machines. Think of these as typical workstations in a corporate office.

### Part 1: Creating the Hyper-V Virtual Machine

* Hypervisor: **Hyper-V**
* Name: **client1-`SenecaUsername`**
* RAM: **4 GB**
* CPU: **2 processors**
* Storage: **64 GB**
* NIC: **1** (default)
* Security: **Enable Trusted Platform Module**
* ISO: **Windows 11 Education**

1. Drag and drop the *Microsoft Windows 11 ISO* file from Lab 1 onto the desktop of *srv1*. It should copy the file there. If it doesn't, ask your professor for help before continuing.
1. Open the **Server Manager** application and go to **Tools > Hyper-V Manager**.
1. This opens the *Hyper-V Manager* application.
1. In the right-hand column, click on **New > Virtual Machine...**
1. The *New Virtual Machine Wizard* appears.
1. On the *Before You Begin* page, check the **Do not show this page again** option and click **Next**.
1. **Specify Name and Location**:
    * Name: **client1-`SenecaUsername`**
    * Leave the rest at their defaults.
1. **Specify Generation:** Leave defaults.
1. **Assign Memory:** 4096 MB
1. **Configure Networking:**
    * Connection: **HQ Network**
1. **Connect Virtual Hard Disk:**
    * **Create a virtual hard disk** selected.
    * Size: **64 GB**
    * Leave the rest at their defaults.
1. **Installer Options:**
    * Select **Install an operating system from a bootable CD/DVD-ROM**
    * Select **Image file (.iso):**
    * Click **Browse** and select the Windows 11 ISO image file you copied to *srv1*.
1. **Summary:** Click *Finish*.

> **WARNING:** Do not start the *client1* VM, yet! We need to modify it. See *Part 2*.

### Part 2: Modifying *client1* VM for Windows 11

Despite Hyper-V being a Microsoft product and Windows 11 being a Microsoft product, Hyper-V doesn't automatically add everything needed to run Windows 11.

If you think this is silly... It is. Welcome to the wonderful world of IT.

1. In the **Hyper-V Manager** application, select the *client1* VM.
1. In the right-hand column, look for and click on the **Settings...* option.
1. This opens up the settings window for the *client1* VM. Change the following:
1. **Processor:**
    * Number of virtual processors: **2**
    * Make sure to click **Apply**!
1. **Security:**
    * Encryption Support > Enable Trusted Platform Module: **Checked**
    * Make sure to click **Apply**!
1. Click **OK** to close the VM settings window.

### Part 3: Hyper-V Client Management

Knowing how to power on and off your Hyper-V VM is important and includes a few cool features.

In the **Hyper-V Manager** application, select the **client1** VM. (Single click. Do not double-click.)

On the right-hand column, towards the bottom, you have:

* **Connect**: This *connects* to the VM. It will open a virtual monitor to the machine. Just like a normal computer, if the computer (VM) is off, connecting to it will do nothing but give you a blank screen.
* **Settings**: You used this in *Part 2*. Modify VM settings. Don't change anything unless told to.
* **Start**: Power on the VM. If you are already connected to it, you will see it power up. If not connected, it still powers up. Click on **Connect** to open a screen to it.

When the VM is powered on, you have additional options in that column:

* **Turn Off:** This is the equivalent of pulling the power plug. Don't do this unless the VM is fully frozen and not responding from commands. (This is the *"Nuke it from orbit."* option. It works, but it's overkill and may cause damage.)

* **Shut down:** This is the polite version of the option above. Hyper-V will send a gentle "Please shut down." command to the Windows 11 client. If the Windows 11 client is working, it'll close all applications and shut down properly. Hyper-V detects the OS having shut down and then powers off the VM itself. (**This is your preferred option.**)

* **Reset:** Equivalent to pressing the *Reset* physical button on a computer. Yanks the power and then turns it back on. AKA *"Nuke and reload."* (Again, don't use unless unresponsive.)

* The **Start** button disappears when the VM is on, which makes sense.

There are other options, but those are enough to get by for now.

> **IMPORTANT:** Always safely shut down all Hyper-V VMs before shutting down your Windows Server!

### Part 4: Installing Windows 11 (*client1*)

1. Once the Windows 11 Setup screen appears:
1. On *Select language settings*, keep the defaults and click **Next**.
1. On *Select keyboard settings*, keep the defaults and click **Next**.
1. On *Select setup option*, do the following:
    1. Select *Install Windows 11*
    1. Check the box next to *I agree everything will be deleted, including files, apps, and settings*
    1. Click **Next**.
1. On the *Product key* page, enter your product key and click **Next**.
1. On *Applicable notices and license terms*, click **Accept**.
1. On *Select location to install Windows 11*, keep the defaults and click **Next**.
1. On *Ready to install*, click **Install**.
1. The Windows Installer will now install the OS. This may take some time, and the percentage may freeze at certain points. Be patient.
1. When the installer finishes, Windows 11 will start up and you will be launched into the First-Run Setup. Go to Part 3.

### Part 5:  First-Run Setup

1. On *Is this the right country or region?*, select **Canada** and click **Yes.**
1. On *Is this the right keyboard layout or input method?*, stick with the default and click **Yes**.
1. On *Want to add a secondary keyboard layout?*, click **Skip**.

  > **Note:** If it has an Internet connection, Windows will now check for available updates from the Internet. If it finds any, it will install them automatically.
  >
  > If it doesn't have an Internet connection, this step will be skipped.

This process may take some time. Please be patient. Your VM may restart on its own.

### Part 6: Account Creation

Now that language/keyboard and installer updates have been applied, it's time to create your account.

> **Note:** Steps 1-2 may not be necessary. If you're asked for a name, start with Step 3.

1. On *Let's set things up for work or school*, select **Sign-in options**.
1. On this next screen, click on **Domain join instead**.
1. On *Who's going to use this device?*, enter your **Seneca username**, (*Not* your full name), then click **Next**.
1. On *Create a super memorable password*, enter the same password you've used for your other VMs and click **Next**.
1. Confirm it on the next screen and continue.
1. On *Now add security questions*, fill out three security questions, clicking **Next** after each.
1. On *Let Microsoft and apps use your location*, select **No**, then click **Accept**.
1. On *Find my device*, select **No**, then click **Accept**.
1. On *Send diagnostic data to Microsoft*, scroll down to select **Required only**, then click **Accept**.
1. On *Improve inking & typing*, select **No**, then click **Accept**.
1. On *Get tailored experiences with diagnostic data*, select **No**, then click **Accept**.

> **Note:** If it has a valid Internet connection, Windows will now check for *more* available updates. As before, if it finds any, it will install them automatically.
>
> If not, it will simply continue.

This process may take some time. Please be patient. Your VM may restart on its own.

Once complete, you will be presented with a login screen. Move to the next part to continue.

## Investigation 4: Post-Installation Tasks *(client1*)

In this investigation, we'll log in for the first time and run through several post-installation tasks both necessary and for user comfort.

### Part 1: First Login

1. Enter your password to login. First-login may take a few minutes as your profile is set up.

### Part 2: Network Configuration

1. Click on the *Start* button and search for **Network Connections**. Open it when found.
1. Find the only Ethernet adapter (it may have different names).
1. Right-click it → Properties and do the following:
    1. Internet Protocol Version 6 (TCP/IPv6): **Unchecked**
    1. Internet Protocol Version 4 (TCP/IPv4): Select and click **Properties**
        1. IPv4 address: **10.0.`UID`.11**
        1. Subnet mask: **255.255.255.0**
        1. Default gateway: 10.0.`UID`.1
        1. Preferred DNS server: 10.0.`UID`.1
        1. Alternate DNS server: 8.8.8.8
        1. Leave all other fields blank and click OK.
1. Open *Microsoft Edge* and nagivate to **eff.org**. If the page loads, you're done!

> **Check it out:** You are now using RRAS+NAT on your Windows Server to make this Internet connection happen.
>
> In *srv1*, go back to the **Routing and Remote Access** application. Inside, go to *SRV1 > IPv4 > NAT*. The **External Network** entry now has non-zero numbers under the 'packets translated' columns. This is your routing and NAT from *client1* at work! Refresh the page to see the count go up.

### Part 2: Setting the Time Zone

1. Login again.
1. Find the time on the bottom right of the screen (in the VM, not your host machine!)
1. Right-click on the time and select **Adjust date and time**.
1. Check the displayed time and time zone.
1. The time should match current time, and the time zone should say *"(UTC-5:00) Eastern Time (US & Canada)"*.
1. If the time zone is wrong, change it to the value above.
1. Confirm your time zone and time matches local time.

### Part 3: Setting Internal Computer Name

When we created our VM, we gave it the name *client1-SenecaUsername*. This only applies to how VMware Workstation sees the VM, not to how the internal Windows OS sees itself. We need to change that to match.

1. In the *Settings* window you still have open (or reopen it from the Search bar), look to the left-hand menu bar and select **System**.
1. The very first thing you see at the top is the current computer name. This is what we're going to change.
1. Click on the **Rename** text link.
1. In the *Name your device* field, enter your VM's name: **client1-SenecaUsername** (replacing *SenecaUsername* with your actual username)
1. Click **Next**.
1. You will now be asked if you'd like to restart. Click **Restart now**.
1. After restarting and logging back in, go into *Settings* > *System* and confirm your computer name is now **client1-SenecaUsername**.

### Part 4: Windows Updates

A critical part of a security-conscious mindset is running regular updates. **This is NOT something you do only once at the start of installation.** You should be running these regularly to keep up to date with security fixes and zero-day exploits.

1. If you're already in the *Settings* application, look through the left menubar for **Windows Update** and click it.
1. In the *Windows Update* main screen, scroll down to **Advanced Options** and click it.
1. The very first option is *Receive updates to other Microsoft products*. Toggle this from **Off** to **On**.
1. At the top of the screen, where it says *Windows Update > Advanced Options*, click **Windows Update** to go back to the previous screen.
1. You will likely already see updates ready. Click on **Download & install all**.
1. As you might expect, this can take a while. Timing will depend on your Internet connection, how fast your computer is, how fast your SSD is, and how many updates there are. Please be patient. Your computer may restart.
1. Once updates have begun, take a break while it does its thing. Grab a drink, make a sandwich, text a friend.
1. After updates are complete, go back into *Windows Update* and click **Check for updates** again. There may be (and often times are) more.
1. If there are more updates, complete Steps 5-8 again until there are no more updates available.
1. In *Windows Update*, scroll back down to **Advanced options** again and click it.
1. Inside *Windows Update > Advanced options* scroll down to **Optional updates** and click it.
1. Select all available updates that appear (you may have to expand some lists).
1. Click **Download & install**.

## Investigation 5: Installing Windows Client 2 with Hyper-V *(client2)*

In this investigation, you'll create your second Hyper-V based virtual machine. This will be another Windows 11 client.

To conserve resources and speed up installation, it's helpful to shutdown *client1* while doing this.

### Part 1: Creating the Hyper-V Virtual Machine

Create a second Hyper-V virtual machine with the following settings:

* Hypervisor: **Hyper-V**
* Name: **client2-`SenecaUsername`**
* RAM: **4 GB**
* CPU: **2 processors**
* Storage: **64 GB**
* NIC: **1** (default)
* Security: **Enable Trusted Platform Module**
* ISO: **Windows 11 Education**

### Part 2: Installing Windows 11 (*client2*)

Set this up exactly as you did with *client1*, with **two important changes**:

1. Computer Name: **client2-`SenecaUsername`**
2. IP address: **10.0.`UID`.12**

### Part 3: Post-Installation Tasks (*client2*)

Don't forget to run your **Post-Installation Tasks**! These are the same as for *client1*

Once *client2* is installed and has a working Internet connection, you're done!

## Lab 2 Sign-Off

It’s essential to complete Lab 2 correctly, as the nested virtualization setup will be the core environment for all advanced labs. Everything you do from this point (DHCP, DNS, security, and domain controller work) will depend on this structure being solid and consistent.

When you finish Lab 2, ask your instructor for a sign-off. Your instructor will check your configuration, verify networking, and confirm all VMs are working as required.

Sign-Off Checklist
  1.	Hyper-V role is installed and running on srv1.
  2.	HQ Network (External Virtual Switch) exists and is bound to the correct NIC.
  3.	RRAS (Routing and Remote Access) is installed and NAT is configured for the two networks:
        1. External Network: **Internet access (DHCP-assigned)**
        1. vEthernet (HQ Network): **10.0.`UID`.1/24**
  4.	client1 and client2 are both running inside Hyper-V, with the following:
        1.	Static IPs set: **10.0.UID.11** and **10.0.UID.12**, respectively.
        1.	Default gateway set to **10.0.UID.1**
        1.	Both can access the internet (test by browsing to eff.org).
        1.	Both have correct computer names set.
        1.	Both have time zone set to EST.
        1.	Both are fully updated via Windows Update.
	5. Graceful shutdown: You can demonstrate safe start, shutdown, and reset procedures for Hyper-V VMs.
	6. All relevant steps, commands, and decisions are recorded in your Lab Logbook.

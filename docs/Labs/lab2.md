---
id: lab2
title: Lab 2 - Security and Remote Connectivity
sidebar_position: 2
description: Lab 2 - Security and Remote Connectivity
---

# Lab 2 - Security and Remote Connectivity

## Lab Preparation

### Purpose / Objectives of Lab 2

In this lab, you will conduct several Windows system administration tasks to secure your servers against would-be attackers and gain preliminary experience with the command line interface.

If you encounter technical issues, please contact your professor via e-mail or in your section's Microsoft Teams group.

### Minimum Requirements

Before beginning, you must have:

1. Successfully completed Lab 1.
2. Attended the Week 3 lecture
3. Read through the Week 3 slides, and have them handy as a reference for concepts.
4. Your external SSD (or personal computer) with your VMs from Lab 1.
5. Your VM login credentials.
6. Optional, but recommended: Caffeine delivery system.

## Key Concepts

### Security: From the Beginning

In the not-too-distant past, companies would focus on getting their product and systems working and relegating security as their last step, often as an afterthought. When security is only considered at the end of a project, it's very difficult to remember all the ways in which your product interacts and things can get missed.

This created several high-profile breaches in the 90s and early 2000s, and our approach to security had to be reconsidered.

As a result, we now consider security **from the beginning**. As you create applications, add users to databases, create links between services, you _must_ keep security in mind at every step of development. Securing as you go is the best method, but even something as simple as simply documenting unsecured parts of your code as you go can be enough (assuming you go back and fix them!)

Generally, we apply the concept of [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) to security. Essentially, this boils down to locking everything down as much as possible and only allowing what and who you need through. Open access makes you a target. You'll be applying this principle to the firewall later in this lab.

We also take a look at defaults. Most systems and software come with pre-configured defaults to make out-of-the-box setup easy. This can take the form of a default username and password, default ports, etc. In a well secured system, these are often changed to avoid hack attempts. If you know the default, there's a high chance that hackers know it as well. You'll be changing some defaults in this lab.

This is not an exhaustive list of applied security, but it does give us a bit of working knowledge. You'll need it for this lab as well as in our later work.

### Firewalls

In short, **a firewall is a utility that sits on your computer between your network connection and the rest of your system.**

Any application, service, or other data that is sent or received by your computer goes through your firewall first. The dominant network protocol is TCP/IP, which means we’re dealing with _packets_.

**A firewall looks at these packets.**

To be clear, the firewall doesn’t look _inside_ packets, but just at the outside data like IP address and/or port destination, etc. The actual transmitted data is still secure and unread.

Generally with firewalls, we apply the _Principle of Least Privilege_ by dropping all new connections by default, and allowing a few exceptions. This is known as **whitelisting**.

### Editing Text Files

As you will sometimes be working in the Windows PowerShell command line environment, it is useful to learn a least one common method of editing text files.

Although programmers and developers usually use graphical IDE's to code and compile programs (Visual Studio, Sublime, Eclipse, etc), they can create source code using a text editor and compile their code directly on the server to generate executable programs (without having to transfer them for compilation or execution).

Developers very often use a text editor to modify configuration files. In this course, you will become familiar with the process of installing, configuring, and running network services. Text editors are an important tool to help this setup, but are also used to "tweak" or make periodic changes in service configurations.

The most readily-available command line text editor built into Windows is **Notepad**.

However, Notepad is not available in Server Core. To edit a text file in that environment, we have three main options:

1. Use PowerShell's object-oriented programming to send an edit directly into a text file. This is cumbersome and is not interactive (you don't see the text file on screen), but it is the only built-in option.
2. Transferring the file to a different computer that does have a text editor (like your Windows 11 client), modifying the file there, and transferring it back to your Server Core machine.
3. Using **Visual Studio Code** to connect remotely the environment. This is by far the coolest and most convenient option.

We'll be doing all three in this lab to show you how, but going forward, VS Code will be our go-to for interacting with Server Core in most instances. (You will be tested on all three options, so don't skip them!)

## Investigation 1: Windows Defender (Firewall)

In this investigation, we're going to take a look at the Windows Defender firewall to see how firewall rules can be applied to secure our servers and let the few things we want to allow through.

It should be noted that the Windows Defender firewall _should not be your only defense_. Production environments will often have managed networks that restrict access on a larger scale, some of which you will get into with your networking courses.

These work together, though we will only be focusing (mostly) on the Windows firewall in this course.

### Part 1: Windows Server GUI (srv1) - Testing VM Network Connectivity

As part of your Lab 1 environment setup, you tested your connection to the Internet from each of your 3 virtual machines.

It's now time to test if we can use each VM to connect to _each other_.

1. On your Windows Server GUI (_srv1_) VM, open: **Command Prompt**
2. Grab this VM's IP address with the following command and write down the IP address result: `ipconfig`

    > Remember, your IP address is the one you configured with the **10.0.`UID`.1** address. We're running `ipconfig` to confirm it's still there.

3. Over on your **Windows Client (client1) VM**, open Command Prompt.
4. Enter the following command to test our ability to talk to _srv1_: `ping "srv1-ipaddress"` where _srv1-ipaddress_ is the IP address from Step 2.

      > Example: `ping 10.0.40.1`

5. **It doesn't work, does it?** That's normal at this stage and we'll fix it below.
6. Continue to **Part 4**.

### Part 2: Windows Server GUI (srv1) - Applying Firewall Rules

We will now apply our security-conscious policy by configuring the firewall on this server.

At the moment, the server's firewall is configured using defaults. As mentioned above, **defaults are a security risk** as they are known to everyone and can be used against you. If an attacker _doesn't_ know your configuration, it's harder for them to know what's open and what's not and how to attack.

As you saw from **Part 1**, ping between _client1_ and _srv1_ didn't work.

By default, the ability to ping a server is turned off. A ping (a type of ICMP packet) is typically used to see if you can get a response from a server. Having a server respond let's an attacker know there's a machine there that they can then try to break into.

  > **Imagine a burglar knocks on a random brick wall (sends a ping).** If you (the server) knock back (send a ping reply), the burglar now knows there's a person there who probably has some nice stuff! Don't knock back, and the burglar has no idea what's on the other side of that wall and will move on.

We want to turn on ping so we can test connections between our machines, but we have to be careful. Turning that on through the firewall too broadly opens us up to that vulnerability.

We are going to turn it on _only_ for our local network. Our VMs will be allowed to ping each other, but anything outside of our subnet (10.0.`UID`.0/24) can't. Best of both worlds.

To do this, we're going to work with the **Windows Defender Firewall** on srv1.

1. In the _Server Manager_ application, go to _Local Server_.
2. In the _Properties_ section of this page, look for the **Microsoft Defender Firewall** line item.
3. Next to the line, it should say **Public: On**. Click on this.
4. The _Windows Security > Firewall & Network protection_ application opens.
5. On this screen, you will see three networks: _Domain network_, _Private network_, and _Public network_.
6. We'll spend more time with these in later weeks, but for now, all three should say: **Firewall is on**.
7. Below this, click on the link that says: **Advanced settings**
8. This opens the _Windows Defender Firewall with Advanced Security_ application.
9. On the first page, you'll notice the same three profiles: _Domain Profile_, _Private Profile_, and _Public Profile_.
10. All three have the same overall rules:
11. **Windows Defender Firewall is on**: This just confirms the firewall is active for this network profile.
12. **Inbound connections that do not match a rule are blocked**: This means that all incoming connections and requests (like trying to ping this machine) are blocked _by default_. If you want to allow a certain type of connection or service into your server, you have to make a specific rule for it. This is whitelisting and our _Principle of Least Privilege_ in action.
13. **Outbound connections that do not match a rule are allowed**: This means all network data leaving the server is automatically allowed. The logic here is that if the server is the one deciding the send out information, it's likely fine. (Note: The only time we ever whitelist outbound connections is for specialized security settings like government compliance.)
14. At the moment, we're dealing with the _Public Profile_ context. Let's allow ICMP ping!
15. On the left-hand menu bar, click **Inbound Rules**.
16. This loads a ton of already-defined rules. Thankfully, the one we need has already been defined. We just need to turn it on.
17. At the top of this area, click **Name** to order the list by name (this is not the default).
18. Now, scroll down until you can find the following and double-click it: **Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)**, **Private, Public**.
19. **Note:** There are two of these rules! Look for the one that has **Private, Public** under the Profile header.
20. This opens this rule's configuration window, starting on the _General_ tab.
21. In the _General_ tab, look for the following and check the box next to it: **Enabled**
22. Click **Apply**, then **OK**.
23. Back in the main _Inbound Rules_ window, you should see that same rule line now says **Yes** under the _Enabled_ header.
24. Repeat the same steps to enable: **Core Networking Diagnostics - ICMP Echo Request (IMPv4-In), Domain**
25. You've enabled ICMP ping! Let's go test it.
26. Switch back to your _client1_ machine, and run the same ping you ran earlier pointing to srv1. (Refer to _Part 1, Step 4_.)
27. Does it work?
28. **If it does, congratulations!** You've just enabled ping for connectivity checking to _srv1_ and gone through your first foray into the Windows Firewall.

> ℹ️ **A note for later labs:**
> By default, this rule is set to only allow incoming pings (ICMP requests) from computers on your **VM network**—that is, other virtual machines on the same VMware NAT or host-only network as your server—not from the wider Internet or any physical computers in the classroom.
>
> You can see this by checking the **Scope** tab in the rule’s properties, where “Remote IP address” is set to “Local subnet.”

### Part 3: Windows Server Core (srv2) - Applying Firewall Rules

Let's apply the same incoming ping firewall rule to our Server Core machine so we can check it's network connectivity as well.

1. Login to your Server Core _(srv2)_ machine.
2. The _sconfig_ text-based application automatically launches.
3. Select option 8 to find this machine's **Internal Network** IP address. Write it down.
4. Back in your Windows 11 Client (_client1_), try to ping this address. Does it work?
5. Just as in _srv1_, it doesn't.
6. Go back to _srv2_.
7. If you're still in the _Network settings_ page, leave the field blank and hit **Enter** to go back to the main screen.
8. Select option 15 to exit to PowerShell.
9. As there's no GUI, we need to use PowerShell for firewall management.
10. Let's take a look at the existing rules, just like in Part 2. Run the following PowerShell command:

```powershell
Get-NetFirewallRule | Where-Object DisplayName -Like '*ICMPv4-In*'
```

1. Several results appear in the search. Look for the one with the name: **Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)**
2. This firewall rule object, starting from _Name_ all the way down to _PackageFamilyName_ lists all the configured properties for this rule.
3. Read through these properties. Recognize any from our GUI version?
4. This rule matches both our incoming ping requests and keeps to _Private, Public_ profiles. As with our GUI version, the scope is defaulted to local subnet only.
5. Let's turn on this incoming firewall rule by running the following command:

```powershell
Enable-NetFirewallRule -Name CoreNet-Diag-ICMP4-EchoRequest-In
```

1. This selects the right rule and enables it.
2. One of the things you **must** get into the habit of doing with CLI commands is _**double-checking your work**_.
3. Let's do that now by asking the system if it was actually enabled:

```powershell
Get-NetFirewallRule -Name CoreNet-Diag-ICMP4-EchoRequest-In
```

1. Look for the _Enabled:_ field. See how it's changed to **True**? It worked!
2. Optional: We can even check the scope for local subnet only as we did in the GUI version if we want with the following command:

```powershell
Get-NetFirewallRule -Name CoreNet-Diag-ICMP4-EchoRequest-In | Get-NetFirewallAddressFilter
```

Output:

```
LocalAddress  : Any
RemoteAddress : LocalSubnet4
```

1. Last, because there are two incoming ICMP rules (_Domain_ profile and _Private, Public_ profile), let's check that only the rules are enabled:

Command:

```powershell
Get-NetFirewallRule -DisplayName "Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)" | Select-Object DisplayName, Profile, Enabled
```

1. This shows a brief status of both rules. Both versions should show **True**.

Output:

```
DisplayName                                                         Profile Enabled
-----------                                                         ------- -------
Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)          Domain   True
Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In) Private, Public    True
```

1. Now, let's test our ping. Switch over to your Windows 11 Client (_client1_).
2. Open a Command Prompt window, and run the following command: `ping srv2-ipaddress` (Refer to _Part 2, Step 3_).
3. Does it work?
4. **If it does, congratulations!** You've just enabled ping for connectivity checking to _srv2_ and gone through your first foray into the PowerShell!

### Part 3: Windows Client (client1) - Applying Firewall Rules

Finally, let's enable ping on our Windows Client machine.

1. Login to _client1_.
2. Click the **Start** button and type the following search: **firewall**
3. Of the options that appear, select: **Windows Defender Firewall with Advanced Security**
4. Just as in _Part 2_ with _srv1_, navigate to **Inbound Rules**.
5. Click the _Name_ header to sort by name.
6. Find the following rules:
7. Name: **Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)**
8. Profiles: **Private, Public** and **Domain**
9. Double-click it to open the rule's configuration settings.
10. In the _General_ tab, find and check the box next to **Enabled**.
11. Click **Apply**, then **OK**.
12. Open a _Command Prompt_ window and run the following to get your client's IP address: `ipconfig`
13. Switch to your _Server Core (srv2)_ machine.
14. In PowerShell, run the following command: `ping client1-ipaddress` (Where client1-ipaddress is the address from Step 10.)
15. Example: `ping 10.0.40.11`
16. Does it work?
17. **It if does, congratulations!** You've just enabled ping for connectivity checking to _client1_ and have full connectivity checking for your entire environment! This will become _**very**_ handy in Labs 3-4.

## Investigation 2: Remote Management - Windows Server GUI (_srv1_)

While you have direct access to all three VMs from VMware, these days most machines are remote. This means you typically do not have direct, physical access to the servers.

Even if you do, having remote access to allow you to manage your servers from the comfort of your office (or home!) instead of walking to each physical machine is far better.

In this investigation, we'll set up remote access to our Windows Server GUI (srv1) so we can connect to it using your Windows 11 Client VM.

### Part 1: Enabling Remote Desktop Connections

In this part, we'll turn on Remote Desktop (RDP) on the server. This will allow you to remotely connect to _srv1_ from your Windows 11 Client.

1. In the _Server Manager_ application, navigate to _Local Server_.
2. In the main _Properties_ window, find the line entry for: **Remote Desktop**
3. Click the link next to it: **Disabled**
4. In the new _System Properties_ window, you should automatically be on the _Remote_ tab.
5. Look down to the section on **Remote Desktop**.
6. Toggle the option for: **Allow remote connections to this computer**
7. Ensure the checkbox next to this is on: **Allow connections only from computers running Remote Desktop with Network Level Authentication (recommended)**
8. Click **Apply**, then **OK**.

### Part 2: Adding your RDP Firewall Rules

In this part, we'll add a firewall rule to allow the connection over the local network.

1. On _srv1_, open **Windows Defender Firewall with Advanced Security**.
2. In the _Inbound Rules_ section, look for the following rules:
    1. **Remote Desktop - Shadow (TCP-In)**
    2. **Remote Desktop - User Mode (TCP-In)**
    3. **Remote Desktop - User Mode (UDP-In)**
3. If they're all enabled (they should be), you're good to go!
4. Why check? Always check your firewall rules when enabling a new service, even if it _usually_ enables the associated firewall rule automatically. **Never assume!**
5. If these rules are **not** enabled, enable them now.

### Part 3: Connecting to Windows Server (_srv1_) from Windows 11

In this part, we'll verify our work by connecting to the server using our Windows 11 client VM.

1. Make sure you've got your _srv1_ IP address from earlier. (You wrote it down, _right_?)
2. On _client1_, click the **Start** menu.
3. Search for and open: **Remote Desktop Connection**
4. In the newly opened application, in the _Computer_ field, enter the IP address for _srv1_.
5. Click **Connect**.
6. Enter your _srv1_ username and password, then click **OK**.
7. A security dialogue box pops up. **Check the box next to _"Don't ask me again for connections to this computer"_ before the next step.**
8. Click **Yes**.
9. If you can see your _srv1_ desktop, congratulations! You now have remote access.
10. To quit the remote session, find the floating HUD at the top of the screen with the IP address and click on the **X** icon.
11. Note: **This will not shutdown your _srv1_ VM. It only ends your remote session.**
12. Note 2: A remote session will automatically lock your VM's direct session. You will need to unlock it again when you go back to it through VMware Workstation directly.

## Investigation 3: Remote Management - Windows Server Core (srv2) with Remote Desktop

In this investigation, we'll set up remote access to our Windows Server Core (srv2) so we can connect to it using your Windows 11 Client VM.

### Part 1: Enabling Remote Desktop Connections

In this part, we'll turn on Remote Desktop (RDP) on the Core server (_srv2_).

1. Login to _srv2_.
2. If you're at the PowerShell command prompt, run the following command to get back into the text-based _Server Manager_ application: `sconfig`
3. Select option 7.
4. In the _Remote desktop_ page, select option **E** to enable Remote Desktop.
5. It will ask you to select a security level. Select **Option 1**.
6. Press **Enter** to complete and go back to the main _Server Manager_ page.
7. **Double-check your work.** Select option 7 again.
8. Is the _Remote desktop status_ changed and correct?
9. If not, go through Steps 4-9 again.
10. If yes, press **Enter** without entering any options to go back to the main screen without making any changes.

### Part 2: Adding a RDP Firewall Rules

In this part, we'll check that the RDP firewall rules have been enabled.

1. Select **Option 15** to exit to PowerShell.
2. Run the following PowerShell command to check the status of your RDP firewall rules:

Command:

```powershell
Get-NetFirewallRule | Where-Object { $_.DisplayName -like '*Remote Desktop*' -and $_.Direction -eq 'Inbound' } | 
    Select-Object Name, DisplayName, Profile, Enabled
```

Output:

```
Name                          DisplayName                         Profile Enabled
----                          -----------                         ------- -------
RemoteDesktop-In-TCP-WS       Remote Desktop - (TCP-WS-In)            Any   False
RemoteDesktop-In-TCP-WSS      Remote Desktop - (TCP-WSS-In)           Any   False
RemoteDesktop-Shadow-In-TCP   Remote Desktop - Shadow (TCP-In)        Any    True
RemoteDesktop-UserMode-In-TCP Remote Desktop - User Mode (TCP-In)     Any    True
RemoteDesktop-UserMode-In-UDP Remote Desktop - User Mode (UDP-In)     Any    True
```

1. Check that your output matches the above. This is the same as on _srv1_, just at the command line. Notice which are enabled and which are not?
2. If you need to, grab the IP address for _srv2_ here by running this command in PowerShell: `ipconfig`

### Part 3: Connecting to Windows Server (srv2) from Windows 11 via RDP

In this part, we'll verify our work by connecting to _srv2_ using our Windows 11 client VM. This is essentially the same as our instructions for _srv1_, but with a different IP address.

1. Make sure you've got your _srv2_ IP address from earlier. (You wrote it down, _right_?)
2. On _client1_, click the **Start** menu.
3. Search for and open: **Remote Desktop Connection**
4. In the newly opened application, in the _Computer_ field, enter the IP address for _srv2_.
5. Click **Connect**.
6. Enter your _srv2_ username and password, then click **OK**.
7. A security dialogue box pops up. **Check the box next to _"Don't ask me again for connections to this computer"_ before the next step.**
8. Click **Yes**.
9. If you can see your _srv2_ Command Prompt window, congratulations! You now have remote access.
10. To quit the remote session, find the floating HUD at the top of the screen with the IP address and click on the **X** icon.
11. Note: **This will not shutdown your _srv2_ VM. It only ends your remote session.**
12. Note 2: A remote session will automatically lock your VM's direct session. You will need to unlock it again when you go back to it through VMware Workstation directly.

## Investigation 4: Remote Management - Windows Server Core (srv2) with SSH

### Part 1: Enabling SSH Connections

In this part, we'll turn on incoming SSH connections so we can connect to our server using Visual Studio Code from the Windows 11 client.

1. Login to your Server Core _(srv2)_.
2. Exit the _Server Manager_ program by selecting **Option 15**.
3. In PowerShell, run the following command to install the OpenSSH Server

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

1. After it installs, we have to start the SSH service. Run the following:

```powershell
Start-Service sshd
```

1. We now have to set the SSH service to start automatically every time the system boots. Run the following:

```powershell
Set-Service -Name sshd -StartupType 'Automatic'
```

 1. As with our other work, let's confirm the SSH service is running:

  ```powershell
  Get-Service sshd
  ```

1. You should have a response that looks like this:

```powershell
Status   Name               DisplayName
------   ----               -----------
Running  sshd               OpenSSH SSH Server
```

### Part 2: Adding an SSH Firewall Rule

1. We'll now add an SSH rule. One already exists, but it's only attached to the _Private_ profile and would not work for us.
2. Instead, we'll create a custom rule. Run the following command:

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22 -Profile Private,Public
```

1. This rule sets the following:
    1. Name: sshd
    2. Display Name: OpenSSH Server (sshd)
    3. Enabled: True
    4. Profile: Private, Public
    5. Direction: Inbound
    6. Action: Allow
    7. Protocol: TCP
    8. Port: 22
2. Verify your work! First, let's look at the firewall object by running the following command:
3. Now, let's verify the protocol and port with the following command:

Command:

```powershell
Get-NetFirewallRule -Name 'sshd'
```

Output:

```powershell
Name                          : sshd
DisplayName                   : OpenSSH Server (sshd)
Description                   :
DisplayGroup                  :
Group                         :
Enabled                       : True
Profile                       : Private, Public
Platform                      : {}
Direction                     : Inbound
Action                        : Allow
EdgeTraversalPolicy           : Block
LooseSourceMapping            : False
LocalOnlyMapping              : False
Owner                         :
PrimaryStatus                 : OK
Status                        : The rule was parsed successfully from the store. (65536)
EnforcementStatus             : NotApplicable
PolicyStoreSource             : PersistentStore
PolicyStoreSourceType         : Local
RemoteDynamicKeywordAddresses : {}
PolicyAppId                   :
PackageFamilyName             :
```

Command:

```powershell
Get-NetFirewallRule -Name 'sshd' | Get-NetFirewallPortFilter | Select-Object Name, Protocol, LocalPort
```

Output:

```powershell
Name Protocol LocalPort
---- -------- ---------
     TCP      22
```

1. If your own output looks like the above, congratulations! If not, try your configuration commands again or ask for help.

### Part 3: Connecting to Windows Server Core (srv 2) from Windows 11 via SSH

1. On your Windows 11 Client (_client1_), open Command Prompt.
2. Enter the following command: `ssh Administrator@srv2ipaddress` (Where _srv2ipaddress_ is your actual _srv2_ IP address.)

    > Example: `ssh Administrator@10.0.40.2`

3. The first time you connect, a prompt will ask you if you are sure. Type **yes** and hit **Enter**.
4. Type your password when asked and hit **Enter**.
    > ℹ️ **Note:** By default, You will _not_ see asterisks or other characters as you type. **This is normal.** It is taking your keyboard input. It may take you a few tries to get used to it.
5. Once logged in, you are now on _srv2_'s Command Prompt environment.
6. To get back into PowerShell (which you should), run: `powershell`
7. It is **not** recommended to run the _sconfig_ program over an SSH connection. If you need to use the _sconfig / Server Manager_ text-based program, connect through your RDP connection in Part 1.

> ℹ️ **A note about SSH and _Server Manager_:**  
> Your SSH connection will allow you to run the _sconfig_ Server Manager program and load it. However, many of it's functions fail over an SSH connection because it relies on background interactive processes that can't run on an SSH session. They must be run from the RDP connection, despite it _looking_ the same.  
>
> For an example of this, run `sconfig` from your SSH session and try to run Windows Updates. If it finds any and you try to install them, it will fail. This is an SSH+sconfig problem; run the same thing through RDP and it will work fine.

## Lab 2 Sign-Off

It’s essential to complete Lab 2 correctly, as the networking setup will be required for all advanced labs.

When you finish Lab 2, ask your instructor for a sign-off.

### Sign-Off Checklist

On _client1_:

1. Ping _srv1_
1. Ping _srv2_
1. SSH into _srv2_
1. RDP into _srv2_
1. In Firefox, test your Internet connection by loading: **eff.org**

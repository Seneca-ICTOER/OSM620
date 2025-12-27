---

id: lab5
title: Lab 5 - Implementing DHCP
sidebar_position: 5
description: Lab 5 - Implementing DHCP
---

# Lab 5 - Implementing DHCP

## Lab Preparation

### Purpose / Objectives of Lab 5

In this lab, you will deploy **DHCP** on *srv1* to automatically provide IP configuration for your internal lab network.

By the end, you will:

1. Install the **DHCP Server** role on *srv1*.
2. Create an IPv4 **scope** for your internal subnet **10.0.`UID`.0/24** with proper **options** (Router, DNS servers, DNS Suffix).
3. Create **reservations** so *srv2*, *client1*, and *client2* receive predictable addresses that match prior labs.
4. Flip *srv2*, *client1*, and *client2* from **static** addressing to **DHCP**, then verify leases.
5. Validate **DHCP + DNS** end‑to‑end with `ipconfig`, `nslookup`, and `ping`.

### Minimum Requirements

Before beginning, you must have:

* Successfully completed **Lab 4 – Implementing DNS**, with *srv1* resolving internal names and using loopback for local DNS testing.
* Your **UID** value handy.
* All four VMs present from earlier labs: *srv1* (GUI), *srv2* (Core), *client1*, *client2*.

### Key Concepts

* **DHCP** automatically assigns IP addressing (IPv4 address, subnet mask), **Default Gateway (Option 003)**, **DNS servers (Option 006)**, and **DNS Suffix (Option 015)** to clients on a subnet.
* **Scope** defines the usable address pool on a network. **Reservations** ensure specific MAC addresses always receive the same IP.
* In this course, your internal network remains **10.0.`UID`.0/24**; *srv1* is the router at **10.0.`UID`.1**.

> **Design Note:** We will keep client addresses consistent with Labs 2–4 using **reservations** so your existing DNS host records remain correct.

---

## Investigation 1: Install the DHCP Server Role on *srv1*

### Part 1: Add the DHCP Server Role

In this investigation, we'll install the DHCP role on our server so other machines on our network can get their IP address and network configuration from *srv1*.

1. **Login to srv1** with your Administrator account.
2. Open **Server Manager** → **Manage > Add Roles and Features**.
3. **Installation Type**: *Role-based or feature-based installation* → **Next**.
4. **Server Selection**: choose **srv1** → **Next**.
5. **Server Roles**: check **DHCP Server** → **Add Features** if prompted → **Next** through *Features* and *DHCP Server* pages.
6. **Confirmation**: **Install** and wait for completion.

### Part 2: Post‑Install Configuration

1. In **Server Manager**, click the yellow **!** → **Complete DHCP configuration**.
2. In the wizard, click **Commit** → **Close**.

Once fully complete, move on to the next investigation.

## Investigation 2: Create Scope and Options (10.0.`UID`.0/24)

In this investigation, we'll configure the **scope**. 

A DHCP scope is , basically, what slice of the network are we going to use and define. (Example are 10.0.45.0/24, 192.168.1.0/24, 172.16.0.0/16, etc.)

Here, we're going to tell DHCP we want to manage the **10.0.`UID`.0/24** network. Any machine that connects to our *Internal Network* will get its IP address, subnet mask, default gateway, and DNS using what we set up below.

In short: *srv1* will give other machines their IP addresses automatically, along with other network configuration.

### Part 1: Create a New IPv4 Scope

1. **Tools > DHCP**.
2. Expand your server (srv1) → **IPv4** → **Right‑click > New Scope…**
3. **Scope Name**: `OSM620 HQ`
4. **IP Address Range**:

   * **Start**: `10.0.UID.1`
   * **End**: `10.0.UID.254`
   * **Subnet mask**: `255.255.255.0`
5. **Add Exclusions and Delay**: Add the following:
    * `10.0.UID.1`
    * `10.0.UID.200` to `10.0.UID.254`
    * Leave the delay blank.
6. **Lease Duration**: default (8 days) → **Next**.
7. **Router (Default Gateway) – Option 003**: `10.0.UID.1` → **Add** → **Next**.
8. **Domain Name and DNS Servers – Option 006/015**:

   * **Parent domain** (Option 015): `yourSenecaUsername.com` *(replace with your lab domain from Lab 4)*
   * **DNS Servers** (Option 006): 
       * `10.0.UID.1` *(srv1)*
       * **149.112.121.20** *(CIRA)*
9. **WINS Servers**: leave blank → **Next**.
10. **Activate Scope**: select **Yes, I want to activate this scope now** → **Finish**.

> **Why start at `.2`?** `.1` is the gateway on *srv1*. We want to make sure our DHCP server can't give out the IP address that *srv1* already uses!

## Investigation 3: Reservations for Predictable Addresses

By default, a DHCP server will assign IP addresses to other machines randomly from the range you defined earlier. (10.0.`UID`.2 - 10.0.`UID`.254)

Every time one of your other machines connect, it could get *any* IP address inside that range. It will always start with 10.0.`UID`., but that last number (octet) could be anything between 2-254. It can and will change from day to day.

In this investigation, we're going to use **Reservations** to let us decide, from *srv1*, what IP addresses each of our machines should *always* have. After we're done, when srv2 gets an IP address, it won't be random. It will be what we decide.

We’ll bind specific IPs to each machine’s **MAC address** so they keep the same addresses used in Labs 2–4.

### Part 1: Collect Physical Addresses From Each NIC

To set this up, we need to grab the physical address (otherwise known as MAC address) of each network card on each computer. This is how *srv1* will known which machine is which when it needs to give out IP addresses.

Write down the physical address for each:

1. On **srv2** (Core): `ipconfig /all` → record the **Physical Address** for the **Internal Network** adapter.
2. On **client1**: `ipconfig /all` → record Physical Address.
3. On **client2**: `ipconfig /all` → record Physical Address.

### Part 2: Create Reservations on *srv1*

In this part, we'll know create a DHCP reservation for each computer. This is how we make sure they get the specific IP address we want them to have, instead of a random one.

Each computer will have its own reservation. This is like a record.

In **DHCP Manager** → **IPv4 > OSM620 HQ > Reservations** → **Right‑click > New Reservation…**

Create the following (replace `UID` and MACs):

* **srv2**

  * **IP address**: `10.0.UID.2`
  * **MAC**: `<srv2 MAC>`
  * **Description**: `Windows Server 2025 Datacenter Core`
  * **Supported types**: **Both** → **Add**
* **client1**

  * **IP address**: `10.0.UID.11`
  * **MAC**: `<client1 MAC>`
  * **Description**: `Windows 11 Education`
  * **Supported types**: **Both** → **Add**
* **client2**

  * **IP address**: `10.0.UID.12`
  * **MAC**: `<client2 MAC>`
  * **Description**: `Windows 11 Education`
  * **Supported types**: **Both** → **Add**

Verify all three appear under **Reservations**.


## Investigation 4: Switch from Static to DHCP

Right now, you're other computers are using **manual static network configuration**. You set up their IP addresses manually.

In this investigation, we'll switch our other machines back to using DHCP so they can get their network information, including IP address, from *srv1*. 

### Part 1: Swtiching *srv2* (Server Core)

1. Login as **Administrator**.
2. In PowerShell, identify the interface index of your network adapter (look for **IfIndex**):

   ```powershell
   Get-NetIPInterface
   ```
3. Enable DHCP for IPv4 (replace `4` with your **IfIndex** on the Internal Network adapter):

   ```powershell
   Set-NetIPInterface -InterfaceIndex 4 -Dhcp Enabled
   Set-DnsClientServerAddress -InterfaceIndex 4 -ResetServerAddresses
   ipconfig /release
   ipconfig /renew
   ipconfig /all
   ```
4. Run `ipconfig /all` to confirm the following:
 **IPv4 Address = 10.0.UID.2**, **Default Gateway = 10.0.UID.1**, **DNS = 10.0.UID.1**.

### Part 2: *client1* (Windows 11)

1. **Settings > Network & Internet > Ethernet > Edit** next to **IP assignment** → **Automatic (DHCP)** → **Save**.
2. **Edit** next to **DNS server assignment** → **Automatic (DHCP)** → **Save**.
3. Command Prompt:

   ```
   ipconfig /release
   ipconfig /renew
   ipconfig /all
   ```
4. Confirm **IPv4 = 10.0.UID.11**, **Gateway 10.0.UID.1**, **DNS 10.0.UID.1**.

### Part 3: *client2* (Windows 11)

Repeat the exact steps as client1. Confirm **IPv4 = 10.0.UID.12**.

> If a machine doesn’t pick up the reserved address, re‑check the MAC you entered in the reservation and ensure the NIC you changed to DHCP is the **Internal** (HQ) adapter.

## Investigation 5: Verify Leases and Name Resolution

In this investigation, we'll run checks to confirm that our machines are:
* Using DHCP
* Have an IP address
* Can connect to other machines
* Can connect to the Internet

### Part 1: Check Leases in DHCP Manager

1. On *srv1*: **DHCP Manager** → **IPv4 > OSM620 HQ > Address Leases**.
2. Confirm entries for **srv2**, **client1**, **client2** with correct **Reservation** type and IPs.

### Part 2: Test DHCP + DNS

From **client1** (or **client2**):

```powershell
ipconfig /all
nslookup srv1.yourSenecaUsername.com
nslookup client2.yourSenecaUsername.com
nslookup eff.org
ping srv1.yourSenecaUsername.com
```

* All should resolve and ping (ICMP allowed per Lab 3). If `nslookup` for Internet names fails, revisit **DNS Forwarders** on *srv1*.

## Troubleshooting

* **Client keeps old static IP**: ensure **IP assignment = Automatic (DHCP)**; on Core, run `Set-NetIPInterface -Dhcp Enabled` then `ipconfig /release` + `/renew`.
* **Gets wrong pool address**: verify the **reservation MAC** matches the **Internal adapter**; remove/re‑add reservation if needed.
* **No default gateway**: re‑check **Option 003** in scope options.
* **DNS wrong**: verify **Option 006** points to `10.0.UID.1` and **Option 015** is your lab domain.
* **No Internet**: confirm *srv1* still NATs traffic between Internal and External (Lab 2 RRAS) and that **Forwarders** work (Lab 4).

## Lab 5 Sign‑Off

To complete Lab 5, show your instructor:

1. **Scope** `10.0.UID.2–199` active with Options **003**, **006**, **015** set.
2. **Reservations** for **srv2 (10.0.UID.2)**, **client1 (10.0.UID.11)**, **client2 (10.0.UID.12)**.
3. On each client, `ipconfig /all` showing **DHCP Enabled = Yes** and the correct leased address.
4. In Address Leases, entries for all three machines marked as **Reservation**.
5. `nslookup` for `srv1.yourSenecaUsername.com` **and** an Internet site returns addresses successfully.

> **You’re done!** Your lab environment now uses **DHCP** correctly while keeping earlier labs’ addressing and DNS intact.

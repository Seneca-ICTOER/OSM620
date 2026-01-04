---
id: lab6
title: Lab 6 - AD Users, Groups, Computers, and GPOs
sidebar_position: 6
description: Lab 6 - AD Users, Groups, Computers, and GPOs
--- 

# Lab 6 - AD Users, Groups, Computers, and GPOs

## Lab Preparation

### Purpose of Lab 6

In this lab, you’ll reshape your domain into something closer to real life:

* Build a clean **OU (Organizational Unit)** structure for **people** and **computers**.
* Stop using the built-in **Administrator** and switch to a **personal admin** account.
* Use **GPOs** (Group Policy Objects) to control **how things behave**.
* Use **Groups + Delegation** to control **who is allowed to do what** in Active Directory.
* You’ll prove the difference between “policy” and “permission” with three concrete users:
  * **Bob (Accounting)** — a normal employee who gets a **lockdown** user policy.
  * **Enzo (IT L1)** — can **reset passwords** in a department.
  * **Dot (IT L2)** — can **create/modify/move/delete** users in department OUs.
* Finally, you’ll create the start of a **Computers OU** structure so **device-level** settings have a proper home.

### Objectives

By the end of this lab, you will be able to:

* Create a tidy **HQ\Users** scaffold and a starter **HQ\Computers** scaffold.
* Create a **personal domain admin account**, verify access, and **disable** the built-in **Administrator** safely.
* Create a **User-scope GPO** (**User – Employee Lockdown**), link it to a specific **Accounting** OU, and verify that it follows the user (Bob) wherever he signs in.
* Create a **User-scope GPO** for IT (**User – IT Environment**), link it to **HQ\Users\IT**, and verify that Enzo/Dot get the intended desktop experience.
* Create ***Global Groups (GG_)*** to represent **who people are** and **Role Groups (RG_)** to represent **what a role can do**; wire them together and **delegate**:
  * **L1**: reset passwords in a department OU (e.g., Accounting).
  * **L2**: create/modify/move/delete users in department OUs (not IT).
* Build the first **Computers OUs** by hand, then complete the scaffold with a small **CSV > PowerShell script**; move *srv1* and *laptop1* into place.

### Concepts

This section is your mental model. Read it once now; refer back while you work.

#### Users vs Computers (two different kinds of objects)

* **User object** = a person. When a **User-scope** GPO applies, it **follows the person** to any domain-joined machine they sign in to.  
  
  **Example:** Bob gets “no Control Panel” everywhere he signs in.

* **Computer object** = a device. When a **Computer-scope** GPO applies, it **sticks to that device**, no matter who signs in.  

  **Example:** Laptop sleep/USB rules apply for every user on that laptop.

> **Checkpoint:** If the question is “Does this follow the person or the device?” — you already know which scope to use.

#### OUs: what they are (and are not)

* An **OU (Organizational Unit)** is a folder inside AD used for **targeting** and **delegation**:
  * You **link GPOs** to OUs to target either the **users in that OU** (User-scope) or the **computers in that OU** (Computer-scope).
  * You **delegate control** on an OU to allow a group to perform tasks (like reset passwords) **for the objects inside that OU**.

* An OU is **not** a security principal. You cannot “give permissions to an OU.” You always grant permissions to **groups** (or users), then point those groups at an OU via **delegation**.

> **Analogy:** Think of OUs as labeled **folders**. Policies and delegation are **rules on the folder**. Groups are the **keyrings** you hand to people.

#### Groups: Global Groups VS Role Groups

We use a simple naming convention to keep your head clear:

* **`GG_*` (Global Group)** = **who they are** (identity buckets).  
  You put **users** into **GGs**.  

  **Examples:** `GG_IT_L1`, `GG_IT_L2`.

* **`RG_*` (Role Group)** = **what they can do** (delegated abilities).  
  You assign **rights** to **RGs** by using the **Delegation of Control** wizard on an OU.  

  **Examples:** `RG_PasswordReset`, `RG_OUAdmin_AllDepts`.

**Wiring rule:** *Put the GG inside the RG.*  

* You never delegate directly to a user in production.  
* You delegate to an **RG** on the target **OU**, then make the **GG** a **member** of that **RG**.  
* Users gain the delegated ability by being in the **GG**.

> **Example (you’ll do this):**  
> Delegate **“Reset user passwords”** on `HQ\Users\Accounting` **to** `RG_PasswordReset`.  
>
> Make `GG_IT_L1` a **member of** `RG_PasswordReset`.  
>
> Add **Enzo** to `GG_IT_L1`.  
>
> Result: Enzo can reset passwords **in that OU**.

#### Delegation of Control (where + what + who)

* **Where:** right-click the **target OU** (e.g., `HQ\Users\Accounting`) > **Delegate Control…**
* **What:** pick the task(s) (e.g., **Reset user passwords**, or **Create, delete, and manage user accounts**).
* **Who:** choose the **RG** (e.g., `RG_PasswordReset`, `RG_OUAdmin_AllDepts`).

> **Important:** The rights you delegate **apply only to objects in that OU (and its child OUs)**.  
> If you want the same right in another department, you repeat the delegation on that department’s OU.

#### GPOs: two halves, linked to an OU

A **GPO** is a bundle of settings with two halves:

* **User Configuration** — shapes the **user session** (Start menu, Control Panel access, etc.).  
  **Link** it to a **Users** OU to target those users.
* **Computer Configuration** — shapes the **device** (firewall, power settings, removable storage).  
  **Link** it to a **Computers** OU to target those devices.

We avoid advanced tricks in this lab (no **Block Inheritance**, no **Enforced**). We simply **link at the lowest sensible OU** so there’s no ambiguity.

> **Examples you’ll build:**  
>
> * **User – Employee Lockdown** *(User)* > link to `…\Users\Accounting`.  
> * **User – IT Environment** *(User)* > link to `…\Users\IT`.  
> * **Computer – Workstations Baseline** *(Computer)* > link to `…\Computers\Workstations`.  
> * **Computer – Laptops Baseline** *(Computer)* > link to `…\Computers\Laptops`.

#### Policies VS Permissions (why both exist)

* **Policy** = **configuration/behavior** the system adopts.

  *Example:* “Hide Control Panel” (policy) changes the UI for Bob.

* **Permission** = **authority** to perform an action in AD.  

  *Example:* “Reset passwords in Accounting” (delegation) gives Enzo the right to change those user objects.

They solve **different problems** and often work together:

* policies make the environment safe and consistent
* permissions define who can administer AD

### Minimum Progression Requirements

Before beginning, you must have:

1. **Successfully graded *Lab 6***.
1. Attended the **Week 10 - AD Users, Groups, Computers, and GPOs** lecture.
1. Your external SSD (or personal computer) with your VMs.
1. Your OSM620 Lab Logbook.
1. Optional, but recommended: Caffeine delivery system.

### Pre-Flight Check

Have the following VMs turned on:

1. router
1. srv1
1. srv2
1. srv3

## Investigation 1: Create Personal Domain Controller User

We’re going to give the domain a proper identity backbone before we do anything clever. That means building a clean **HQ\Users** OU path, creating a named administrator for yourself, and retiring the **built-in Administrator** account. Named admins are audit-friendly and safer. OUs (not the default *Users* container) are where policy and delegation behave predictably.

You’ll create your personal admin, verify it on both DCs, add a break-glass backup admin, and then disable the built-in Administrator.

**From this point on, every right you have will be applied purpose**, and every policy we link later will land exactly where we intend.

### Part 1: First OU Scaffolding Setup

Before we make any accounts, we need a proper home for them. The built-in **Users** container isn’t an OU, so you can’t link GPOs to it or delegate neatly. Building **HQ\Users** (with an **IT** branch) gives us a clean, predictable place where policy and delegation will behave the way we want.

1. On *srv1*, open **Active Directory Users and Computers** (ADUC).
1. Open your domain name from the list. (Example: *cjohnson30.com*)
1. Right-click your domain name and select: **New > Organizational Unit** (Do not use the built-in Users folder!)
1. Name this new OU: **HQ**
1. Using the same **New > Organizational Unit** function, create the following hierarchy:  

    > ```
    > HQ  
    > └─ Users  
    >    └─ IT  
    >       ├─ Helpdesk
    >       ├─ SecAdmins  
    >       └─ SysAdmins
    > ```

1. Verify your work. Does your HQ object hold the above items in the correct levels?
1. If so, move on to the next part. If not, fix your mistakes or ask for help.

### Part 2: Create Personal User

**Admins shouldn’t live and work in the default *Administrator* account.** A named admin is safer (auditing, accountability) and lets us save the built-in Administrator for emergencies. We’ll create your personal admin under **HQ\Users\IT\SysAdmins** so any IT-specific policy we link there applies to you automatically.

1. On **srv1** in ADUC, navigate to: **HQ\Users\IT\SysAdmins**
1. Right-click on *SysAdmins* (or the whitespace to the right) and select: **New > User**
1. Fill out the following in the *New Object - User* dialog box:  
    1. **First name:** *Your Proper First Name*
        > Example: Chris
    1. **Last name:** *Your Proper Last Name*
        > Example: Johnson
    1. **User login name:** *firstname.lastname*
        > Example: chris.johnson
    1. Click **Next**.
1. Fill out the following in the next page:
    1. **Password:** *same strong password as your other accounts*
    1. **Confirm password:** *same strong password as your other accounts*
    1. Check/uncheck the following:  
        1. **User must change password at next logon:** *Unchecked*
        1. **User cannot change password:** *Unchecked*
        1. **Password never expires:** *Checked*
        1. **Account is disabled:** *Unchecked*
    1. Click **Next**.
1. On the final page, review your settings and click **Finish**.

### Part 3: Add New Personal User to Domain Controllers

Now we grant your account the rights to actually run the domain. Adding it to **Domain Admins** makes the change global.

  > **NOTE:** For your new personal user to have those rights *applied*, you need to log off of that account and log back in. In this case, we haven't logged on to the account yet, so we can skip this step. But it's important to note for changes made to accounts currently logged on.

 We’ll also open Server Manager under your new personal account and re-add the other DCs (*srv2/srv3*).

1. On **srv1** in ADUC, navigate to: **HQ\Users\IT\SysAdmins**
1. Right-click on the user you created in Step 2 and click **Properties**.
1. In the *Properties* dialog box, click on the tab: **Member Of**
1. In the list, you should already see one entry, *Domain Users*. Click **Add**.
1. In the *Select Groups* dialog, type: **Domain Admins**
1. To verify an object named *Domain Admins* exists, click on the **Check Names** button. If it's correct, you'll see *Domain Admins* become underlined. (If not, check for typos or ask for help!)
1. Click **OK** to add.
1. Back in the *Member Of* tab, you should now see two entries. If so, click **OK** to fully apply.
1. On *srv1*, log out of your Administrator account and log back in using your new personal account by clicking on **Other user**.
    > Logging in using *firstname.lastname* should be sufficient, but if not, you can use either *`yourSenecaUsername\firstname.lastname`* or *`firstname.lastname@yourSenecaUsername.com`*.
1. Open *Server Manager > All Servers* and add *srv2* and *srv3*.

You now have full domain admin access using your personal account instead of the default Administrator account. This is far more secure and best practice.

> **IMPORTANT: Going forward, *always log into your servers using this personal account*, never the default Administrator account.**

### Part 4: Create a Secondary Backup Administrator

**This is your break-glass admin.** If your main account gets locked or broken, this one saves the day. It should work everywhere the primary does. We’ll test it right away so you know it works before you ever need it.

1. On **srv1** in ADUC, create a new user in: **HQ\Users\IT\SysAdmins**
1. Same steps as Part 3, but use the following values:  
    1. **Full name:** *Backup Admin*
    1. **User login name:** *backup.admin*
    1. All others the same as Part 3.
1. In this user's *Member Of*, add **Domain Admins**
1. Logout of srv1, log back in with this new user, and add *srv2/srv3* to Server Manager.
1. On **srv2**, log on with `backup.admin` to confirm you have backup access to your DCs.
1. If the previous step worked, congrats! Keep going. If not, stop and ask for help.

### Part 5: Disable Default Administrator Account

The built-in Administrator is a well-known target for attackers. Disabling it removes a giant bullseye without taking away any capability. Your named admin replaces it (as does your backup admin).

From now on, do everything with your personal account. **Make sure you're logged on to your *firstname.lastname* admin account before continuing!**

1. On **srv1** in ADUC, navigate to: **yourSenecaUsername.com\Users** (this is *not* inside HQ)
1. Find the **Administrator** account and right-click it.
1. From the drop-down menu, select: **Disable Account**

You've now disabled the default admin account and prevented a possible attack vector.

## Investigation 2: Bob from Accounting

**Now we add a real employee so we can see user-scope policy do real work.** We’ll carve out **HQ\Users\Accountants** and create **Bob Smith** as a standard user: no special rights, just a normal person who should get normal restrictions.

With that in place, we’ll link a **User-scope GPO** to the **Accountants OU** and watch the settings follow Bob wherever he signs in.

This is the first half of the model: people live under Users OUs, and user policy follows the person. It also sets us up to delegate tasks to IT without giving them domain-wide power in the next investigations.

### Part 1: Add Accountants OUs

Departments deserve their own space. By carving out **HQ\Users\Accountants**, we get a tidy spot to place real people and a perfect target for user-scope GPOs and delegation. Policies linked here will follow accountants wherever they sign in.

1. On **srv1** in ADUC, navigate to: **HQ\Users**
1. Using the **New > Organizational Unit** function, ***add*** the following to your existing hierarchy:  

    > ```
    > HQ  
    > └─ Users  
    >    └─ Accountants 
    > ```

1. Verify your work. Does your HQ object hold the above items in the correct levels?
1. If so, move on to the next part. If not, fix your mistakes or ask for help.

### Part 2: Create User - Bob Smith

Bob is our test user for “normal staff”. We’ll give him standard settings so we can see our **Employee Lockdown** user-GPO take effect exactly where we expect later in this investigation.

1. Create a new user in the following location: **HQ\Users\Accountants**
1. Sse the following values:  
    1. **First name:** *Bob*
    1. **Last name:** *Smith*
    1. **User login name:** *bob.smith*
1. Fill out the following in the next page:
    1. **Password:** *same strong password as your other accounts*
    1. **Confirm password:** *same strong password as your other accounts*
    1. Check/uncheck the following:  
        1. **User must change password at next logon:** *Checked*
        1. **User cannot change password:** *Unchecked*
        1. **Password never expires:** *Checked*
        1. **Account is disabled:** *Unchecked*

    > **Note:** Modern views on passwords are to always enable *Password never expires*. In the past, we disabled this option to deal with security issues where users would write down passwords, share them, lose them, etc.  A password would become invalid after a certain amount of time (say, 90 days).
    >
    > These days, we use **Multi-factor Authentication (MFA)** instead. We won't set that up in this lab, but it's a very typical addition to managed user accounts.

### Part 3: Create GPO, *User - Employee Lockdown*

Bob from Accounting is a *terrible* user. He ignores written computer polices and likes to mess around with settings due to boredom and 'fixing things' instead of contacting IT support. So, we create a **Group Policy Object (GPO)** to disable his ability to change important system settings.

We have to create the GPO first before we can have it applied to Bob (and other users we want).

1. On *srv1*, open the **Group Policy Management** application.
1. In the left-hand column, expand the following: **Forest: yourSenecaUsername.com > Domains > yourSenecaUsername.com > Group Policy Objects**
1. Right-click *Group Policy Objects* and select: **New**  
    1. **Name:** *User - Employee Lockdown*

### Part 4: Edit GPO, *User - Employee Lockdown*

Let's add those restrictions to our new **User - Employee Lockdown** GPO.

1. In the *Group Policy Objects* list, right-click on your newly created Group Policy Object (GPO) and click **Edit**.
1. This opens the **Group Policy Management Editor** application, and loads the **User - Employee Lockdown** GPO automatically.
1. We are now going to turn on certain restrictions.
1. Prohibit access to Control Panel and PC settings.  
    1. Navigate to: **User Configuration > Policies > Administrative Templates > Control Panel**.
    1. Find **Prohibit access to Control Panel and PC settings** in the list and double-click it to open.
    1. Click **Enabled** then **OK** to apply.
1. Repeat for the following:  
    1. User Configuration > Policies > Administrative Templates > System > **Prevent access to the command prompt = Enabled**
    1. User Configuration > Policies > Administrative Templates > System > **Prevent access to registry editing tools = Enabled**
    1. User Configuration > Policies > Administrative Templates > Start Menu and Taskbar > **Remove Run menu from Start Menu = Enabled**
    1. User Configuration > Policies > Administrative Templates > Windows Components > File Explorer > **Remove “Map Network Drive” and “Disconnect Network Drive” = Enabled**
    1. User Configuration > Policies > Administrative Templates > Control Panel > Printers > **Prevent addition of printers = Enabled**

### Part 5: Link the *User - Employee Lockdown* GPO to Accounting

Now that we have our GPO with all the restrictions added, it's time to apply it.

The best method is to apply it at the *HQ\Users\Accounting* OU so **any** accounting employee will inherit this GPO and its restrictions, including Bob.

Much better than applying it to 50 different employees one at a time!

1. Back in the *Group Policy Management* application, right-click on: **HQ\Users\Accounting**
1. Click **Link an existing GPO**
1. Select **User - Employee Lockdown** from the list and click **OK**.
1. You should now see it in the list in the main *Linked Group Policy Objects* pane for *Accounting*. If you don't, double-check your work and ask for help before proceeding.

### Part 6: Verify Bob's Restrictions

We should check that our restrictions have worked.

1. Logout of your account on *srv1*.
1. Log on to Bob's account: **bob.smith**
    1. You will be asked to reset the password on first login. This is normal. Pick something complex but basic and **write it down in your Lab Logbook**.
1. Once logged in, try to open up **Settings**. Does it let you?
1. If not, nicely done! Move on to the next part of the lab.

This is a good example of applying settings to a user account. No matter what HQ workstation Bob logs into, these restrictions will *follow him*.

  > **Note:** Normally, we'd use a workstation to test Bob's account, not a server. We're using *srv1* here to conserve lab resources. We'll test with *laptop1* towards the end of this lab.

## Investigation 3: Enzo Matrix - IT Helpdesk

We're going to create a user that provides Level 1 IT support to employees in the rest of the company.

Unlike Bob, who we only needed to restrict what access he had to computers he logged into directly, with Enzo, we have to grant him *specific* admin privileges in Active Directory so he can manage certain things.

A Level 1 Helpdesk employee always has the same terrible job: Reset passwords for employees who have forgotten their own.

This requires some extra steps beyond what we did with Bob from Accounting.

Let's set that up.

### Part 1: Create GPO, *User - IT Environment*

The previous GPO we created is far too restrictive for an IT employee. They can be trusted to make changes to their own system.

**Remember, Group Policy Objects (GPOs) are about modifying *default behaviour*.** Think about when you log into a normal Windows machine you own. You can access Control Panel, right? So, to allow it here, all we need to do is **not** disable it. The default is to allow access. Same with the other restrictions we placed on the previous GPO.

Instead of adding restrictions, we're going to add some shortcuts for common AD management tools to the IT users' desktops.

1. Open the **Group Policy Management** and navigate to:  **Forest: yourSenecaUsername.com > Domains > yourSenecaUsername.com > Group Policy Objects**
1. Create a new GPO here called: **User - IT Environment**
1. **User Configuration > Preferences > Windows Settings > Shortcuts > *Right-Click* > New > Shortcut**
1. Add the following shortcuts:
    1. Server Manager
        1. **Name:** *Server Manager*
        1. **Location:** *Desktop*
        1. **Target Path:** *%SystemRoot%\System32\ServerManager.exe*
        1. Keep all other fields as defaults.
    1. Active Directory Users and Computers
        1. **Name:** *Active Directory Users and Computers*
        1. **Location:** *Desktop*
        1. **Target Path:** *%SystemRoot%\System32\dsa.msc*
        1. Keep all other fields as defaults.
    1. Group Policy Management
        1. **Name:** *Group Policy Management*
        1. **Location:** *Desktop*
        1. **Target Path:** *%SystemRoot%\System32\gpmc.msc*
        1. Keep all other fields as defaults.

### Part 2: Create User - Enzo Matrix

Enzo is our **Level 1 Helpdesk** employee. He needs an IT-friendly desktop (shortcuts, fewer restrictions than Bob) and, soon, the ability to reset passwords in a specific department.

We’ll set him up now so the GPO and group wiring we do next has a real user to affect.

1. Create a new user in the following location: **HQ\Users\IT\Helpdesk**
1. See the following values:  
    1. **First name:** *Enzo*
    1. **Last name:** *Matrix*
    1. **User login name:** *enzo.matrix*
1. Fill out the following in the next page:
    1. **Password:** *same strong password as your other accounts*
    1. **Confirm password:** *same strong password as your other accounts*
    1. Check/uncheck the following:  
        1. **User must change password at next logon:** *Checked*
        1. **User cannot change password:** *Unchecked*
        1. **Password never expires:** *Unchecked*
        1. **Account is disabled:** *Unchecked*

### Part 3: Link the *User - IT Environment* GPO to IT

Now that we have our GPO with all our additions included, it's time to apply it. We'll apply it to the *IT* OU, since it should apply to everyone there (all IT employees).

1. Back in the *Group Policy Management* application, right-click on: **HQ\Users\IT**
1. Click **Link an existing GPO**
1. Select **User - IT Environment** from the list and click **OK**.
1. You should now see it in the list in the main *Linked Group Policy Objects* pane for *IT*. If you don't, double-check your work and ask for help before proceeding.

### Part 4: Global Groups

This is where we get into personal desktop environment settings VS Active Directory admin settings. GPOs are for personal desktop environments that a user sees when they login.

By contrast, we use Global Groups+Role Groups to grant certain AD admin tasks to groups of users.

For reference:

1. **GPO+User OU** = Desktop settings and restrictions on the local machine a user logs into.
1. **Global Groups+Role Groups** = Combinations of AD administrative tasks assigned to groups of users. This is to allow certain users to change Active Directory settings and configuration, not local computers.

A **Global Group *(GG)*** is, essentially, an *identity group*. We use Global Groups to add users together, and then apply a role to the group. Each user in that group then gets that role applied. (We do the roles part a bit later.)

Let's create an AD admin-type group (Global Group, GG) for IT Level 1. Remember, this is just a group of users.

1. In ADUC, navigate to: **HQ**
1. Using the **New > Organizational Unit** function, ***add*** the following to your existing hierarchy:  

    > ```
    > HQ  
    > └─ Groups 
    >    └─ Global
    > ```

1. In *HQ\Groups\Global*, create a new **Group** object with the following settings:
    1. **Group name:** *GG_IT_L1*
    1. **Group scope:** *Global*
    1. **Group type:** *Security*

### Part 5: Role Groups

In contrast to Global Groups, a **Role Group *(RG)*** is a collection of AD admin privileges under a single profile. You can have many RGs, each with different sets of AD admin privileges.

Let's create our first RG and then add a single AD admin privilege to it: Reset passwords.

1. In ADUC, navigate to: **HQ**
1. Using the **New > Organizational Unit** function, ***add*** the following to your existing hierarchy:  

    > ```
    > HQ  
    > └─ Groups 
    >    └─ Roles
    > ```

1. In *HQ\Groups\Roles*, create a new **Group** object with the following settings:
    1. **Group name:** *RG_PasswordReset*
    1. **Group scope:** *Domain Local*
    1. **Group type:** *Security*

1. In *HQ\Users\Accounting*, right-click and select: **Delegate control...**
1. In the *Delegation of Control Wizard*'s > *Users or Groups* page, add: **RG_PasswordReset**
1. In *Delegation of Control Wizard* > *Tasks to Delegate*, select: **Reset user passwords and force password change at next logon**
1. Review your settings on the last page, then click **Finish**.

### Part 6: Wire Global Group to Role Group

We'll now assign our new RG from *Part 5* to our Global Group **GG_IT_L1**. This means that anyone who's a part of *GG_IT_L1* will get the AD admin privilege assigned in *RG_PasswordReset*.

1. In ADUC, navigate to: **HQ\Groups\Roles**
1. Double-click on **RG_PasswordReset** to open its Properties.
1. In the *Properties* window, go to the **Members** tab and click on: **Add...**
1. In the *Select Users, Contacts, Computers, Service Accounts, or Groups* window, type: **GG_IT_L1**
1. Click **OK** and verify it shows up in the *Members* tab.

The Global Group of users *GG_IT_L1* now has all the AD admin privileges given to Role Group *RG_PasswordReset*.

### Part 7: Add Enzo to GG IT Helpdesk Level 1

And now we put it all together. We add Enzo to the Global Group *GG_IT_L1*. Once we do that, Enzo will be able to reset passwords for all users inside *HQ\Users\Accounting*.

1. In ADUC, navigate to: **HQ\Groups\Global**
1. Double-click on **GG_IT_L1** to open its Properties.
1. In the *Properties* window, go to the **Members** tab and click on: **Add...**
1. In the *Select Users, Contacts, Computers, Service Accounts, or Groups* window, type: **Enzo Matrix**
1. Click **OK** and verify it shows up in the *Members* tab.

That's it! If we ever want to add more IT employees to Helpdesk Level 1, we'd just repeat Part 7 with the new user. The rest is already setup.

### Part 8: Verify Enzo's Access and Environment

We should check that our new Enzo account works, has our user GPO applied (those desktop shortcuts), and our Helpdesk Level 1 admin access applied.

1. Logout of your account on *srv1*.
1. Log on to Enzo's account: **enzo.matrix**
    1. You will be asked to reset the password on first login. This is normal. Pick something complex but basic and **write it down in your Lab Logbook**.
1. Once logged in, try to open up **Settings**. Does it let you? It should, this time.
1. Does the desktop contain the three shortcuts we configured:
    1. **Server Manager**
    1. **Active Directory Users and Computers**
    1. **Group Policy Management**
1. Open ADUC and go to: **HQ\Users\Accounting**
1. Right-click on **Bob Smith** and select *Disable Account*. It doesn't work, does it? It shouldn't; Enzo doesn't have permissions to do that.
1. Right-click on **Bob Smith** and select *Reset Password*.
1. Reset Bob's password. Can you?
1. If so, move on to the next part of the lab.

## Investigation 4: Dot Matrix - IT Helpdesk L2

We're going to create a user that provides **Level 2 IT support** to employees in the rest of the company.

A Level 2 Helpdesk employee has their own special AD admin privileges, along with those in Level 1.

In theory, a Level 2 Helpdesk employee shouldn't be contacted for password resets, but she may be asked to do that after completing some L2 tasks for an employee. It happens.

Fortunately, most of the work here is already done. All we need to do is create a new user, and the L2 GG and RGs.

### Part 1: Global Group - GG_IT_L2

Let's create an AD admin-type group (Global Group, GG) for IT Level 2. Remember, this is just a group of users.

1. In ADUC, navigate to: **HQ\Groups\Global** and create a new **Group** object with the following settings:
    1. **Group name:** *GG_IT_L2*
    1. **Group scope:** *Global*
    1. **Group type:** *Security*

### Part 2: Role Group - RG_OUAdmin_AllDepts

Remember, a **Role Group *(RG)*** is a collection of AD admin privileges under a single profile.

Let's create an RG that allows L2 staff to create, modify, move, and delete users in other departments.

As an example, if Bob from Accounting gets reassigned to a different department, a Level 2 specialist like Dot would be the one to move him from one department to the next in the AD structure.

Or disable his account if he gets fired. (*He is terrible, after all.*)

1. In ADUC, navigate to: **HQ\Groups\Roles** and create a new **Group** object with the following settings:
    1. **Group name:** *RG_OUAdmin_AllDepts*
    1. **Group scope:** *Domain Local*
    1. **Group type:** *Security*

1. In *HQ\Users\Accounting*, right-click and select: **Delegate control...**
1. In the *Delegation of Control Wizard*'s > *Users or Groups* page, add: **RG_OUAdmin_AllDepts**
1. In *Delegation of Control Wizard* > *Tasks to Delegate*, select: **Create, delete, and manage user accounts**
1. Review your settings on the last page, then click **Finish**.

### Part 3: Wire Global Group to Role Group

We'll now assign our new RG from *Part 2* to our Global Group **GG_IT_L2**. This means that anyone who's a part of *GG_IT_L2* will get the AD admin privilege assigned in *RG_OUAdmin_AllDepts*.

1. In ADUC, navigate to: **HQ\Groups\Roles**
1. Double-click on **RG_OUAdmin_AllDepts** to open its Properties.
1. In the *Properties* window, go to the **Members** tab and click on: **Add...**
1. In the *Select Users, Contacts, Computers, Service Accounts, or Groups* window, type: **GG_IT_L2**
1. Click **OK** and verify it shows up in the *Members* tab.

The Global Group of users *GG_IT_L2* now has all the AD admin privileges given to Role Group *RG_OUAdmin_AllDepts*.

### Part 4: Create User - Dot Matrix

Time to create our L2 user, Dot Matrix.

1. Create a new user in the following location: **HQ\Users\IT\Helpdesk**
1. Sse the following values:  
    1. **First name:** *Dot*
    1. **Last name:** *Matrix*
    1. **User login name:** *dot.matrix*
1. Fill out the following in the next page:
    1. **Password:** *same strong password as your other accounts*
    1. **Confirm password:** *same strong password as your other accounts*
    1. Check/uncheck the following:  
        1. **User must change password at next logon:** *Checked*
        1. **User cannot change password:** *Unchecked*
        1. **Password never expires:** *Unchecked*
        1. **Account is disabled:** *Unchecked*

### Part 5: Add Dot to GG IT Helpdesk Level 1 *and* Level 2

As mentioned, a Level 2 Helpdesk staff should have L2 AD admin privileges *and* L1 AD admin privileges. **Let's add both.**

1. In ADUC, navigate to: **HQ\Groups\Global**
1. Double-click on **GG_IT_L1** to open its Properties.
1. In the *Properties* window, go to the **Members** tab and click on: **Add...**
1. In the *Select Users, Contacts, Computers, Service Accounts, or Groups* window, type: **Dot Matrix**
1. Click **OK** and verify it shows up in the *Members* tab.
1. Go back to ADUC and navigate to: **HQ\Groups\Global**
1. Double-click on **GG_IT_L2** to open its Properties.
1. In the *Properties* window, go to the **Members** tab and click on: **Add...**
1. In the *Select Users, Contacts, Computers, Service Accounts, or Groups* window, type: **Dot Matrix**
1. Click **OK** and verify it shows up in the *Members* tab.

That's it! Dot has now been granted the AD admin privileges for Level 1 ***and*** Level 2.

### Part 6: Verify Dot's Access and Environment

We should check that our new Dot account works, has our user GPO applied (those desktop shortcuts), and our Helpdesk Level 1/Level 2 admin access applied.

  > **Note:** We didn't have to do anything with GPOs this time, as we applied our IT GPO to the IT folder in Users earlier. So, when Dot was created there, she inherited that GPO. **Cool, right?**

1. Logout of your account on *srv1*.
1. Log on to Dot's account: **dot.matrix**
    1. You will be asked to reset the password on first login. This is normal. Pick something complex but basic and **write it down in your Lab Logbook**.
1. Once logged in, try to open up **Settings**. Does it let you? It should, this time.
1. Does the desktop contain the three shortcuts we configured:
    1. **Server Manager**
    1. **Active Directory Users and Computers**
    1. **Group Policy Management**
1. Open ADUC and go to: **HQ\Users\Accounting**
1. Right-click on **Bob Smith** and select *Disable Account*. It works now, doesn't it? It should. Enzo didn't have permissions to do that, but Dot does.
1. Right-click and Enable the Bob account again so we can keep using it.
1. Verify your L1 permissions: Right-click on **Bob Smith** and select *Reset Password*.
1. Don't reset his password again. Opening the window is enough
1. If all these work, move on to the next part of the lab.

## Investigation 5: Computers - OUs and GPOs

In this investigation, we'll create computer-based OUs and GPOs.

Until now, we've been working with User OUs and GPOs. Complicated as they may get, at the end of the day, these are all about settings and permissions that **follow a user**.

Computer OUs, by contrast, are applied at the computer level regardless of who's logged on.

An easy example: Network settings, like configuring an IP address, are applied at the computer level, not the user level. (Can you imagine if network settings changed every time a different user logged in?)

The following investigation allows us to organize the computers we have joined to our Active Directory domain and what settings to apply to each.

### Part 1: First OU Scaffolding Setup

1. On *srv1*, open **Active Directory Users and Computers** (ADUC).
1. Open your domain name from the list. (Example: *cjohnson30.com*)
1. Open the **HQ** OU as before.
1. Inside it, right-click **HQ** and select: **New > Organizational Unit**
1. Name this new OU: **Computers**
1. Using the same **New > Organizational Unit** function, ***add*** the following to your existing HQ\Computers hierarchy:  

    > ```
    > HQ  
    > └─ Computers 
    >    └─ Workstations  
    >       ├─ Accounting
    >       └─ IT
    >    └─ Laptops 
    >       ├─ Accounting
    >       └─ IT
    >    └─ Servers 
    >       └─ Members
    > ```

1. Verify your work. Does your HQ object hold the above items in the correct levels?
1. If so, move on to the next part. If not, fix your mistakes or ask for help.

### Part 2: Move Computers into the Correct OUs

1. In ADUC, click the built-in Computers container (CN=Computers) and your domain root to find machine accounts.
1. Move (drag and drop) to the new OUs:
    1. *srv1* > **HQ\Computers\Servers\Members**
    1. *laptop1* > **HQ\Computers\Laptops\Accounting**

    > **Note:** Do not move *srv2* and *srv3*! Keep them where they are.

1. Verify your work. Are your *srv1* and *laptop1* computers in the correct OUs?
1. If so, move on to the next part. If not, fix your mistakes or ask for help.

> **Why not move *srv2* and *srv3*?**  
> Domain Controllers live in a special **Domain Controllers** OU with their own built-in GPO (`Default Domain Controllers Policy`).  
>
> In real environments, DCs are usually managed separately from normal servers, so we’re leaving them where they are.

### Part 3: Create Computer Baseline GPOs

Now that we have our Computers OUs, we’ll create **Computer-scope** GPOs that can apply to devices in those OUs.

We’ll make two simple baseline GPOs:

* **Computer – Workstations Baseline**
* **Computer – Laptops Baseline**

In a real company, these would eventually diverge (different sleep policies, USB rules, etc.). For this lab, we’ll give them the same simple setting so you can *see* a device-level policy in action on `laptop1`.

We’ll add a standard **logon banner**. This is a message that appears before anyone logs in. Because it’s a **Computer** setting, it will apply to *any* user who signs in on that device. (In this case, before anyone logs in at all, too.)

1. On *srv1*, open **Group Policy Management**.
2. In the left pane, navigate to:  
   **Forest: yourSenecaUsername.com > Domains > yourSenecaUsername.com > Group Policy Objects**
3. Right-click **Group Policy Objects** > **New…**
4. Create a new GPO:
   1. **Name:** `Computer – Workstations Baseline`
5. Repeat **Step 3–4** to create a second GPO:
   1. **Name:** `Computer – Laptops Baseline`

Now we’ll edit **both** GPOs to add the same logon message.

1. Right-click **Computer – Workstations Baseline** > **Edit…**
2. In the *Group Policy Management Editor* window, go to:

   **Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options**

3. In the right pane, find the following two policies:
   1. **Interactive logon: Message title for users attempting to log on**
   2. **Interactive logon: Message text for users attempting to log on**

4. Double-click **Interactive logon: Message title for users attempting to log on**:
   1. Set **Security Setting** to something like:  
      `OSM620 HQ – Authorized Use Only`
   2. Click **OK**.
5. Double-click **Interactive logon: Message text for users attempting to log on**:
    1. Set **Security Setting** to something like:  

       `This system is for authorized use only. Activity may be monitored. By signing in, you agree to comply with company policy.`

    2. Click **OK**.

6. Close the *Group Policy Management Editor* window.
7. Repeat **Steps 6–11**, but this time for **Computer – Laptops Baseline** so that both GPOs have the same logon banner configured.

> **Why a logon banner?**  
> It’s a **Computer-scope** setting that shows up *before* anyone signs in.  
> That makes it a perfect visual example of “policy sticks to the device, not the person.”

### Part 4: Link Computer GPOs to the Computers OUs

Now we’ll link our new Computer GPOs at the correct places in the **HQ\Computers** tree.

1. In **Group Policy Management**, expand:  
   **Forest: yourSenecaUsername.com > Domains > yourSenecaUsername.com > yourSenecaUsername.com > HQ > Computers**
2. Right-click **Workstations** > **Link an Existing GPO…**
3. Choose **Computer – Workstations Baseline** > **OK**.
4. Right-click **Laptops** > **Link an Existing GPO…**
5. Choose **Computer – Laptops Baseline** > **OK**.

You should now see:

* `Computer – Workstations Baseline` linked at **HQ\Computers\Workstations**
* `Computer – Laptops Baseline` linked at **HQ\Computers\Laptops**

Remember from **Part 2**:

* `srv1` is in **HQ\Computers\Servers\Members**
* `laptop1` is in **HQ\Computers\Laptops\Accounting**

So right now, **only `laptop1`** will actually inherit **Computer – Laptops Baseline**.

### Part 5: Verify Laptop Baseline on *laptop1*

Let’s prove that this GPO is really a **Computer-scope** policy that “sticks” to `laptop1`.

1. On *laptop1*, sign in with your normal domain account (*yourSenecaUsername*).
1. Open **Command Prompt** using *Run as Adminstrator*.
1. Run: `gpupdate /force`  
   This forces the new Computer policy to apply right away instead of waiting.
1. When it completes, confirm your work by running the following: `gpresult /r /scope computer`
1. You should see something similar to the following:

```powershell
C:\Windows\System32>gpresult /r /scope computer

Microsoft (R) Windows (R) Operating System Group Policy Result tool v2.0
© Microsoft Corporation. All rights reserved.

Created on 2025-11-22 at 4:47:29 PM


RSOP data for  on LAPTOP1 : Logging Mode
-----------------------------------------

OS Configuration:            Member Workstation
OS Version:                  10.0.26200
Site Name:                   Default-First-Site-Name
Roaming Profile:
Local Profile:
Connected over a slow link?: No


COMPUTER SETTINGS
------------------
    CN=LAPTOP1,OU=IT,OU=Laptops,OU=Computers,OU=HQ,DC=cjohnson30,DC=com
    Last time Group Policy was applied: 2025-11-22 at 4:37:30 PM
    Group Policy was applied from:      srv2.cjohnson30.com
    Group Policy slow link threshold:   500 kbps
    Domain Name:                        CJOHNSON30
    Domain Type:                        Windows 2008 or later

    Applied Group Policy Objects
    -----------------------------
        Computer - Laptop Baseline
        Default Domain Policy

    The following GPOs were not applied because they were filtered out
    -------------------------------------------------------------------
        Local Group Policy
            Filtering:  Not Applied (Empty)

    The computer is a part of the following security groups
    -------------------------------------------------------
        BUILTIN\Administrators
        Everyone
        BUILTIN\Users
        NT AUTHORITY\NETWORK
        NT AUTHORITY\Authenticated Users
        This Organization
        LAPTOP1$
        Domain Computers
        Authentication authority asserted identity
        System Mandatory Level
```

> **Notice the line: *Computer - Laptop Baseline***

1. Once confirmed, restart `laptop1`.
1. After the restart, look carefully at the **logon screen**:
   1. Do you see the **title** and **message text** you configured in Part 3?
1. Sign in as a different domain user (for example, Enzo or Dot, once their accounts are set up).
1. Confirm that the **same logon banner** appears for them as well.

If the banner shows up before *any* user signs in, you’ve just seen a **Computer-scope** GPO in action:

* It follows the **device (`laptop1`)**, not the individual user.
* Every domain user on that device must see and acknowledge the message.

If you **don’t** see the message:

1. Double-check that:
   1. `laptop1` is in **HQ\Computers\Laptops\Accounting** in ADUC.
   2. **Computer – Laptops Baseline** is linked at **HQ\Computers\Laptops**.
2. Run `gpupdate /force` again on *laptop1* and restart one more time.
3. If it still doesn’t work, ask your instructor for help.

> **Note:** In a real environment, admins can trigger a *Group Policy Update* remotely from GPMC on the domain or OU, but for this lab you’ll run `gpupdate /force` locally on *laptop1*.

## Lab 7 Sign-Off

**It’s essential to complete Lab 7 correctly.** Lab 8 assumes you have your users, groups, and computers properly organized in Active Directory.

When you finish Lab 7, ask your instructor for a sign-off.

### Sign-Off Checklist

Please have the following on screen and ready to show. You will need to power on the following VMs:

* srv1
* srv2
* laptop1

#### On srv1

**Using your `firstname.lastname` Domain Administrator account**, show the following in *Active Directory Users and Computers*.

HQ\Users OU structure

1. In Active Directory Users and Computers, show:
 1. Bob Smith in HQ > Users > Accountants
 2. Enzo Matrix in HQ > Users > IT > Helpdesk
 3. Dot Matrix in HQ > Users > IT > SysAdmins

HQ\Computers OU structure

1. In HQ > Computers, show:
 1. srv1 in Servers > Members
 2. laptop1 in Laptops > Accounting
 3. User and Computer GPOs
1. In Group Policy Management, show that:
 1. Your Employee user GPO (for example, User – Employee Lockdown) is linked at HQ > Users > Accountants.
 2. Your IT user GPO (for example, User – IT Environment) is linked at HQ > Users > IT.
 3. Your Laptop computer GPO (for example, Computer – Laptops Baseline) is linked at HQ > Computers > Laptops.

If all of the above are present, your directory structure and GPO links are considered correct for this lab.

Test your user accounts:

1. Log into Enzo Matrix.
2. Log into Dot Matrix.

#### On laptop1

1. Log into Bob Smith.
1. Log into your `firstname.lastname` Domain Administrator account.

---
id: lab8
title: Lab 8 - Network File Shares
sidebar_position: 8
description: Lab 8 - Network File Shares
--- 

# Lab 8 - Network File Shares

## Lab Preparation

### Purpose of Lab 8

In this lab, you’ll turn your domain into something more useful for actual humans by giving departments a central place to store their files and making those locations show up automatically when users sign in.

You will:

- Use **SMB shares** on a Windows Server to provide **department file storage**.
- Control access with **Active Directory Global Groups**, not individual users.
- Use **Group Policy Preferences** to automatically map **department drives** and create **desktop shortcuts** for the right people.

You’ll build a very typical small-organization pattern that still appears in the real world:

 **Two department file shares + department drive letters, driven by AD groups.**

> **Important:** In production, you normally **do not** host file shares on a domain controller.  
>
> We are using **SRV2 (DC1)** as a temporary file server in this lab to save RAM/VMs. In a production environment, you would use a completely separate File Server.

### Objectives

By the end of this lab, you will be able to:

- Create simple **department security groups** (`GG_Dept_*`) and add users to them.
- Publish **SMB file shares** from a Windows Server for Accounting and IT.
- Configure **NTFS permissions** so only the correct department can access its folder.
- Use **Group Policy Preferences (GPP)** to:
  - Automatically map **department network drives** (`M:` / `X:`).
  - Create **desktop shortcuts** pointing at those drives.
- Verify **end-user experience** from a domain-joined client (`laptop1`).

### Minimum Progression Requirements

Before beginning, you must have:

1. Successfully completed and **Lab 6** and **Lab 7**.
2. Your **external SSD** (or personal computer) with your OSM620 VMs.
3. Your **OSM620 Lab Logbook**.
4. Optional, but recommended: caffeine delivery system.

### Pre-Flight Check

Power on the following VMs:

1. `router`
2. `srv1`
3. `srv2` (DC1)

Unless otherwise stated, perform all administrative tasks from **srv1** using your **personal domain admin account** created in Lab 7.

## Before You Begin

Your environment from Lab 6 and Lab 7 should already have:

- A domain: `yourSenecaUsername.com`
- Two domain controllers:
  - `srv2` (DC1)
  - `srv3` (DC2)
- An admin workstation / management server: `srv1`
- A domain-joined Windows 11 client: `laptop1`
- OUs and users similar to:
  - `HQ\Users\Accounting`
    - `Bob Smith` (Accounting)
  - `HQ\Users\IT`
    - `Enzo Matrix`, `Dot Matrix` (IT)
- A `HQ\Groups\Global` OU where your **Global Groups (GG_)** live.

## Investigation 1: Department Global Groups

Before we build file shares, we need **AD groups** that represent who belongs to each department. We’ll use these groups for both **folder permissions** and **GPO targeting**.

### Part 1: Create Department Global Groups

1. On **srv1**, sign in with your **personal domain admin** account. (Not *Administrator*!)
1. Open **Active Directory Users and Computers**.
1. In the left pane, expand your domain and browse to:

   `HQ \ Groups \ Global`

1. Right-click the **Global** OU and select **New > Group**.
1. Create the following group:

   - **Group name:** `GG_Dept_Accounting`  
   - **Group scope:** `Global`  
   - **Group type:** `Security`

1. Repeat the process to create:

   - **Group name:** `GG_Dept_IT`  
   - **Group scope:** `Global`
   - **Group type:** `Security`

1. Under **HQ\Groups\Global**, you should now see:

   - `GG_Dept_Accounting`
   - `GG_Dept_IT`

### Part 2: Add Users to the Department Groups

We’ll add the existing users you created in previous labs.

1. In **HQ\Groups\Global**, double-click **`GG_Dept_Accounting`**.
2. Go to the **Members** tab and click **Add…**
3. Add your Accounting user:

   - `bob.smith`

4. Click **OK**, then **OK** again.
5. Double-click **`GG_Dept_IT`**.
6. Go to the **Members** tab and click **Add…**
7. Add your IT users:

   - `enzo.matrix`  
   - `dot.matrix`

8. Click **OK**, then **OK** again.

At this point:

- **Accounting staff** are in `GG_Dept_Accounting`.
- **IT staff** are in `GG_Dept_IT`.

> **Remember:**  
> OUs are **folders** for organization and GPO targeting.
>
> **Groups** are how you express “who can access what.”

## Investigation 2: Department SMB Shares on DC1 (SRV2)

In this investigation, you’ll publish two SMB shares on **SRV2 (DC1)** and secure them with the department groups created in Investigation 1.

You will create:

- `C:\CorpData\Accounting` > shared as `\\srv2\Accounting`
- `C:\CorpData\IT` > shared as `\\srv2\IT`

We’ll use the **New Share Wizard** from **srv1**. The wizard will create **both** the folder and the share.

> **Reminder**  
> In a real environment, you would normally do this on a dedicated file server, not a DC.
>
> We’re using SRV2 **only for this lab** to save resources.

### Part 1: Create the Accounting Share

1. On **srv1**, open **Server Manager**.
2. In the left pane, click **File and Storage Services**.
3. Click **Shares**.
4. In the **TASKS** dropdown (top-right of the Shares pane), click **New Share…**
5. In the **New Share Wizard**:

   1. **Select the profile:**  
      - Choose **SMB Share – Quick** > click **Next**.

   2. **Select the server:**  
      - Select **srv2** as the server > click **Next**.

   3. **Select the folder:**  
      - Choose **Type a custom path**.
      - Enter: `C:\CorpData\Accounting`
      - Click **Next**.
      - If prompted that the folder doesn’t exist, allow the wizard to **create it**.

   4. **Specify share name:**  
      - Share name: `Accounting`
      - Confirm the remote path shows as: `\\srv2\Accounting`
      - Click **Next**.

   5. **Configure other settings:**  
      - Leave the defaults > click **Next**.

   6. **Specify permissions:**
      - Click **Customize permissions…**
      - In the **Security** tab, click **Advanced**.
      - Click **Disable inheritance**.
      - When prompted, choose:
        - **Convert inherited permissions into explicit permissions on this object.**
      - Back in the permission entries list:
        - **Keep** `SYSTEM` and `Administrators` with **Full control**.
        - Remove **Users**, **Authenticated Users**, or any other broad “everyone” entries if present.
      - Click **Add…** to create a new permission entry:
        - **Principal:** `GG_Dept_Accounting`
        - **Type:** Allow
        - **Applies to:** This folder, subfolders and files
        - **Basic permissions:** `Modify`
      - Click **OK** until you return to the wizard.

   7. Click **Next**, then **Create**, then **Close**.

You now have:

- Folder: `C:\CorpData\Accounting` on **srv2**
- Share: `\\srv2\Accounting`
- Permissions: only `GG_Dept_Accounting` (plus Administrators/SYSTEM) can modify the contents.

### Part 2: Create the IT Share

Repeat the process for the IT department.

1. On **srv1**, in **Server Manager  File and Storage Services > Shares**, click **TASKS > New Share…**
2. In the **New Share Wizard**:

   1. **Profile:**  
      - **SMB Share – Quick** > **Next**.

   2. **Server:**  
      - **srv2** > **Next**.

   3. **Folder:**  
      - Select **Type a custom path**.
      - Enter: `C:\CorpData\IT`
      - Click **Next** and allow the wizard to create the folder if prompted.

   4. **Share name:**  
      - `IT`
      - Confirm remote path: `\\srv2\IT`
      - **Next**.

   5. **Other settings:**  
      - Leave defaults > **Next**.

   6. **Permissions:**
      - Click **Customize permissions…**
      - Click **Advanced > Disable inheritance**.
      - Choose **Convert inherited permissions into explicit permissions**.
      - Keep `SYSTEM` and `Administrators` with **Full control**.
      - Remove `Users` / `Authenticated Users` if present.
      - Click **Add…**:
        - **Principal:** `GG_Dept_IT`
        - **Type:** Allow
        - **Applies to:** This folder, subfolders and files
        - **Basic permissions:** `Modify`
      - Click **OK** until you return to the wizard.

   7. Click **Next**, then **Create**, then **Close**.

You now have:

- `\\srv2\Accounting` secured for Accounting.
- `\\srv2\IT` secured for IT.

### Part 3: Quick Sanity Check from srv1

1. On **srv1**, open **File Explorer**.
2. In the address bar, type: `\\srv2` and press **Enter**.
3. Confirm you see these shares:

   - `Accounting`
   - `IT`

4. As your domain admin account, verify that you can open both shares and create a test text file in each folder. (`test.txt`)
5. Delete the test files when you’re done.

We’ll test with real users in *Investigation 3*.

## Investigation 3: Department Drive Mappings with Group Policy

Now that the server side is ready, you’ll configure a **User-scope GPO** to automatically:

- Map an **H:** drive to `\\srv2\Accounting` for Accounting staff.
- Map an **I:** drive to `\\srv2\IT` for IT staff.
- Create matching **desktop shortcuts** pointing at those drive letters.

This uses **Group Policy Preferences** and **Item-level targeting** by security group.

### Part 1: Create and Link the GPO

1. On **srv1**, open **Group Policy Management**.
2. In the left pane, expand your domain until you locate the OU where your **user accounts** live, for example:

   `yourSenecaUsername.com \ HQ \ Users`

   (Use the OU that contains your department user OUs created in Lab 7, such as `HQ\Users\Accounting` and `HQ\Users\IT`.)

3. Right-click the **Users** OU and select:

   **Create a GPO in this domain, and Link it here…**

4. Name the GPO:

   **HQ – Department Network Drives**

5. Click **OK**.

The GPO is now linked where it can apply to your users.

### Part 2: Map M: for Accounting

1. In **Group Policy Management**, right-click:

   **HQ – Department Network Drives**

   and choose **Edit…**

2. In the GPO editor, browse to:

   `User Configuration > Preferences > Windows Settings > Drive Maps`

3. Right-click in the right pane and select **New > Mapped Drive**.
4. Configure the **General** tab as follows:

   - **Action:** `Create`
   - **Location:** `\\srv2\Accounting`
   - **Reconnect:** ✔ (checked)
   - **Label as:** `Accounting Department Data`
   - **Drive letter:** `M:`

5. At the bottom of the **General** tab, click **Common**.
6. Check **Item-level targeting** and click **Targeting…**
7. In the **Targeting Editor**:

   1. Click **New Item > Security Group**.
   2. Click **…** and choose: `GG_Dept_Accounting`
   3. Confirm it reads:  
      `User is a member of the security group GG_Dept_Accounting`
   4. Click **OK**.

8. Click **OK** again to close the Mapped Drive properties.

Result:

- Any user **in `GG_Dept_Accounting`** will get **M:** mapped to `\\srv2\Accounting` when they sign in.
- Other users will not see this mapped drive.

### Part 3: Map X: for IT

1. Still under **Drive Maps**, right-click in the right pane > **New > Mapped Drive**.
2. Configure the **General** tab:

   - **Action:** `Create`
   - **Location:** `\\srv2\IT`
   - **Reconnect:** ✔
   - **Label as:** `IT Department Data`
   - **Drive letter:** `X:`

3. Click **Common >** check **Item-level targeting > Targeting…**
4. Add a **Security Group** targeting item:

   - Group: `GG_Dept_IT`
   - Condition: `User is a member of the security group`

5. Click **OK**, then **OK** again.

Result:

- Members of **`GG_Dept_IT`** get drive **X:** mapped to `\\srv2\IT`.

### Part 4: Desktop Shortcuts

To make the user experience even friendlier, you’ll create desktop shortcuts that point to those drive letters.

1. In the same GPO editor, browse to:

   `User Configuration > Preferences > Windows Settings > Shortcuts`

2. Right-click in the right pane > **New > Shortcut**.

#### Accounting Shortcut

1. On the **General** tab:

   - **Action:** `Create`
   - **Name:** `Accounting Files`
   - **Target type:** `File System Object`
   - **Location:** `Desktop`
   - **Target path:** `M:\`
   - Leave other fields at defaults.

2. Click the **Common** tab.
3. Check **Item-level targeting > Targeting…**
4. Add a **Security Group** item targeting `GG_Dept_Accounting`.
5. Click **OK**, then **OK** again.

#### IT Shortcut

1. Right-click again in the Shortcuts pane > **New > Shortcut**.
2. Configure:

   - **Action:** `Create`
   - **Name:** `IT Files`
   - **Target type:** `File System Object`
   - **Location:** `Desktop`
   - **Target path:** `X:\`

3. On the **Common** tab, enable **Item-level targeting** and target the `GG_Dept_IT` group.
4. Click **OK**.

Now:

- Accounting users will see an **H:** drive and an **Accounting Files** desktop icon.
- IT users will see an **I:** drive and an **IT Files** desktop icon.

### Part 5: Test with Accounting User (Bob)

1. On **laptop1**, sign in as your **Accounting** user, `bob.smith`.
2. Open a Command Prompt and run:

   ```cmd
   gpupdate /force
   ```

3. Once `gpupdate` completes, sign out and sign back in as Bob.
4. After logon, open File Explorer and confirm:
   1. **M:** exists and points to `\\srv2\Accounting`.
   1. You can create a second test file (`bob-test.txt`) on M:.
5. On the desktop, confirm:
   1. There is an **Accounting Files** shortcut.
   1. Double-clicking it opens **M:**
6. Confirm:
   1. There is no **X:** drive.
   1. There is no **IT Files** shortcut.

### Part 6: Test with IT User - Enzo Matrix

 1. Sign out Bob from **laptop1**.
 2. Sign in as your IT user: **enzo.matrix**
 3. Run:

     ```cmd
     gpupdate /force
     ```

 4. Sign out and sign back in as Enzo.
 5. Open **File Explore**r and confirm:
    1. **X:** exists and points to `\\srv2\IT`.
    1. You can create a test file (`IT-Test.txt`) on **X:**.
 6. On the desktop, confirm:
    1. There is an **IT Files** shortcut.
    1. Double-clicking it opens **X:**.
 7. Confirm:
    1. There is no **M:** drive.
    1. There is no **Accounting Files** shortcut.

### Part 7: Test with IT User - Dot Matrix

 1. Sign out Enzo from **laptop1**.
 2. Sign in as your IT user: **dot.matrix**
 3. Run:

     ```cmd
     gpupdate /force
     ```

 4. Sign out and sign back in as Dot.
 5. Open **File Explore**r and confirm:
    1. **X:** exists and points to `\\srv2\IT`.
    1. (`IT-Test.txt`) already exists on **X:**.
 6. On the desktop, confirm:
    1. There is an **IT Files** shortcut.
    1. Double-clicking it opens **X:**.
 7. Confirm:
    1. There is no **M:** drive.
    1. There is no **Accounting Files** shortcut.

## Lab 8 Sign-Off

**It’s essential to complete Lab 8 correctly.** Lab 8 assumes you have your users able to access their correct file shares properly.

When you finish Lab 8, ask your instructor for a sign-off.

### Sign-Off Checklist

Please have the following on screen and ready to show. You will need to power on the following VMs:

 * srv1
 * srv2
 * laptop1

To receive sign off for Lab 8, you must demonstrate the following in person to your instructor:

#### Department Groups in AD

In Active Directory Users and Computers on srv1, show:

1. GG_Dept_Accounting exists and contains your Accounting user (e.g., bob.smith).
1. GG_Dept_IT exists and contains your IT user(s) (e.g., enzo.matrix, dot.matrix).

#### Department Shares on SRV2

From srv1, open \\srv2 in File Explorer and show:
1. The Accounting and IT shares are present.
1. Open the Security properties for one of the folders (either C:\CorpData\Accounting or C:\CorpData\IT on srv2) and show:
   1. The matching GG_Dept_* group has Modify permission.
   1. SYSTEM and Administrators still have Full control.

#### Accounting User View (laptop1)

Sign in to laptop1 as your Accounting user and show:
1. Drive M: is mapped and points to \\srv2\Accounting.
1. There is an Accounting Files shortcut on the desktop that opens M:
1. There is no X: drive and no IT Files shortcut.
1. You can create and delete a test file on H:.

#### IT User View (laptop1)

Sign in to laptop1 as your IT user (Enzo or Dot) and show: 

1. Drive X: is mapped and points to \\srv2\IT.
1. There is an IT Files shortcut on the desktop that opens X:
1. There is no M: drive and no Accounting Files shortcut.
1. You can create and delete a test file on X:.

Once you have demonstrated all four items, your instructor will sign off Lab 8.

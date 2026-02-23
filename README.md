# Building a Windows Server 2022 Domain Controller & Windows 10 Client (Screenshots yet to upload)

## Overview
This documentation covers the step-by-step process of provisioning, installing, and configuring a Windows Server 2022 Domain Controller (DC) and a Windows 10 client within a Proxmox Virtual Environment. It includes the configuration of Active Directory (AD DS), DNS, DHCP, and Group Policy Objects (GPO) for automated network drive mapping.

---

## Prerequisites
Before starting, ensure you have the following resources available in your Proxmox ISO storage:
* **Windows Server 2022 ISO** (180-day evaluation available from Microsoft)
* **Windows 10 ISO** * **VirtIO Drivers ISO** (Required for Proxmox Windows virtualization)

> 📸 **Screenshot:** Proxmox storage view listing the three required ISO files.

---

## Part 1: Provisioning the Windows Server 2022 VM

1. Log into your Proxmox web interface and click **Create VM**.
2. **General:** Assign a VM ID  and Name the VM `DC`.
3. **OS:** Select your Windows Server 2022 ISO. Change the Guest OS Type to **Microsoft Windows**.
   > 📸 **Screenshot:** The 'OS' tab in the Proxmox VM creation wizard.
4. **System:** Enable **Add TPM** and **Add EFI Disk**. Select your target storage for both.
5. **Disks:** Change the Bus/Device to **SCSI**. Select your storage pool  and set the Disk size to **64 GB**.
   > 📸 **Screenshot:** The 'Disks' tab highlighting the SCSI selection and 64GB size.
6. **CPU & Memory:** Allocate **2 Cores** and **4096 MB (4GB)** of RAM.
7. **Network:** Assign the interface to your lab bridge (e.g., `vmbr1`) and set the VLAN Tag (e.g., `20`).
   > 📸 **Screenshot:** The 'Network' tab showing the Bridge and VLAN Tag settings.
8. **Add VirtIO Drivers:** Before starting the VM, go to the VM's **Hardware** tab, click **Add > CD/DVD Drive**, and mount the `virtio-win.iso`.
   > 📸 **Screenshot:** The completed VM Hardware list showing both the Windows OS CD/ROM and the VirtIO CD/ROM.

---

## Part 2: Installing Windows Server 2022

1. Start the `DC` VM and open the Console. Press any key to boot from the CD.
2. Click **Install Now**.
3. **Crucial Step:** Select **Windows Server 2022 Standard Evaluation (Desktop Experience)**.
   > 📸 **Screenshot:** The OS selection screen with "Desktop Experience" highlighted. 
4. Accept the licensing terms and choose **Custom: Install Windows only (advanced)**.
5. **Load Storage Drivers:**
   * You will notice no drives are found. Click **Load driver** -> **Browse**.
   * Navigate to the VirtIO CD drive -> `vioscsi` -> `w2k22` -> `amd64`.
   * Click **OK** and select the **Red Hat Virtual SCSI pass-through controller**.
   > 📸 **Screenshot:** The Browse window navigating the VirtIO folder structure.
   > 
   > 📸 **Screenshot:** The "Where do you want to install Windows?" screen successfully showing the 64GB drive.
6. Select the drive, click Next, and allow the installation to complete. Set a secure Administrator password upon reboot.

---

## Part 3: Initial Network & System Configuration

### Allow ICMP (Ping)
1. Open **Windows Defender Firewall with Advanced Security**.
2. Click **Inbound Rules** -> **New Rule...**
3. Select **Custom** -> **All Programs** -> Protocol type: **ICMPv4**.
4. Allow the connection, apply to all profiles, name it `Allow ICMPv4`, and click Finish.
   > 📸 **Screenshot:** The completed Inbound Rule wizard showing the ICMPv4 protocol selection.

### Static IP Assignment
1. Open **Network Connections** (`ncpa.cpl`).
2. Right-click your Ethernet adapter -> **Properties** -> **Internet Protocol Version 4 (TCP/IPv4)**.
3. Configure the static IP details for your VLAN (e.g., IP: `10.10.20.10`, Gateway: `10.10.20.254`, DNS: `8.8.8.8`).
   > 📸 **Screenshot:** The IPv4 Properties window with the filled-out static IP addresses.

### Rename Server
1. Right-click **This PC** -> **Properties** -> **Rename this PC (advanced)**.
2. Change the computer name to `DC` and restart the server.

---

## Part 4: Promoting to a Domain Controller

1. Open **Server Manager** and click **Add roles and features**.
2. Proceed to Server Roles and check the following:
   * **Active Directory Domain Services**
   * **DHCP Server**
   * **DNS Server**
   > 📸 **Screenshot:** The 'Server Roles' selection screen with AD DS, DHCP, and DNS checked.
3. Complete the installation wizard. 
4. Once finished, click the **Flag icon** (Notifications) in Server Manager and select **Promote this server to a domain controller**.
   > 📸 **Screenshot:** The Server Manager dashboard highlighting the promotion link.
5. Select **Add a new forest** and enter your Root domain name (e.g., `jisan.local`).
   > 📸 **Screenshot:** The Deployment Configuration tab showing the new forest name.
6. Enter a Directory Services Restore Mode (DSRM) password. Click Next through the defaults and click **Install**. The server will reboot automatically.

---

## Part 5: Creating Active Directory Users and Groups

1. Open **Active Directory Users and Computers** (`dsa.msc`).
2. Expand your domain (`jisan.local`). Right-click the **Users** container -> **New** -> **Group**. Name it `Shared Folder Access`.
   > 📸 **Screenshot:** The New Object - Group dialog box.
3. Right-click the **Users** container -> **New** -> **User**. Create a standard user (e.g., `jjisan`) and set the password to never expire.
4. Create an admin user (e.g., `jjadmin`).
5. Right-click the admin user -> **Properties** -> **Member Of** tab -> **Add**. Type `Domain Admins` and apply.
   > 📸 **Screenshot:** The user's 'Member Of' properties tab showing Domain Admins.
6. Add both users to the `Shared Folder Access` group.

---

## Part 6: Migrating DHCP Services

*(Note: If DHCP was previously handled by a firewall like pfSense, disable it on the firewall for this VLAN first).*

1. Open the **DHCP** management console from Server Manager tools.
2. Expand your server, right-click **IPv4**, and select **New Scope**.
   > 📸 **Screenshot:** The DHCP console showing the right-click menu for IPv4.
3. Name the scope (e.g., `VLAN 20`).
4. Set the IP range (e.g., Start: `10.10.20.100`, End: `10.10.20.120`) and Subnet mask (`255.255.255.0`).
   > 📸 **Screenshot:** The IP Address Range configuration screen.
5. In the DHCP Options screen, add your Router (Default Gateway) IP (e.g., `10.10.20.254`).
6. Finish the wizard. Go back to Server Manager, click the Flag icon, and select **Complete DHCP configuration** -> **Commit**.

---

## Part 7: Shared Folder & Group Policy (GPO) Mapping

### Create the Share
1. Open File Explorer and create a folder at `C:\shared folder`.
2. Right-click the folder -> **Properties** -> **Sharing** -> **Advanced Sharing**.
3. Check **Share this folder**, click **Permissions**, and grant Everyone **Full Control** (for lab purposes only).
   > 📸 **Screenshot:** The Share Permissions window showing Full Control granted.

### Create the Group Policy
1. Open **Group Policy Management** (`gpmc.msc`).
2. Expand the forest and domain. Right-click your domain (`jisan.local`) and select **Create a GPO in this domain, and Link it here...** Name it `Map Network Drive`.
   > 📸 **Screenshot:** The GPMC tree view showing the newly linked policy.
3. Right-click the new policy and click **Edit**.
4. Navigate to **User Configuration** -> **Preferences** -> **Windows Settings** -> **Drive Maps**.
5. Right-click the empty space -> **New** -> **Mapped Drive**.
6. **General Tab:**
   * Action: **Update**
   * Location: `\\DC\Shared`
   * Label as: `Shared`
   * Drive Letter: Use `G`
   > 📸 **Screenshot:** The New Drive Properties window (General tab).
7. **Common Tab:**
   * Check **Run in logged-on user's security context**.
   * Check **Item-level targeting** and click **Targeting...**
   * Click **New Item** -> **Security Group**. Browse and select the `Shared Folder Access` group.
   > 📸 **Screenshot:** The Targeting Editor window showing the Security Group configuration.
8. Click OK and apply the policy.

---

## Part 8: Building and Joining the Windows 10 Client

1. Create a new Proxmox VM (e.g., `prod-win10`) following the exact same hardware steps as the Windows Server (SCSI, 64GB, VLAN 20, add VirtIO CD-ROM).
2. Start the VM and boot into the Windows 10 setup.
3. Choose **Custom: Install Windows only (advanced)**.
4. Click **Load driver** -> **Browse** -> Navigate to the VirtIO CD -> `vioscsi` -> `w10` -> `amd64`. Load the driver and install Windows 10.
   > 📸 **Screenshot:** The driver selection screen showing the Windows 10 specific SCSI driver.
5. During the out-of-box experience (OOBE), select **Domain join instead** (bottom left corner) to skip the Microsoft Account login.
   > 📸 **Screenshot:** The Windows 10 setup screen highlighting the "Domain join instead" option.
6. Once on the desktop, open Command Prompt and run `ipconfig`. Verify the machine received an IP address from your newly configured DHCP scope (e.g., `10.10.20.100`).
   > 📸 **Screenshot:** Command prompt showing the successful DHCP IP lease.
7. Open **System Properties** (`sysdm.cpl`).
8. Click **Change...**, select **Domain**, and type `jisan.local`. 
9. When prompted, authenticate using the `jjadmin` domain account.
   > 📸 **Screenshot:** The "Welcome to the domain" success prompt.
10. Restart the PC.
11. At the login screen, choose **Other User** and log in with the standard user account created earlier (e.g., `jisan\jjisan`).
12. Open File Explorer and verify that the `G:` drive has mapped automatically via Group Policy.
   > 📸 **Screenshot:** File explorer showing the mapped network drive.

---

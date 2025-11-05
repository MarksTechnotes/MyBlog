---
title: "Running DBCA in GUI Mode on an OCI Linux VM"
date: 2025-11-03
draft: false
description: "Steps to run Oracle Database Configuration Assistant (DBCA) in GUI mode on an OCI Linux VM using GNOME, VNC, and SSH tunneling."
tags: ["Oracle", "DBCA", "OCI", "Linux", "VNC", "SSH", "GNOME"]
categories: ["Database", "Cloud"]
---

When working with Oracle databases on OCI Linux VMs, you may want to run **Database Configuration Assistant (DBCA)** in GUI mode instead of command-line mode.  
This requires a **GNOME desktop environment**, a **VNC session**, and optionally **SSH tunneling** for secure access.

---

### Prerequisites
Before testing DBCA in GUI mode, ensure the following:

- A **GNOME desktop environment** is installed on the OCI VM. GUI applications like DBCA require a desktop environment to display.  
- A **VNC server** (e.g., `tigervnc-server`) is installed and running on the OCI VM.  
- A **VNC client** (such as RealVNC, TightVNC, or TigerVNC Viewer) is installed on your local desktop or laptop.  
- If you’re connecting directly over the network, open port **5901** (for display `:1`) in your OCI security list and VM firewall.  
- If you use **SSH tunneling** to access VNC, you **do not need to open any VNC ports** externally — the tunnel handles secure forwarding.

Example of an SSH tunnel for VNC:

```bash
ssh -L 5901:localhost:5901 opc@<vm_public_ip>
```


---

### Steps

**1. Login as root**  
Start by logging in as the `root` user. You’ll need root privileges to allow the Oracle user access to the GUI environment.

**2. Enable GUI access for Oracle**  
Run the following command:  
```bash
xhost + si:localuser:oracle
```
This grants the `oracle` user permission to connect to the X server (the graphical environment).
Without this step, GUI applications like DBCA cannot display. 

         
**3. Login as Oracle**  
Switch to the `oracle` user: 
 ```bash
 su - oracle 
 ```
This ensures that you’re operating in the correct environment to run DBCA.
 
**4. Set the Display**  
Run: 
```bash 
export DISPLAY=:1 
```
This tells GUI applications to render their output to display `:1`, which corresponds to your local laptop/desktop environment.

**5. Run oraenv**   
Before starting DBCA, run the environment setup command: 
```bash
.oraenv
``` 
This sets up Oracle-related environment variables like `ORACLE_HOME` and `ORACLE_SID`.

**6. Launch DBCA**   
Finally launch the Database Configuration Assistant:
```bash
 dbca
 ``` 
If everything is configured properly, the DBCA GUI window should appear inside your VNC client.

---

### Verification
If DBCA launches successfully, GUI-based configuration is working.  
If it doesn’t display, check:

- That your **VNC connection** or **SSH tunnel** is active and connected to the correct display (`:1`).  
- That your **display settings** (`echo $DISPLAY`) are correct.  
- X11 forwarding or graphical support is properly configured.  
- Oracle environment variables (`ORACLE_HOME`, `ORACLE_SID`) are set correctly.

---

### In summary
Running DBCA in GUI mode on an OCI VM is straightforward once **GNOME, VNC and SSH tunneling** are properly configured.  
With tunneling, you can securely run DBCA without exposing VNC ports to the public network, which is more secure.

 
  

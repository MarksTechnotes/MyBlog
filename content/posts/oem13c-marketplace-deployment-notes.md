---
title: "Deploying OEM 13c Marketplace Image on OCI — Field Notes from a Quick Deployment"
date: 2025-11-09
draft: false
description: "Personal deployment notes on setting up Oracle Enterprise Manager 13c using the Marketplace image on Oracle Cloud Infrastructure."
tags: ["Oracle", "OEM", "OCI", "Cloud", "Marketplace"]
categories: ["Database", "Cloud"]
---

I recently deployed **Oracle Enterprise Manager (OEM) 13c** on **Oracle Cloud Infrastructure (OCI)** using the **OEM Marketplace image**.  
This post captures the key steps and observations from that quick deployment.

---

### **OEM Marketplace Image**
The OEM Marketplace image used was available here:  
👉 [OEM 13c Marketplace Image](https://cloudmarketplace.oracle.com/marketplace/en_US/listing/104183179)

The Marketplace option made setup straightforward — a single-instance OMS installation without manual configuration steps.

---

### **Deployment Setup**
- Chose the **Simple** deployment size which provides a single node deployment with an Oracle single instance database as a repository
- Used an **existing VCN and public subnet** to avoid creating a bastion host.  
- Selected a **Standard2 VM shape** due to **quota restrictions** in my tenancy.  
- Chose to deploy in a **public subnet** so I could access the OEM console directly from my laptop browser.

To resolve an **invalid SSH key** error during provisioning, I generated a new PEM-format key:

```bash
ssh-keygen -t rsa -m PEM
```

### Installation Observations

- **Total installation time:** ~25 minutes  

The setup completed successfully and provided post-install details, including:


**Post-Installation Details**

- **EM URL:** `https://<public_ip>:7799/em`  
- **WebLogic URL:** `https://<public_ip>:7101/console`  
- **EM Middleware Home:** `/u01/app/oracle/em/middleware_135`  
- **EM Agent Home:** `/u01/emagent/agent_13.5.0.0.0`  
- **Oracle Database SID:** `emrep`  
- **DB Connect String:**  
```text
(DESCRIPTION=
    (ADDRESS_LIST=
        (ADDRESS=(PROTOCOL=TCP)(HOST=oms1)(PORT=1521))
    )
    (CONNECT_DATA=
        (SERVICE_NAME=empdb)
    )
)
```

### Accessing OEM

After the installation finished, I accessed the OEM console directly from my laptop browser:

- URL: `https://<public_ip>:7799/em`  
- Login: `sysman/<your password>`

WebLogic Console access was also verified:

- URL: `https://<public_ip>:7101/console`
- Login: `weblogic /<your password>`

Since the VM was deployed in a **public subnet**, I could reach **Oracle Enterprise Manager (OEM)**  — direct browser access worked perfectly.

---

### Optional: Access the OEM Database from SQL Developer on your laptop

You can connect to the **OEM database (empdb)** from your laptop using **SSH tunneling**:

```bash
ssh -L 1521:localhost:1521 opc@<VM_PUBLIC_IP> -i <PRIVATE_KEY>
```
Then, in SQL Developer, create a new connection:

| Field           | Value                     |
|-----------------|---------------------------|
| Username / Role | `SYS` as `SYSDBA`         |
| Password        | `<YOUR_DB_PASSWORD>`       |
| Hostname / IP   | `localhost`               |
| Port            | `1521`                    |
| Service Name    | `empdb`     |

Keep the SSH tunnel session open while connecting from SQL Developer.

### Summary

This was a quick and successful deployment of **OEM 13c on OCI** using an OEM Marketplace image.  
The Marketplace installer handled most of the configuration automatically, and direct browser access made the setup clean and simple.  

**Total time from launch to login:** under 30 minutes.



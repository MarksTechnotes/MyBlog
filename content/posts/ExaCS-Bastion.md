---
title: "Quick Guide: Connect to ExaCS in a private subnet from SQL Developer on your laptop"
date: 2025-11-02
draft: false
tags: ["Oracle", "ExaCS", "SQL Developer", "Bastion", "Port Forwarding"]
categories: ["Cloud", "Database"]
---

### 1. Prerequisites
- SSH key (`id_rsa`) configured.
- Bastion session (SSH Port forwarding) details ready.
- SQL Developer installed.

Reference: Bastion Concepts (https://docs.oracle.com/en-us/iaas/Content/Bastion/Concepts/bastionoverview.htm)


---

### 2. Port Forwarding Command
Forward local port 1521 to the ExaCS VM:

```bash
ssh -v -i id_rsa -N -L 1521:<EXACS_VM_IP>:1521 -p 22 <BASTION_SESSION_OCID>
```

### Explanation

- `-v` → verbose mode (for debugging)  
- `-i id_rsa` → use your SSH key  
- `-N` → no remote shell command, just forward ports  
- `-L 1521:<EXACS_VM_IP>:1521` → forward local port 1521 to the VM’s 1521 port  
- `-p 22` → SSH port for the Bastion  

Keep this terminal session open while connecting from SQL Developer.

---

### 3. Connect Using SQL Developer

1. Open **SQL Developer** on your laptop.  
2. Create a New Connection with the following details:

| Field | Value |
|-------|-------|
| Hostname / IP | localhost |
| Port | 1521 |
| Service Name | `<EXACS_SERVICE_NAME>` |
| Username / Password | Your ExaCS credentials |

3. Test the connection and save.
   
### 4. Tips

- Keep the **Bastion session** active while using SQL Developer.  
- Use `-v` for debugging connection issues.  
- Optionally, create a script or SSH config for repeated use.



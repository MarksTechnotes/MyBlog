---
title: "Connect to ATP using Basic/EZConnect, TNS and/or JDBC"
date: 2026-01-12
draft: false
description: "Connect to ATP using Basic/EZConnect, TNS and/or JDBC"
tags: ["Oracle", "ATP-S", "SQLcl", "TLS", "mTLS", "TNS", "Autonomous Database"]
categories: ["Database", "Autonomous Database"]
---
## Overview

There are several methods to connect to an Oracle Database and ATP including Basic/EZConnect, TNS , JDBC and Wallet. This post talks about these different connection types.

## 1. TCP vs. TCPS: The "Safe" Analogy
In Oracle networking, these define the "pipe" used for your data.

* **TCP (Plain Text):** Like a postcard. Fast, but anyone handling it can read your data. Generally forbidden for Cloud databases.
* **TCPS (Encrypted):** Like a locked safe. It wraps TCP in a TLS (SSL) tunnel. This is **mandatory** for Oracle ATP to protect your data over the internet.

| Feature | TCP | TCPS |
| :--- | :--- | :--- |
| **Full Name** | Transmission Control Protocol | TCP over SSL/TLS |
| **Encryption** | None (Plain Text) | Full (Encrypted) |
| **Cloud Usage** | Not allowed for ATP | **Mandatory** |
| **Analogy** | Postcard | Locked Safe |

## 2. Connection Methods for SQLcl
| Method | Syntax / Format | Notes |
| :--- | :--- | :--- |
| **Basic** | `user/pass@//tcps://host:1521/service_name?ssl_server_dn_match=true` | Uses Easy Connect Plus. Explicitly requires `tcps://` for ATP. |
| **JDBC** | `user/pass@jdbc:oracle:thin:@(description=...)` | Most robust; bypasses local TNS files by passing the full connection string. |
| **TNS** | `user/pass@reviews26ai_tp` | Requires a `tnsnames.ora` file and `TNS_ADMIN` environment variable. |

The `tnsnames.ora` file should be a plain text file containing your connection string mapped to a TNS Name. For example:

```bash
reviews26ai_tp = (description= (retry_count=20)(retry_delay=3)(address=(protocol=tcps)(port=1521)(host=host_name))(connect_data=(service_name=service_name))(security=(ssl_server_dn_match=yes)))
```
You could also optionally directly enter the Connection string above instead of specifying a TNS name which requires an entry in a `tnsnames.ora` file. 

---

## 3. Protocol and Port Relationship
In Oracle Cloud (OCI), ports are mapped to specific security behaviors:

* **Port 1521 (TLS/One-Way):** Encrypted connection that does **not** require a client-side wallet. SQLcl uses the standard Java truststore to verify the server.
* **Port 1522 (mTLS/Mutual):** Encrypted connection that typically **requires a wallet**.
* **The "Optional" Rule:** If the Cloud Console is set to **mTLS: Optional**, you can connect to Port 1522 without a wallet because the server allows a fallback to One-Way TLS.

---

## 4. Network Configuration Files
| File | Location | Role |
| :--- | :--- | :--- |
| **tnsnames.ora** | Client | The "Address Book." Maps aliases to full connection strings. |
| **sqlnet.ora** | Client/Server | The "Policy Manual." Sets timeouts, wallet locations, and naming methods. |
| **listener.ora** | Server | The "Receiver." Listens for incoming traffic on specific ports (1521/1522). |

---

## 5. Key SQLcl Commands
* `show tns`: Displays where SQLcl is looking for config files and which aliases it found.
* `show connection`: Shows the current JDBC URL and driver type (Thin vs. Thick).
* `set TNS_ADMIN=<path>`: Manually points SQLcl to your configuration folder during a session.

---

## 6. Local Setup Checklist (macOS)
1.  **Driver:** Using **JDBC Thin** (standard in SQLcl/VS Code), so no Oracle Client installation is required.
2.  **Environment:** Set `TNS_ADMIN` in `~/.zshrc` to point to your configuration folder.
3.  **Security:** Always use `(PROTOCOL=tcps)` for ATP connections to ensure encryption.
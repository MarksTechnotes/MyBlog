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

## 1. Connection Methods for SQLcl
| Method | Syntax / Format | Notes |
| :--- | :--- | :--- |
| **Basic** | `user/pass@//tcps://host:1521/service_name?ssl_server_dn_match=true` | Uses Easy Connect Plus. Explicitly requires `tcps://` for ATP. |
| **JDBC** | `user/pass@jdbc:oracle:thin:@(DESCRIPTION=...)` | Most robust; bypasses local TNS files by passing the full descriptor. |
| **TNS** | `user/pass@network_alias` | Requires a `tnsnames.ora` file and `TNS_ADMIN` environment variable. |

---

## 2. Protocol and Port Relationship
In Oracle Cloud (OCI), ports are mapped to specific security behaviors:

* **Port 1521 (TLS/One-Way):** Encrypted connection that does **not** require a client-side wallet. SQLcl uses the standard Java truststore to verify the server.
* **Port 1522 (mTLS/Mutual):** Encrypted connection that typically **requires a wallet**.
* **The "Optional" Rule:** If the Cloud Console is set to **mTLS: Optional**, you can connect to Port 1522 without a wallet because the server allows a fallback to One-Way TLS.

---

## 3. Network Configuration Files
| File | Location | Role |
| :--- | :--- | :--- |
| **tnsnames.ora** | Client | The "Address Book." Maps aliases to full connection strings. |
| **sqlnet.ora** | Client/Server | The "Policy Manual." Sets timeouts, wallet locations, and naming methods. |
| **listener.ora** | Server | The "Receiver." Listens for incoming traffic on specific ports (1521/1522). |

---

## 4. Key SQLcl Commands
* `show tns`: Displays where SQLcl is looking for config files and which aliases it found.
* `show connection`: Shows the current JDBC URL and driver type (Thin vs. Thick).
* `set TNS_ADMIN=<path>`: Manually points SQLcl to your configuration folder during a session.

---

## 5. Local Setup Checklist (macOS)
1.  **Driver:** Using **JDBC Thin** (standard in SQLcl/VS Code), so no Oracle Client installation is required.
2.  **Environment:** Set `TNS_ADMIN` in `~/.zshrc` to point to your configuration folder.
3.  **Security:** Always use `(PROTOCOL=tcps)` for ATP connections to ensure encryption.
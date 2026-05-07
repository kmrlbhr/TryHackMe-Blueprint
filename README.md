Enviroment Setup

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

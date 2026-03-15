# WRITEUP
**Operating System**: _Linux_  
**HTB Difficulty**: _Easy_  
**My Recommended Difficulty**: _Medium_  
**Summary**: _This machine introduces a vulnerable CMS application where users are forced to make educated inferences about available exploits based on limited information and under conditions that restrict too many wrong guesses. Once initial access is obtained, Writeup tests the user's understanding of $PATH lookup order and the ability to identify relative binaries that can be hijacked._  


### STEP #1: Port Scanning
**TCP Scan**
* `sudo nmap -Pn -n X.X.X.X -sC -sV -sT -p- --open`  

  *  <img width="535" height="186" alt="Image" src="https://github.com/user-attachments/assets/63e818b5-e2ef-4dc5-a642-d273935c2e50" />

  
**UDP Scan**
* `nmap -v -sU -T4 -Pn --top-ports 100 X.X.X.X`  

  * No results shown.

We see from our scan results that we have a web server featuring an interesting directory named _writeup_ as well as SSH accessibility to administer the web server. We will want to perform thorough enumeration of the web server. Typical initial access exploits within SSH only involve identifying old/vulnerable versions or abusing weak credentials using brute force. OpenSSH 9.2p1 does appear vulnerable to RegreSSHion but this is a complex exploit requiring a long duration of time (see _https://cyberint.com/blog/research/cve-2024-6387-rce-vulnerability-in-openssh/_) therefore I doubt that this supposedly easy machine requires that avenue. Alternatively, SSH enumeration can be used to identify the operating system of the target and we can try to abuse that for access. At this time, since we see an obvious connection between the name of the machine and the web server directory structure, let's deep dive the web application.

### STEP #2: Service Enumeration
**HTTP (Port 80)**:
 * **Manual Enumeration**:  
Inside _FireFox_, we navigate to `http://X.X.X.X/` and are confronted with the home page indicating that DoS protection is in place and that our IP could be banned if enought 40x errors occur. We will want to refrain from performing fuzzing for the remainder of this machine. One additional piece of information we discover is an email address `jkr@writeup.htb`. This may become useful later so let's store that for now.  
Let's navigate to `http://X.X.X.X/robots.txt` given that our nmap scan pointed that out to us where we can confirm that _/writeup/_ is disallowed meaning search engines would not crawl that page. Navigating to `http://X.X.X.X/writeup/`, we are confronted with a page indicating that we found a repository of HTB machine writeups similar to the repository I have been putting together here. In this repository, we see three machines (_ypuffy_, _blue_, and _writeup_).  
    * If we inspect the source code of `http://X.X.X.X/writeup/`, we can deduce from the header that the server is running _CMS Made Simple_.  
    
      <img width="690" height="200" alt="Image" src="https://github.com/user-attachments/assets/d35b943e-f0e0-4beb-ae55-f68553ad1061" />  
      
    * Furthermore, we discover that there is stored session cookie. Perhaps this will be useful later as well.  
      
        <img width="1173" height="197" alt="Image" src="https://github.com/user-attachments/assets/53f28942-76ad-4820-87ca-2ecd2812cef9" />  

 At this point, we know that the web server is hosting _CMS Made Simple_ with a copyright of 2019. Without the version number, it can be tricky to look up known exploits but let's see if we can find any exploits in the 2019 timeframe. A quick Google search locates _https://www.exploit-db.com/exploits/46635_ which is a SQL injection exploit as the top result. Let's download this exploit and try it out:
  * `searchsploit -m 46635`
  * `python3 ./46635.py -u http://X.X.X.X/writeup/`
    * NOTE: to do this, I had to first convert this exploit to Python3 from Python2 (using _2to3_) and use Pip to install several additional packages. I recommend doing this in a virtual environment. I used the path specified because inspection of the exploit is appending _/moduleinterface.php?_ to the input specified so using the full URL to include _index.php_ would not make sense.
        <img width="442" height="87" alt="Image" src="https://github.com/user-attachments/assets/f90629a8-d087-4ca0-82d5-e50f398b19f7" />

_Having confirmed the username we discovered previously and now discovered a corresponding password hash, we will proceed forward to initial access. In the event that this had not worked, other strategies we could have tried include finding vulnerabilities against the Apache 2.4.25 application and trying additional exploits against CMS Made Simple_.

### STEP #3: Initial Access
Let's first work to crack the password hash. Using tools like `hashid` or `hash-identifier`, we can presume that the hash is simply an MD5 hash. Using _https://hashcat.net/wiki/doku.php?id=example_hashes_, we can form the expected salted hash structure using `echo "62def4866937f08cc13bab43bb14e6f7:5a599ef579066807" > hash.txt`.
 * `hashcat -m 20 hash.txt /usr/share/wordlists/rockyou.txt`
   * We successfully crack the password and determine it to be `raykayjay9`.
  
 Let's login to obtain initial access:  `ssh -p 22 jkr@X.X.X.X` and locate the user flag at _/home/jkr/user.txt_.


### STEP #4: Privilege Escalation
With access to the system, the first thing we should do is upload and run _linpeas.sh_. The intersting observations we can make are the following:
  * We have write permission to _/usr/local/bin_ and _/usr/local/sbin_ due to our membership in _staff_.
  * _/usr/local/bin_ and _/usr/local/sbin_ are \the first priority paths inside crontab.

This means that path hijacking is going to be the likely exploit for privilege escalation. We need to find a binary file being called by it's relative path (E.g., _test_, not _/usr/bin/env/test_) so that we can create a malicious binary and write it to _/usr/local/bin_ or _/usr/local/sbin_ where it will be called with a higher priority.
  * We upload _pspy64_ to monitor background processes being called and happen to notice when we login via a second SSH session that _run-parts_ is executed.
    
      <img width="922" height="156" alt="Image" src="https://github.com/user-attachments/assets/d06be5cc-916b-41ea-a3c7-80753c2f6a1f" />  
  * `echo "chmod u+s /bin/bash" > /usr/local/bin/run-parts`
  * `chmod +x /usr/local/bin/run-parts`

Login again using `ssh -p 22 jkr@X.X.X.X` and we will notice that our sesion appearance changes. Run `/bin/bash -p` to assume the root user. From here, the root flag can be found at _/root/root.txt_.  
  * <img width="193" height="98" alt="Image" src="https://github.com/user-attachments/assets/d452afff-e88e-4e27-be5a-9397fe133aa7" />  


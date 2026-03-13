# microsoft-sccm-lab
This guide provides a comprehensive, step-by-step walkthrough for building a fully functional Microsoft Endpoint Configuration Manager (SCCM/MECM) home lab using VirtualBox. Perfect for IT professionals preparing for certifications, learning SCCM administration, or testing deployment scenarios in a safe environment. The lab consists of three virtual machines: a Windows Server 2022 Domain Controller with DNS and DHCP, a Windows Server 2022 SCCM Primary Site Server with SQL Server 2019, and a Windows 10/11 client for testing deployments. All components are configured to work together in an isolated network environment, allowing you to practice application deployment, operating system deployment, software updates, compliance settings, and reporting without affecting production systems. The entire setup requires approximately 8-10 hours to complete and assumes basic knowledge of Windows Server, Active Directory, and virtualization concepts.  

Windows 2022 Server download: https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022  
Windows 10/11 Enterprise: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise  
SQL Server Developer Edition download: https://go.microsoft.com/fwlink/?linkid=866662  
Microsoft Endpoint Configuration Manager Current Branch (SCCM): https://info.microsoft.com/ww-landing-microsoft-endpoint-configuration-manager.html?culture=en-us&country=us  
Windows Assessment and Deployment Kit ADK: https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install  

First go and setup the NAT Network that will be used:  
SCCM Lab: 192.168.10.0/24  

Setup:  
VM1: Domain Controller  
Memory: 4096 MB  
Space: 60 GB  
Network: NAT Network on Adapter 1  (SCCM Lab)
Processer: 2 CPUs  
Version: Windows 2022

VM2: SCCM01  
Processer: 4 CPUs  
Network: NAT Network on Adapter 1  (SCCM Lab)
Memory: 12288 MB  
Space: 150 GB  

VM3: Client  
Memory: 4096 MB  
Network: NAT Network on Adapter 1  
Space: 60 GB  

# Setting up VM1  
After starting it up you will be greeted by this screen and press next:  
<img width="1022" height="763" alt="image" src="https://github.com/user-attachments/assets/ca8493bb-1015-4268-92fe-7e12a5ca3cf2" />  
Then click on install now and wait a couple seconds and you will see this screen, go to the second option:  
<img width="1029" height="769" alt="image" src="https://github.com/user-attachments/assets/6fdfe9ad-3a57-4115-bc5b-ade215d678b7" />  
Click on next until see the option for custom installation, click on it and continue and then wait a while for the VM to install.  
After waiting a while, you will be greeted with this screen:  
<img width="1027" height="769" alt="image" src="https://github.com/user-attachments/assets/077bfe15-56bd-405c-a581-679310fe6dd6" />  
Set the Admin password as something that can easily remembered and write it down. Then sign in when prompted and wait it for it boot up.  
After booting up:  
<img width="1032" height="776" alt="image" src="https://github.com/user-attachments/assets/94ba63b8-2315-466c-8228-8e8f5d6b39e2" />  
Then click on Local Server, then Ethernet,  
<img width="1024" height="779" alt="image" src="https://github.com/user-attachments/assets/9d4664e1-e09b-49f9-970c-f4780e2441cf" />  
Then double click on it to open up the Ethernet Status, then click on properties, then click on TCP/IPv4, and enter in this information:  
<img width="1025" height="770" alt="image" src="https://github.com/user-attachments/assets/921b4f89-1439-476c-858e-94bffd6e4560" />  
Then click on OK twice, and restart the VM and login again ad the Admin account.  
After it restarts click on Manage and click on Add roles and features.  
Then click on next a couple times till you hit server roles and check Active Directory Domain Services, then hit next a couple more times and click on install and wait a few minutes, then click close.  
Then click on the yellow triangle icon and click promote to domain controller, and then click on add a new forest and call it lab.local and click on next and set a password for the DSRM that can be remembered.  
Then hit next for Create DNS and for NetBIOS name it LAB and hit next, until the prerequisite check and after its done click on install and wait for it to restart.  
After restarting, sign in as admin in the LAB NetBIOS name with the password that was set.  
Then go back to manage and click on add roles and features once again and click on next a couple more times until you get to server roles and click on DHCP Server this time and click next a couple more times, then click on install and wait.  
Once its done click on the flag icon and click on complete DHCP configuration, click on next, then commit and close it out.  
Then in Server Manager click on tools and then DHCP:  
<img width="1029" height="767" alt="image" src="https://github.com/user-attachments/assets/c91fdd00-c9fd-4e45-8480-72ba5a297ce6" />  
Right click on IPv4 and choose New Scope and name the scope SCCMLab Scope and hit next and enter this for the IP configuration:  
<img width="520" height="431" alt="image" src="https://github.com/user-attachments/assets/1970b640-4a1e-43d6-a8f9-1c5649c9146e" />  
Then hit next a couple times with the lease duration set to 8 days and select yes to configuring the options now and make the default gateway 192.168.10.1, add it and hit next.  
Then continue hitting next until you see finish and click on it.  
Then go to tools and go to Active Directory Users and Computers and expand lab.local and right click on users and create 3 new users:  

Account 1:  
First name: SCCM Admin  
Logon: sccmadmin  
Password: something easily remembered, and uncheck at next login, and never expires and click on next and finish.  

Account 2:
First name: SQL Service  
Logon: sqlservice  
Password: something easily remembered, and uncheck at next login, and never expires.  

Account 3:  
SCCM NAA  
Logon: sccmnaa  
Password: something easily remembered, and uncheck at next login, and never expires.  

Then refresh lab.local and go the SCCMAdmin user, what its suppose to look like:  
<img width="839" height="527" alt="image" src="https://github.com/user-attachments/assets/24111b0d-32fc-42a1-b714-e18203b71d4e" />  
Then right click on sccmadmin user and choose properties, then go to member of tab and click add and type Domain Admins, check names to verify and click on ok, then appy and then ok.  

# Setting up VM2
Click on new and create the VM, and set it to Microsoft Windows version 2022 G4 Bit.  
Memory: 12288 MB or 12 GB.  
Video Memory: 128 MB  
Space: 150 GB  
Processors: 4 or 5  
Network: NAT Network set to SCCM Lab.  
Then attach the Windows Server 2022 ISO to the storage.  

Then start the VM and follow the installation steps similiar to the first VM.  
Click next on the first screen then install now, then for OS select the second opption for standard evaluation, then click on custom so you can see the unallocated space option for the drive.  
Then wait for a while for the VM to get up and installed.  
Then after a few minutes, the screen to customize settings will pop up to set the password and username. After setting the password, click finish and then you will see the login screen.  
After a few seconds the server manager should appear.  
<img width="1023" height="777" alt="image" src="https://github.com/user-attachments/assets/8f8a6d7f-abd9-47cd-9956-7ce7c5bbfa20" />  
Then go to Local Server and click on Ethernet:  
<img width="764" height="414" alt="image" src="https://github.com/user-attachments/assets/e42d6053-1f0d-45c3-b3bd-6a0f0e25e9f7" />  
After clicking on the thing beside it:  
<img width="788" height="593" alt="image" src="https://github.com/user-attachments/assets/c1bcfe9b-be8b-45e2-8207-d6b3b39d9469" />  
Then right click on Ethernet -> then Properties and then double click on IPv4 and enter in this info:  
<img width="401" height="461" alt="image" src="https://github.com/user-attachments/assets/10d9b65d-24fc-4f14-9166-8ed910480116" />  
Then click on OK twice and and close it out.  
Then go to in Local Server and click on Computer Name:  
<img width="418" height="469" alt="image" src="https://github.com/user-attachments/assets/1f05fd8d-063c-4cc9-8059-6e912e4b1e7e" />  
Click on change and name it SCCM01 and the domain type lab.local and then click on OK.  
*Make sure to have the DC up and running during this*  
After clicking on OK you will see this:  
<img width="462" height="303" alt="image" src="https://github.com/user-attachments/assets/93639a51-6101-4852-908c-364254de6b4d" />  
After entering in the correct credentials you will see this:  
<img width="266" height="148" alt="image" src="https://github.com/user-attachments/assets/60b25051-5349-4297-9317-44e8bd8e913e" />  
Then click OK a couple times and then go and restart the VM.  
After restarting you should see the Other User option pop up:  
<img width="1043" height="777" alt="image" src="https://github.com/user-attachments/assets/51b31487-41be-4837-9ba7-e9077a031482" />  
Click on Other User and sign as the SCCMAdmin user account that was created.  
After waiting for a few seconds you should see this:  
<img width="1027" height="778" alt="image" src="https://github.com/user-attachments/assets/7c3074cc-3927-48a4-9223-c0e83a1df3db" />  
Then go to the Server Manager and click on Manage -> Add Roles and Features, then click on Web Server.  
Then go check .Net Extensibility 3.5 and 4.8, ASP.NET 3.5 and ASP.NET 4.8 and their sub items, check IIS 6 and its subs, Security with windows authentication checked and a couple others.  
Then click next till installation review, check restart, hit yes and then install.  
After waiting for a few minutes, then click close and 
























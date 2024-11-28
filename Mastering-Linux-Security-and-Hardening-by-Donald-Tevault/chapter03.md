**Chapter 03: Securing Normal User Accounts**

- Locking down users’ home directories
- Enforcing strong password criteria
- Setting and enforcing password and account expiration
- Preventing brute-force password attacks
- Locking user accounts
- Setting up security banners
- Detecting compromised passwords
- Understanding central user management systems

**Locking down users’ home directories the Red Hat way**
CentOS 7 VM, scroll down toward the bottom of the file, and you’ll see this:
```
CREATE_HOME yes
UMASK 077
```

In the **login.defs** file of a RHEL 8 or RHEL 9-type distro, such as AlmaLinux, you’ll see that the UMASK is set for wide-open permissions, which seems a bit strange. Here’s what that looks like:
```
UMASK 022
```
But a few lines below that, you’ll see a brand-new directive that we never had before, which looks
like this:
```
HOME_MODE 0700
```
 
So even though the UMASK is wide-open, new user home directories still get properly locked down.
Most non-Red Hat distros usually have a **UMASK value of 022**, which creates home directories with a permissions value of **755**. This allows everybody to enter everybody else’s home directories and access each others’ files.

**Locking down users’ home directories the Debian/ Ubuntu way**
```
sudo useradd -m -d /home/frank -s /bin/bash frank
```
Here’s a breakdown of what all this means:
- -m creates the home directory.
- -d specifies the home directory.
- -s specifies Frank’s default shell. (Without the -s, Debian/Ubuntu would assign to Frank the /bin/sh shell.)
```
drwxr-xr-x 2 frank frank 4096 Oct 1 23:58 frank
```

To change the default permissions setting for home directories, open **/etc/login.defs** for editing.
Look for this line:
```
UMASK 077
```

**Hands-on lab for creating an encrypted home directory with adduser**
For this lab, we’ll be working with the adduser utility on an Ubuntu 22.04 VM:
1. Install the ecryptfs-utils package:
```
sudo apt install ecryptfs-utils
```
2. Create a user account with an encrypted home directory for Cleopatra and then view the results:
```
sudo adduser --encrypt-home cleopatra
ls -l /home
```
3. Log in as Cleopatra and run the ecryptfs-unwrap-passphrase command:
```
su - cleopatra
ecryptfs-unwrap-passphrase
exit
```

**Installing and configuring pwquality**
We’ll be using the pwquality module for the **Pluggable Authentication Module (PAM)**. This is a newer technology that has replaced the old cracklib module.
On Debian and Ubuntu, you’ll need to install pwquality yourself, like this:
```
sudo apt install libpam-pwquality
```
The rest of the procedure is the same for all of our operating systems and consists of just editing
the /etc/security/pwquality.conf file. 

**Setting expiry data on a per-account basis with useradd**
**and usermod**
Let’s say that you want to create an account for Charlie that will expire at the end of 2025. On a RedHat-type machine, you could enter this:
```
sudo useradd -e 2025-12-31 charlie
```
On a non-Red Hat-type machine, you’d have to add the option switches that create the home directory and assign the correct default shell:
```
sudo useradd -m -d /home/charlie -s /bin/bash -e 2025-12-31 charlie
```
Use chage -l to verify what you’ve entered:
```
donnie@ubuntu-steemnode:~$ sudo chage -l charlie
```
Now, let’s say that Charlie’s contract has been extended, and you need to change his account expiration to the end of January 2026. You’ll use usermod the same way on any Linux distro:
```
sudo usermod -e 2026-01-31 charlie
```
Optionally, you can set the number of days before an account with an expired password will get locked out:
```
sudo usermod -f 5 charlie
```

**Preventing brute-force password attacks**
Configuring the pam_tally2 PAM module on CentOS 7
To make this magic work, we’ll rely on our good friend, PAM. The pam_tally2 module comes already
installed on CentOS 7, but it isn’t configured. We’ll begin by e**diting the /etc/pam.d/login file**. 
```
auth required pam_tally2.so deny=4 even_deny_root unlock_time=1200
```
- deny=4: This means that the user account under attack will get locked out after only four failed login attempts.
- even_deny_root: This means that even the root user account will get locked if it’s under attack.
- unlock_time=1200: The account will get automatically unlocked after 1,200 seconds, or 20 minutes.
You can also use **pam_tally2** to manually unlock a locked account.
```
donnie@centos7:~$ sudo pam_tally2
Login Failures Latest failure From
charlie 5 10/07/17 16:38:19
donnie@centos7:~$ sudo pam_tally2 --user=charlie --reset
```

**Hands-on lab for configuring pam_faillock on Ubuntu 20.04 and Ubuntu 22.04**
1. Open the **/etc/pam.d/common-auth** file in your favorite text editor. At the top of the file, insert these two lines:
```
auth required pam_faillock.so preauth silent
auth required pam_faillock.so authfail
```
2. Open the /etc/pam.d/common-account file in your text editor. At the bottom of the file, add this line:
```
account required pam_faillock.so
```
3. Configure the **/etc/security/faillock.conf** file the same way that I showed you in Step 5 of the preceding lab for AlmaLinux.
4. Test the setup as outlined in Steps 6 through 8 of the preceding AlmaLinux lab.
5. And that’s all there is to it. Next, let’s look at how to manually lock a user’s account.
**Using usermod to lock a user account**
Let’s say that Katelyn has gone on maternity leave and will be gone for several weeks. We can lock
her account by doing:
```
sudo usermod -L katelyn
```
When you look at Katelyn’s entry in the /etc/shadow file, you’ll now see an exclamation point in front of her password hash, like this:
```
katelyn:!$6$uA5ecH1A$MZ6q5U.cyY2SRSJezV000AudP.
ckXXndBNsXUdMI1vPO8aFmlLXcbGV25K5HSSaCv4RlDilwzlXq/hKvXRkpB/:17446:0:99999:7:::
```
This exclamation point prevents the system from reading her password hash, which effectively locks her out of the system.
To unlock her account, just do this:
```
sudo usermod -U katelyn
```

**Setting up security banners**
Using the motd file
motd stands for **Message of the Day**.
The **/etc/motd** file will present a message banner to anyone who logs in to a system through Secure Shell.
Using the issue file
The issue file, also found in the /etc  directory, shows a message on the local terminal, just above
the login prompt.

**Detecting compromised passwords**
So, how do you know if your password is on one of those lists? Easy. Just use one of the online services that will check your password for you. One popular site is **Have I Been Pwned?**

**Understanding centralized user management**

**Microsoft Active Directory**
I’m not exactly a huge fan of either Windows or Microsoft. But when it comes to Active Directory, I’ll have to give credit where it’s due.

**Samba on Linux**
Samba is a Unix/Linux daemon that can serve three purposes:
- Its primary purpose is to share directories from a Unix/Linux server with Windows workstations. The directories show up in Windows File Explorer as if they were being shared from other Windows machines.
- It can also be set up as a network print server.
- It can also be set up as a Windows domain controller.

**FreeIPA/Identity Management on RHEL-type distros**
This is what IPA stands for:
- Identity
- Policy
- Audit
It’s something of an answer to Microsoft’s Active Directory, but it still isn’t a complete one. It does some cool stuff, but it’s still very much a work in progress. 









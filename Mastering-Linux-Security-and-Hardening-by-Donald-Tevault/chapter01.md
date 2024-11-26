Chapter 02 Securing Administrative User Accounts


**The advantages of using sudo:**
- Assign certain users full administrative privileges, while assigning other users only the privileges they need to perform tasks that are directly related to their respective jobs.
- Allow users to perform administrative tasks by entering their own normal user passwords so that you don’t have to distribute the root password to everybody and their brother.
- Make it harder for intruders to break into your systems. If you implement sudo and disable the root user account, would-be intruders won’t know which account to attack because they won’t know which one has admin privileges.
- Create sudo policies that you can deploy across an entire enterprise network, even if that network has a mix of Unix, BSD, and Linux machines.
- Improve your auditing capabilities because you’ll be able to see what users are doing with their admin privileges.

On Unix, BSD, and most Linux systems, you would add users to the wheel group. (Members of the Red
Hat family, including CentOS and AlmaLinux, fall into this category.) When I do the groups command
on any of my RHEL-type machines, I get this:
```
[donnie@localhost ~]$ groups
donnie wheel
[donnie@localhost ~]$
```

```
## Allows people in group wheel to run all commands
%wheel ALL=(ALL) ALL
```
ALL mean that
**members of that group can perform ALL commands, as ALL users, on ALL machines in the network.**

```
sudo visudo
```
You can, for example, create a BACKUPADMINS user alias for backup administrators, a WEBADMINS user alias for web server administrators, or whatever else you desire. So, you could add a line that looks something like this:
```
User_Alias SOFTWAREADMINS = vicky, cleopatra
```
That’s good, except that Vicky and Cleopatra still can’t do anything. You’ll need to assign some duties to the user alias.

 when a command is listed in the command alias with a subcommand, option, or argument, that’s all anyone who’s assigned to the command alias can run. With the SERVICES command alias in its current configuration, the systemctl commands just won’t work.
A better solution would be to add a wildcard to each of the systemctl subcommands:
```
Cmnd_Alias SERVICES = /sbin/service, /sbin/chkconfig, /usr/bin/systemctl start
*, /usr/bin/systemctl stop *, /usr/bin/systemctl reload *, /usr/bin/systemctl
```

 For example, you could create a WEBSERVERS host alias, a WEBADMINS user alias, and a WEBCOMMANDS command alias with the appropriate commands.
Your configuration would look something like this:
```
Host_Alias WEBSERVERS = webserver1, webserver2
User_Alias WEBADMINS = junior, kayla
Cmnd_Alias WEBCOMMANDS = /usr/bin/systemctl status httpd, /usr/bin/systemctl
start httpd, /usr/bin/systemctl stop httpd, /usr/bin/systemctl restart httpd

WEBADMINS WEBSERVERS=(ALL) WEBCOMMANDS
```
Now, when a user types a command into a server on the network, sudo will first look at the hostname of that server. If the user is authorized to perform that command on that server, then sudo allows it. Otherwise, sudo denies it.

**The sudo timer**
By default, the sudo timer is set for 5 minutes. 
 you could just reset the sudo timer by running this command:
```
sudo -k
```

**Preventing users from using shell escapes**
Certain programs, especially text editors and pagers, have a handy shell escape feature. This allows a user to run a shell command without having to exit the program first. 
Other programs that have a shell escape feature include the following:
- **Vim**
- **emacs**
- **less**
- **view**
- **more**
Let’s fix that by adding the **NOEXEC**: option to the su-
doers rule:
```
vicky ALL=(ALL) NOEXEC: /usr/bin/less
```
This prevents Vicky from escaping to even her own shell.

**Preventing users from using other dangerous programs**
Some programs that don’t have shell escapes can still be dangerous if you give users unrestricted
privileges to use them. These include the following:
- cat
- cut
- awk
- sed

Let’s say that Vicky is a database admin, and you want her to run as the database user:
```
vicky ALL=(database) /usr/local/sbin/some_database_script.sh
```
Vicky could then run the command as the database user by entering the following command:
```
sudo -u database some_database_script.sh
```
This is one of those features that you might not use that often, but keep it in mind anyway. You never know when it might come in handy.

Tags: [[Linux]], [[Security]]. [[Mastering Linux Security and Hardening Book by Donald A. Tevault]]

**Securing Your Server with a Firewall – Part 1**
> The name of the Linux firewall is netfilter. This netfilter code is compiled into the Linux kernel and is what performs the actual packet filtering.

There have been three helper programs, which are:
- **ipchains**: This was the first one and was part of the Linux kernel up through kernel version  2.4. It’s now ancient history, so we won’t say any more about it.
- **iptables**: This replaced ipchains in Linux kernel version 2.6. It’s still used in a lot of Linux  distros but is rapidly disappearing.
- **nftables**: This is the new kid on the block and is rapidly replacing iptables. As we’ll see later, it has a lot of advantages over the older iptables.
All three of these helper programs do two things for us:
- They provide a command-line interface for human users.
- They take the commands that human users input and inject them into netfilter.

**An overview of iptables**
There are also a few disadvantages:
- IPv4 and IPv6 each require their own special implementation of iptables. So, if your organization still needs to run IPv4 while in the process of migrating to IPv6, you’ll have to configure two firewalls on each server and run a separate daemon for each. (One for IPv4; the other for IPv6.)
- If you need to do MAC bridging, that requires ebtables, which is the third component of iptables, with its own unique syntax.
- arptables, the fourth component of iptables, also requires its own daemon and syntax.
- Whenever you add a rule to a running iptables firewall, the entire iptables ruleset has to be reloaded, which can have an impact on performance.

**Mastering the basics of iptables**
iptables consists of five tables of rules, each with its own distinct purpose:
- Filter table: For basic protection of our servers and clients, this might be the only table that we use.
- Network Address Translation (NAT) table: NAT is used to connect the public Internet to private networks.
- Mangle table: This is used to alter network packets as they go through the firewall.
- Raw table: This is for packets that don’t require connection tracking.
- Security table: The security table is only used for systems that have SELinux installed.

```
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```
- -A INPUT: -A places a rule at the end of the specified chain, which in this case is the INPUT  chain. We would have used -I had we wanted to place the rule at the beginning of the chain.
- -m: This calls in an iptables module. In this case, we’re calling in the conntrack module to track connection states. This module allows iptables to determine whether our client has made a connection to another machine, for example.
- --ctstate: The ctstate, or connection state, portion of our rule is looking for two things. First, it’s looking for a connection that the client established with a server. Then, it looks for the related connection that’s coming back from the server in order to allow it to connect to the client. So, if a user was to use a web browser to connect to a website, this rule would allow packets from the web server to pass through the firewall to get to the user’s browser.
- -j: This stands for jump. Rules jump to a specific target, which in this case is ACCEPT. (Please don’t ask me who came up with this terminology.) So, this rule will accept packets that have been returned from the server to which the client has requested a connection.

```
sudo iptables -A INPUT -p tcp --dport ssh -j ACCEPT
```
Here’s the breakdown:
- -A INPUT: As before, we want to place this rule at the end of the INPUT chain with -A.
- -p tcp: -p indicates the protocol that this rule affects. This rule affects the TCP protocol, of which Secure Shell is a part.
- --dport ssh: When an option name consists of more than one letter, we need to precede it with two dashes, instead of just one. The --dport option specifies the destination port on which we want this rule to operate. (Note that we could have also listed this portion of the rule as --dport 22 since 22 is the number of the SSH port.)
- -j ACCEPT: If we put this all together with -j ACCEPT, then we have a rule that allows other machines to connect to this one through Secure Shell.

Now, let’s say that we want this machine to be a DNS server. For that, we need to open port 53 for both the TCP and UDP protocols:
```
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
```

we need to allow traffic for the loopback interface. This is okay because it gives us a good chance to see how to insert a rule where we want it if we don’t want it at the end.
```
sudo iptables -I INPUT 1 -i lo -j ACCEPT
```


**Blocking ICMP with iptables**
 **Internet Control Message Protocol (ICMP)**. The idea you may have been told is to make your server invisible to hackers by blocking ping packets.
- By using a botnet, a hacker could inundate your server with ping packets from multiple sources at once, exhausting your server’s ability to cope.
- Certain vulnerabilities that are associated with the ICMP protocol can allow a hacker to either gain administrative privileges on your system, redirect your traffic to a malicious server, or crash your operating system.
```
sudo iptables -A INPUT -m conntrack -p icmp --icmp-type 3 --ctstate
NEW,ESTABLISHED,RELATED -j ACCEPT
```
Here’s the breakdown:
- -m conntrack: As before, we’re using the conntrack module to allow packets that are in a cer tain state. This time, though, instead of just allowing packets from a host to which our server has been connected (ESTABLISHED,RELATED), we’re also allowing NEW packets that other hosts are sending to our server.
- -p icmp: This refers to the ICMP protocol.
- --icmp-type: There are quite a few types of ICMP messages, which we’ll outline next.
Here are the three types of ICMP messages that we want to allow:
- type 3: These are the “destination unreachable” messages. Not only can they tell your server that it can’t reach a certain host, but they can also tell it why. For example, if the server has sent out a packet that’s too large for a network switch to handle, the switch will send back an ICMP message that tells the server to fragment that large packet. Without ICMP, the server would have connectivity problems every time it tries to send out a large packet that needs be broken up into fragments.
- type 11: Time-exceeded messages let your server know that a packet that it has sent out has either exceeded its Time-to-Live (TTL) value before it could reach its destination, or that a fragmented packet couldn’t be reassembled before the TTL expiration date. 
- type 12: Parameter problem messages indicate that the server had sent a packet with a bad IP header. In other words, the IP header is either missing an option flag or it’s of an invalid length.
Three common message types are conspicuously absent from our list:
- type 0 and type 8: These are the infamous ping packets. Actually, type 8 is the echo request packet that you would send out to ping a host, while type 0 is the echo reply that the host would return to let you know that it’s alive. Of course, allowing ping packets to get through could be a big help when troubleshooting network problems. If that scenario ever comes up, you could just add a couple of iptables rules to temporarily allow pings.
- type 5: Now, we have the infamous redirect messages. Allowing these could be handy if you have a router that can suggest more efficient paths for the server to use, but hackers can also use them to redirect you to someplace that you don’t want to go. So, just block them.

**Blocking everything that isn’t allowed with iptables**
The difference between **DROP** and **REJECT** is that DROP blocks packets without sending  message back to the sender. REJECT blocks packets, and then sends a message back to the sender  why the packets were blocked. For our present purposes, let’s say that we just want to DROP  that we don’t want to get through. 
```
sudo iptables -A INPUT -j DROP
```

This all looks great, except that if we were to reboot the machine right now, the rules would disappear. The final thing that we need to do is make them permanent. There are several ways to do this, but the simplest way to do this on an Ubuntu machine is to install the iptables-persistent package:
```
sudo apt install iptables-persistent
```

**Blocking invalid packets with iptables**

**TCP three-way handshake:**
- Your workstation sends a packet with only the SYN flag set to the web server. This is your workstation’s way of saying, “Hello, Mr. Server. I’d like to make a connection with you.” 
- After receiving your workstation’s SYN packet, the web server sends back a packet with the  SYN and ACK flags set. With this, the server is saying, “Yeah, bro. I’m here, and I’m willing to  connect with you.” 
- Upon receipt of the SYN-ACK packet, the workstation sends back a packet with only the ACK  flag set. With this, the workstation is saying, “Cool deal, dude. I’m glad to connect with you.”  
- Upon receipt of the ACK packet, the server sets up the connection with the workstation so that they can exchange information.
**With these so-called invalid packets, a few things could happen:**
- The invalid packets could be used to elicit responses from the target machine in order to find out what operating system it’s running, what services are running on it, and which versions of the services are running.
- The invalid packets could be used to trigger certain sorts of security vulnerabilities on the target machine.
- Some of these invalid packets require more processing power than what normal packets require, which could make them useful for performing Denial-of-Service (DoS) attacks.
> By depending on this DROP all rule, we’re allowing these invalid packets to travel through the entire INPUT chain, looking for a rule that will let them through.
Ideally, we’d like to block these invalid packets before they travel through the entire INPUT chain.
```
donnie@ubuntu:~$ sudo iptables -t mangle -A PREROUTING -m conntrack --ctstate
INVALID -j DROP
donnie@ubuntu:~$ sudo iptables -t mangle -A PREROUTING -p tcp ! --syn -m
conntrack --ctstate NEW -j DROP
```
```
 sudo iptables -t mangle -L
 sudo iptables-save > rules.v4
```

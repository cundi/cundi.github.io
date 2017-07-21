# Learng Iptalbes

## iptables NAT for two different subnet
192.168.0.0/16

os: debian 8.0

```shell
vim /etc/sysctl.conf

net.ipv4.ip_forward=1
```
eth0: 192.168.16.93
eth1: 10.10.10.1

need to using internet subnet 10.10.10.0/24

```shell
iptables -P FORWAD DROP
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -j SNAT --to 192.168.16.93
iptables -A FORWARD -s 10.10.10.0/24 -j ACCEPT
```


`https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands`

# Iptables Essentials: Common Firewall Rules and Commands
Introduction

Iptables is the software firewall that is included with most Linux distributions by default. This cheat sheet-style guide provides a quick reference to iptables commands that will create firewall rules are useful in common, everyday scenarios. This includes iptables examples of allowing and blocking various services by port, network interface, and source IP address.

How To Use This Guide
If you are just getting started with configuring your iptables firewall, check out our introduction to iptables
Most of the rules that are described here assume that your iptables is set to DROP incoming traffic, through the default input policy, and you want to selectively allow traffic in
Use whichever subsequent sections are applicable to what you are trying to achieve. Most sections are not predicated on any other, so you can use the examples below independently
Use the Contents menu on the right side of this page (at wide page widths) or your browser's find function to locate the sections you need
Copy and paste the command-line examples given, substituting the values in red with your own values
Keep in mind that the order of your rules matter. All of these iptables commands use the -A option to append the new rule to the end of a chain. If you want to put it somewhere else in the chain, you can use the -I option which allows you to specify the position of the new rule (or simply place it at the beginning of the chain by not specifying a rule number).

Note: When working with firewalls, take care not to lock yourself out of your own server by blocking SSH traffic (port 22, by default). If you lose access due to your firewall settings, you may need to connect to it via the console to fix your access. Once you are connected via the console, you can change your firewall rules to allow SSH access (or allow all traffic). If your saved firewall rules allow SSH access, another method is to reboot your server.

Remember that you can check your current iptables ruleset with sudo iptables -S and sudo iptables -L.

Let's take a look at the iptables commands!

Saving Rules
Iptables rules are ephemeral, which means they need to be manually saved for them to persist after a reboot.

Ubuntu

On Ubuntu, the easiest way to save iptables rules, so they will survive a reboot, is to use the iptables-persistent package. Install it with apt-get like this:

sudo apt-get install iptables-persistent
During the installation, you will asked if you want to save your current firewall rules.

If you update your firewall rules and want to save the changes, run this command:

sudo invoke-rc.d iptables-persistent save
CentOS 6 and Older

On CentOS 6 and older—CentOS 7 uses FirewallD by default—you can use the iptables init script to save your iptables rules:

sudo service iptables save
This will save your current iptables rules to the /etc/sysconfig/iptables file.

Listing and Deleting Rules
If you want to learn how to list and delete iptables rules, check out this tutorial: How To List and Delete Iptables Firewall Rules.

Generally Useful Rules
This section includes a variety of iptables commands that will create rules that are generally useful on most servers.

Allow Loopback Connections

The loopback interface, also referred to as lo, is what a computer uses to for network connections to itself. For example, if you run ping localhost or ping 127.0.0.1, your server will ping itself using the loopback. The loopback interface is also used if you configure your application server to connect to a database server with a "localhost" address. As such, you will want to be sure that your firewall is allowing these connections.

To accept all traffic on your loopback interface, run these commands:

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
Allow Established and Related Incoming Connections

As network traffic generally needs to be two-way—incoming and outgoing—to work properly, it is typical to create a firewall rule that allows established and related incoming traffic, so that the server will allow return traffic to outgoing connections initiated by the server itself. This command will allow that:

sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
Allow Established Outgoing Connections

You may want to allow outgoing traffic of all established connections, which are typically the response to legitimate incoming connections. This command will allow that:

sudo iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT
Internal to External

Assuming eth0 is your external network, and eth1 is your internal network, this will allow your internal to access the external:

sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
Drop Invalid Packets

Some network traffic packets get marked as invalid. Sometimes it can be useful to log this type of packet but often it is fine to drop them. Do so with this command:

sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
Block an IP Address
To block network connections that originate from a specific IP address, 15.15.15.51 for example, run this command:

sudo iptables -A INPUT -s 15.15.15.51 -j DROP
In this example, -s 15.15.15.51 specifies a source IP address of "15.15.15.51". The source IP address can be specified in any firewall rule, including an allow rule.

If you want to reject the connection instead, which will respond to the connection request with a "connection refused" error, replace "DROP" with "REJECT" like this:

sudo iptables -A INPUT -s 15.15.15.51 -j REJECT
Block Connections to a Network Interface

To block connections from a specific IP address, e.g. 15.15.15.51, to a specific network interface, e.g. eth0, use this command:

iptables -A INPUT -i eth0 -s 15.15.15.51 -j DROP
This is the same as the previous example, with the addition of -i eth0. The network interface can be specified in any firewall rule, and is a great way to limit the rule to a particular network.

Service: SSH
If you're using a cloud server, you will probably want to allow incoming SSH connections (port 22) so you can connect to and manage your server. This section covers how to configure your firewall with various SSH-related rules.

Allow All Incoming SSH

To allow all incoming SSH connections run these commands:

sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established SSH connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow Incoming SSH from Specific IP address or subnet

To allow incoming SSH connections from a specific IP address or subnet, specify the source. For example, if you want to allow the entire 15.15.15.0/24 subnet, run these commands:

sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established SSH connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow Outgoing SSH

If your firewall OUTPUT policy is not set to ACCEPT, and you want to allow outgoing SSH connections—your server initiating an SSH connection to another server—you can run these commands:

sudo iptables -A OUTPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
Allow Incoming Rsync from Specific IP Address or Subnet

Rsync, which runs on port 873, can be used to transfer files from one computer to another.

To allow incoming rsync connections from a specific IP address or subnet, specify the source IP address and the destination port. For example, if you want to allow the entire 15.15.15.0/24 subnet to be able to rsync to your server, run these commands:

sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 873 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 873 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established rsync connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Service: Web Server
Web servers, such as Apache and Nginx, typically listen for requests on port 80 and 443 for HTTP and HTTPS connections, respectively. If your default policy for incoming traffic is set to drop or deny, you will want to create rules that will allow your server to respond to those requests.

Allow All Incoming HTTP

To allow all incoming HTTP (port 80) connections run these commands:

sudo iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 80 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established HTTP connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow All Incoming HTTPS

To allow all incoming HTTPS (port 443) connections run these commands:

sudo iptables -A INPUT -p tcp --dport 443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 443 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established HTTP connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow All Incoming HTTP and HTTPS

If you want to allow both HTTP and HTTPS traffic, you can use the multiport module to create a rule that allows both ports. To allow all incoming HTTP and HTTPS (port 443) connections run these commands:

sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established HTTP and HTTPS connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Service: MySQL
MySQL listens for client connections on port 3306. If your MySQL database server is being used by a client on a remote server, you need to be sure to allow that traffic.

Allow MySQL from Specific IP Address or Subnet

To allow incoming MySQL connections from a specific IP address or subnet, specify the source. For example, if you want to allow the entire 15.15.15.0/24 subnet, run these commands:

sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 3306 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 3306 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established MySQL connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow MySQL to Specific Network Interface

To allow MySQL connections to a specific network interface—say you have a private network interface eth1, for example—use these commands:

sudo iptables -A INPUT -i eth1 -p tcp --dport 3306 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -o eth1 -p tcp --sport 3306 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established MySQL connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Service: PostgreSQL
PostgreSQL listens for client connections on port 5432. If your PostgreSQL database server is being used by a client on a remote server, you need to be sure to allow that traffic.

PostgreSQL from Specific IP Address or Subnet

To allow incoming PostgreSQL connections from a specific IP address or subnet, specify the source. For example, if you want to allow the entire 15.15.15.0/24 subnet, run these commands:

sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 5432 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 5432 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established PostgreSQL connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow PostgreSQL to Specific Network Interface

To allow PostgreSQL connections to a specific network interface—say you have a private network interface eth1, for example—use these commands:

sudo iptables -A INPUT -i eth1 -p tcp --dport 5432 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -o eth1 -p tcp --sport 5432 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established PostgreSQL connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Service: Mail
Mail servers, such as Sendmail and Postfix, listen on a variety of ports depending on the protocols being used for mail delivery. If you are running a mail server, determine which protocols you are using and allow the appropriate types of traffic. We will also show you how to create a rule to block outgoing SMTP mail.

Block Outgoing SMTP Mail

If your server shouldn't be sending outgoing mail, you may want to block that kind of traffic. To block outgoing SMTP mail, which uses port 25, run this command:

sudo iptables -A OUTPUT -p tcp --dport 25 -j REJECT
This configures iptables to reject all outgoing traffic on port 25. If you need to reject a different service by its port number, instead of port 25, simply replace it.

Allow All Incoming SMTP

To allow your server to respond to SMTP connections, port 25, run these commands:

sudo iptables -A INPUT -p tcp --dport 25 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 25 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established SMTP connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Note: It is common for SMTP servers to use port 587 for outbound mail.

Allow All Incoming IMAP

To allow your server to respond to IMAP connections, port 143, run these commands:

sudo iptables -A INPUT -p tcp --dport 143 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 143 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established IMAP connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow All Incoming IMAPS

To allow your server to respond to IMAPS connections, port 993, run these commands:

sudo iptables -A INPUT -p tcp --dport 993 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 993 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established IMAPS connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow All Incoming POP3

To allow your server to respond to POP3 connections, port 110, run these commands:

sudo iptables -A INPUT -p tcp --dport 110 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 110 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established POP3 connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Allow All Incoming POP3S

To allow your server to respond to POP3S connections, port 995, run these commands:

sudo iptables -A INPUT -p tcp --dport 995 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 995 -m conntrack --ctstate ESTABLISHED -j ACCEPT
The second command, which allows the outgoing traffic of established POP3S connections, is only necessary if the OUTPUT policy is not set to ACCEPT.

Conclusion
That should cover many of the commands that are commonly used when configuring an iptables firewall. Of course, iptables is a very flexible tool so feel free to mix and match the commands with different options to match your specific needs if they aren't covered here.

If you're looking for help determining how your firewall should be set up, check out this tutorial: How To Choose an Effective Firewall Policy to Secure your Servers.

Good luck!

--------------

# A Deep Dive into Iptables and Netfilter Architecture
PostedAugust 20, 2015 62.5k views FIREWALL
Introduction

Firewalls are an important tool that can be configured to protect your servers and infrastructure. In the Linux ecosystem, iptables is a widely used firewall tool that interfaces with the kernel's netfilter packet filtering framework. For users and administrators who don't understand the architecture of these systems, creating reliable firewall policies can be daunting, not only due to challenging syntax, but also because of number of interrelated parts present in the framework.

In this guide, we will dive into the iptables architecture with the aim of making it more comprehensible for users who need to build their own firewall policies. We will discuss how iptables interacts with netfilter and how the various components fit together to provide a comprehensive filtering and mangling system.

What Are IPTables and Netfilter?
The basic firewall software most commonly used in Linux is called iptables. The iptables firewall works by interacting with the packet filtering hooks in the Linux kernel's networking stack. These kernel hooks are known as the netfilter framework.

Every packet that enters networking system (incoming or outgoing) will trigger these hooks as it progresses through the stack, allowing programs that register with these hooks to interact with the traffic at key points. The kernel modules associated with iptables register at these hooks in order to ensure that the traffic conforms to the conditions laid out by the firewall rules.

Netfilter Hooks
There are five netfilter hooks that programs can register with. As packets progress through the stack, they will trigger the kernel modules that have registered with these hooks. The hooks that a packet will trigger depends on whether the packet is incoming or outgoing, the packet's destination, and whether the packet was dropped or rejected at a previous point.

The following hooks represent various well-defined points in the networking stack:

NF_IP_PRE_ROUTING: This hook will be triggered by any incoming traffic very soon after entering the network stack. This hook is processed before any routing decisions have been made regarding where to send the packet.
NF_IP_LOCAL_IN: This hook is triggered after an incoming packet has been routed if the packet is destined for the local system.
NF_IP_FORWARD: This hook is triggered after an incoming packet has been routed if the packet is to be forwarded to another host.
NF_IP_LOCAL_OUT: This hook is triggered by any locally created outbound traffic as soon it hits the network stack.
NF_IP_POST_ROUTING: This hook is triggered by any outgoing or forwarded traffic after routing has taken place and just before being put out on the wire.
Kernel modules that wish to register at these hooks must provide a priority number to help determine the order in which they will be called when the hook is triggered. This provides the means for multiple modules (or multiple instances of the same module) to be connected to each of the hooks with deterministic ordering. Each module will be called in turn and will return a decision to the netfilter framework after processing that indicates what should be done with the packet.

IPTables Tables and Chains
The iptables firewall uses tables to organize its rules. These tables classify rules according to the type of decisions they are used to make. For instance, if a rule deals with network address translation, it will be put into the nat table. If the rule is used to decide whether to allow the packet to continue to its destination, it would probably be added to the filter table.

Within each iptables table, rules are further organized within separate "chains". While tables are defined by the general aim of the rules they hold, the built-in chains represent the netfilter hooks which trigger them. Chains basically determine when rules will be evaluated.

As you can see, the names of the built-in chains mirror the names of the netfilter hooks they are associated with:

PREROUTING: Triggered by the NF_IP_PRE_ROUTING hook.
INPUT: Triggered by the NF_IP_LOCAL_IN hook.
FORWARD: Triggered by the NF_IP_FORWARD hook.
OUTPUT: Triggered by the NF_IP_LOCAL_OUT hook.
POSTROUTING: Triggered by the NF_IP_POST_ROUTING hook.
Chains allow the administrator to control where in a packet's delivery path a rule will be evaluated. Since each table has multiple chains, a table's influence can be exerted at multiple points in processing. Because certain types of decisions only make sense at certain points in the network stack, every table will not have a chain registered with each kernel hook.

There are only five netfilter kernel hooks, so chains from multiple tables are registered at each of the hooks. For instance, three tables have PREROUTING chains. When these chains register at the associated NF_IP_PRE_ROUTING hook, they specify a priority that dictates what order each table's PREROUTING chain is called. Each of the rules inside the highest priority PREROUTING chain is evaluated sequentially before moving onto the next PREROUTING chain. We will take a look at the specific order of each chain in a moment.

Which Tables are Available?
Let's step back for a moment and take a look at the different tables that iptables provides. These represent distinct sets of rules, organized by area of concern, for evaluating packets.

The Filter Table

The filter table is one of the most widely used tables in iptables. The filter table is used to make decisions about whether to let a packet continue to its intended destination or to deny its request. In firewall parlance, this is known as "filtering" packets. This table provides the bulk of functionality that people think of when discussing firewalls.

The NAT Table

The nat table is used to implement network address translation rules. As packets enter the network stack, rules in this table will determine whether and how to modify the packet's source or destination addresses in order to impact the way that the packet and any response traffic are routed. This is often used to route packets to networks when direct access is not possible.

The Mangle Table

The mangle table is used to alter the IP headers of the packet in various ways. For instance, you can adjust the TTL (Time to Live) value of a packet, either lengthening or shortening the number of valid network hops the packet can sustain. Other IP headers can be altered in similar ways.

This table can also place an internal kernel "mark" on the packet for further processing in other tables and by other networking tools. This mark does not touch the actual packet, but adds the mark to the kernel's representation of the packet.

The Raw Table

The iptables firewall is stateful, meaning that packets are evaluated in regards to their relation to previous packets. The connection tracking features built on top of the netfilter framework allow iptables to view packets as part of an ongoing connection or session instead of as a stream of discrete, unrelated packets. The connection tracking logic is usually applied very soon after the packet hits the network interface.

The raw table has a very narrowly defined function. Its only purpose is to provide a mechanism for marking packets in order to opt-out of connection tracking.

The Security Table

The security table is used to set internal SELinux security context marks on packets, which will affect how SELinux or other systems that can interpret SELinux security contexts handle the packets. These marks can be applied on a per-packet or per-connection basis.

Which Chains are Implemented in Each Table?
We have talked about tables and chains separately. Let's go over which chains are available in each table. Implied in this discussion is a further discussion about the evaluation order of chains registered to the same hook. If three tables have PREROUTING chains, in which order are they evaluated?

The following table indicates the chains that are available within each iptables table when read from left-to-right. For instance, we can tell that the raw table has both PREROUTING and OUTPUT chains. When read from top-to-bottom, it also displays the order in which each chain is called when the associated netfilter hook is triggered.

A few things should be noted. In the representation below, the nat table has been split between DNAT operations (those that alter the destination address of a packet) and SNAT operations (those that alter the source address) in order to display their ordering more clearly. We have also include rows that represent points where routing decisions are made and where connection tracking is enabled in order to give a more holistic view of the processes taking place:

Tables↓/Chains→ PREROUTING  INPUT   FORWARD OUTPUT  POSTROUTING
(routing decision)              ✓   
raw ✓           ✓   
(connection tracking enabled)   ✓           ✓   
mangle  ✓   ✓   ✓   ✓   ✓
nat (DNAT)  ✓           ✓   
(routing decision)  ✓           ✓   
filter      ✓   ✓   ✓   
security        ✓   ✓   ✓   
nat (SNAT)      ✓           ✓
As a packet triggers a netfilter hook, the associated chains will be processed as they are listed in the table above from top-to-bottom. The hooks (columns) that a packet will trigger depend on whether it is an incoming or outgoing packet, the routing decisions that are made, and whether the packet passes filtering criteria.

Certain events will cause a table's chain to be skipped during processing. For instance, only the first packet in a connection will be evaluated against the NAT rules. Any nat decisions made for the first packet will be applied to all subsequent packets in the connection without additional evaluation. Responses to NAT'ed connections will automatically have the reverse NAT rules applied to route correctly.

Chain Traversal Order

Assuming that the server knows how to route a packet and that the firewall rules permit its transmission, the following flows represent the paths that will be traversed in different situations:

Incoming packets destined for the local system: PREROUTING -> INPUT
Incoming packets destined to another host: PREROUTING -> FORWARD -> POSTROUTING
Locally generated packets: OUTPUT -> POSTROUTING
If we combine the above information with the ordering laid out in the previous table, we can see that an incoming packet destined for the local system will first be evaluated against the PREROUTING chains of the raw, mangle, and nat tables. It will then traverse the INPUT chains of the mangle, filter, security, and nat tables before finally being delivered to the local socket.

IPTables Rules
Rules are placed within a specific chain of a specific table. As each chain is called, the packet in question will be checked against each rule within the chain in order. Each rule has a matching component and an action component.

Matching

The matching portion of a rule specifies the criteria that a packet must meet in order for the associated action (or "target") to be executed.

The matching system is very flexible and can be expanded significantly with iptables extensions available on the system. Rules can be constructed to match by protocol type, destination or source address, destination or source port, destination or source network, input or output interface, headers, or connection state among other criteria. These can be combined to create fairly complex rule sets to distinguish between different traffic.

Targets

A target is the action that are triggered when a packet meets the matching criteria of a rule. Targets are generally divided into two categories:

Terminating targets: Terminating targets perform an action which terminates evaluation within the chain and returns control to the netfilter hook. Depending on the return value provided, the hook might drop the packet or allow the packet to continue to the next stage of processing.
Non-terminating targets: Non-terminating targets perform an action and continue evaluation within the chain. Although each chain must eventually pass back a final terminating decision, any number of non-terminating targets can be executed beforehand.
The availability of each target within rules will depend on context. For instance, the table and chain type might dictate the targets available. The extensions activated in the rule and the matching clauses can also affect the availability of targets.

Jumping to User-Defined Chains
We should mention a special class of non-terminating target: the jump target. Jump targets are actions that result in evaluation moving to a different chain for additional processing. We've talked quite a bit about the built-in chains which are intimately tied to the netfilter hooks that call them. However, iptables also allows administrators to create their own chains for organizational purposes.

Rules can be placed in user-defined chains in the same way that they can be placed into built-in chains. The difference is that user-defined chains can only be reached by "jumping" to them from a rule (they are not registered with a netfilter hook themselves).

User-defined chains act as simple extensions of the chain which called them. For instance, in a user-defined chain, evaluation will pass back to the calling chain if the end of the rule list is reached or if a RETURN target is activated by a matching rule. Evaluation can also jump to additional user-defined chains.

This construct allows for greater organization and provides the framework necessary for more robust branching.

IPTables and Connection Tracking
We introduced the connection tracking system implemented on top of the netfilter framework when we discussed the raw table and connection state matching criteria. Connection tracking allows iptables to make decisions about packets viewed in the context of an ongoing connection. The connection tracking system provides iptables with the functionality it needs to perform "stateful" operations.

Connection tracking is applied very soon after packets enter the networking stack. The raw table chains and some basic sanity checks are the only logic that is performed on packets prior to associating the packets with a connection.

The system checks each packet against a set of existing connections. It will update the state of the connection in its store if needed and will add new connections to the system when necessary. Packets that have been marked with the NOTRACK target in one of the raw chains will bypass the connection tracking routines.

Available States

Connections tracked by the connection tracking system will be in one of the following states:

NEW: When a packet arrives that is not associated with an existing connection, but is not invalid as a first packet, a new connection will be added to the system with this label. This happens for both connection-aware protocols like TCP and for connectionless protocols like UDP.
ESTABLISHED: A connection is changed from NEW to ESTABLISHED when it receives a valid response in the opposite direction. For TCP connections, this means a SYN/ACK and for UDP and ICMP traffic, this means a response where source and destination of the original packet are switched.
RELATED: Packets that are not part of an existing connection, but are associated with a connection already in the system are labeled RELATED. This could mean a helper connection, as is the case with FTP data transmission connections, or it could be ICMP responses to connection attempts by other protocols.
INVALID: Packets can be marked INVALID if they are not associated with an existing connection and aren't appropriate for opening a new connection, if they cannot be identified, or if they aren't routable among other reasons.
UNTRACKED: Packets can be marked as UNTRACKED if they've been targeted in a raw table chain to bypass tracking.
SNAT: A virtual state set when the source address has been altered by NAT operations. This is used by the connection tracking system so that it knows to change the source addresses back in reply packets.
DNAT: A virtual state set when the destination address has been altered by NAT operations. This is used by the connection tracking system so that it knows to change the destination address back when routing reply packets.
The states tracked in the connection tracking system allow administrators to craft rules that target specific points in a connection's lifetime. This provides the functionality needed for more thorough and secure rules.

Conclusion
The netfilter packet filtering framework and the iptables firewall are the basis for most firewall solutions on Linux servers. The netfilter kernel hooks are close enough to the networking stack to provide powerful control over packets as they are processed by the system. The iptables firewall leverages these capabilities to provide a flexible, extensible method of communicating policy requirements to the kernel. By learning about how these pieces fit together, you can better utilize them to control and secure your server environments.

If you would like to know more about how to choose effective iptables policies, check out this guide.

These guides can help you get started implementing your iptables firewall rules:

How To Set Up a Firewall Using Iptables on Ubuntu 14.04
Iptables Essentials: Common Firewall Rules and Commands
How To Implement a Basic Firewall Template with Iptables on Ubuntu 14.04
How To Set Up an Iptables Firewall to Protect Traffic Between your Servers

----------------

# How To Implement a Basic Firewall Template with Iptables on Ubuntu 14.04
PostedAugust 20, 2015 83.3k views FIREWALL UBUNTU
Introduction

Implementing a firewall is an important step in securing your server. A large part of that is deciding on the individual rules and policies that will enforce traffic restrictions to your network. Firewalls like iptables also allow you to have a say about the structural framework in which your rules are applied.

In this guide, we will construct a firewall that can be the basis for more complex rule sets. This firewall will focus primarily on providing reasonable defaults and establishing a framework that encourages easy extensibility. We will be demonstrating this on an Ubuntu 14.04 server.

Prerequisites
Before you begin, you should have a basic idea of the firewall policies you wish to implement. You can follow this guide to get a better idea of some of the things you should be thinking about.

In order to follow along, you will need to have access to an Ubuntu 14.04 server. We will be using a non-root user configured with sudo privileges throughout this guide. You can learn how to configure this type of user in our Ubuntu 14.04 initial server setup guide.

When you are finished, continue below.

Installing the Persistent Firewall Service
To get started, you will need to install the iptables-persistent package if you have not done so already. This will allow us to save our rule sets and have them automatically applied at boot:

sudo apt-get update
sudo apt-get install iptables-persistent
During the installation, you'll be asked whether you want to save your current rules. Say "yes" here. We will be editing the generated rules files momentarily.

A Note About IPv6 in this Guide
Before we get started, we should talk briefly about IPv4 vs IPv6. The iptables command only handles IPv4 traffic. For IPv6 traffic, a separate companion tool called ip6tables is used. The rules are stored in separate tables and chains. For iptables-persistent, the IPv4 rules are written to and read from /etc/iptables/rules.v4 and the IPv6 rules are kept in /etc/iptables/rules.v6.

This guide assumes that you are not actively using IPv6 on your server. If your services do not leverage IPv6, it is safer to block access entirely, as we will be doing in this article.

Implementing the Basic Firewall Policy (The Quick Way)
For the sake of getting up and running as quickly as possible, we'll show you how to edit the rules file directly to copy and paste the finished firewall policy. Afterwards, we will explain the general strategy and show you how these rules could be implemented using the iptables command instead of modifying the file.

To implement our firewall policy and framework, we will be editing the /etc/iptables/rules.v4 and /etc/iptables/rules.v6 files. Open the rules.v4 file in your text editor with sudo privileges:

sudo nano /etc/iptables/rules.v4
Inside, you will see a file that looks something like this:

/etc/iptables/rules.v4
# Generated by iptables-save v1.4.21 on Tue Jul 28 13:29:56 2015
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
# Completed on Tue Jul 28 13:29:56 2015
Replace the contents with:

/etc/iptables/rules.v4
*filter
# Allow all outgoing, but drop incoming and forwarding packets by default
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# Custom per-protocol chains
:UDP - [0:0]
:TCP - [0:0]
:ICMP - [0:0]

# Acceptable UDP traffic

# Acceptable TCP traffic
-A TCP -p tcp --dport 22 -j ACCEPT

# Acceptable ICMP traffic

# Boilerplate acceptance policy
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A INPUT -i lo -j ACCEPT

# Drop invalid packets
-A INPUT -m conntrack --ctstate INVALID -j DROP

# Pass traffic to protocol-specific chains
## Only allow new connections (established and related should already be handled)
## For TCP, additionally only allow new SYN packets since that is the only valid
## method for establishing a new TCP connection
-A INPUT -p udp -m conntrack --ctstate NEW -j UDP
-A INPUT -p tcp --syn -m conntrack --ctstate NEW -j TCP
-A INPUT -p icmp -m conntrack --ctstate NEW -j ICMP

# Reject anything that's fallen through to this point
## Try to be protocol-specific w/ rejection message
-A INPUT -p udp -j REJECT --reject-with icmp-port-unreachable
-A INPUT -p tcp -j REJECT --reject-with tcp-reset
-A INPUT -j REJECT --reject-with icmp-proto-unreachable

# Commit the changes
COMMIT

*raw
:PREROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
COMMIT

*security
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT

*mangle
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
COMMIT
Save and close the file.

You can test the file for syntax errors by typing this command. Fix any syntax errors that this reveals before continuing:

sudo iptables-restore -t /etc/iptables/rules.v4
Next, open the /etc/iptables/rules.v6 file to modify the IPv6 rules:

sudo nano /etc/iptables/rules.v6
We can block all IPv6 traffic by replacing the contents of the file with the below configuration:

/etc/iptables/rules.v6
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]
COMMIT

*raw
:PREROUTING DROP [0:0]
:OUTPUT DROP [0:0]
COMMIT

*nat
:PREROUTING DROP [0:0]
:INPUT DROP [0:0]
:OUTPUT DROP [0:0]
:POSTROUTING DROP [0:0]
COMMIT

*security
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]
COMMIT

*mangle
:PREROUTING DROP [0:0]
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]
:POSTROUTING DROP [0:0]
COMMIT
Save and close the file.

To test this file for syntax errors, we can use the ip6tables-restore command with the -t option:

sudo ip6tables-restore -t /etc/iptables/rules.v6
When both rules files report no syntax errors, you can apply the rules within by typing:

sudo service iptables-persistent reload
This will immediately implement the policy outlined in your files. You can verify this by listing the iptables rules currently in use:

sudo iptables -S
sudo ip6tables -S
These firewall rules will be re-applied at each boot. Test to make sure that you can still log in and that all other access is blocked off.

An Explanation of Our General Firewall Strategy
In the basic firewall we've constructed with the above rules, we've created an extensible framework that can be easily adjusted to add or remove rules. For IPv4 traffic, we're mainly concerned with the INPUT chain within the filter table. This chain will process all packets destined for our server. We've also allowed all outgoing traffic and denied all packet forwarding, which would only be appropriate if this server were acting as a router for other hosts. We accept packets in all of the other tables since we are only looking to filter packets in this guide.

In general, our rules set up a firewall that will deny incoming traffic by default. We then go about creating exceptions for the services and traffic types we wish to exclude from this policy.

In the main INPUT chain, we've added some generic rules for traffic that we are confident will always be handled the same way. For instance, we always want to deny packets that are deemed "invalid" and we will always want to allow traffic on the local loopback interface and data associated with an established connection.

Afterwards, we match traffic based on the protocol it is using and shuffle it to a protocol-specific chain. These protocol-specific chains are meant to hold rules that match and allow traffic for specific services. In this example, the only service we allow is SSH in our TCP chain. If we were offering another service, like an HTTP(S) server, we could add exceptions that here as well. These chains will be the focus of most of your customization.

Any traffic that does not match the generic rules or the service rules in the protocol-specific are handled by the last few rules in the INPUT chain. We have set the default policy to DROP for our firewall, which will deny packets that fall through our rules. However, the rules at the end of the INPUT chain reject packets and send a message to the client that mimics how the server would respond if there were no service running on that port.

For IPv6 traffic, we simply drop all traffic. Our server is not using this protocol, so it is safest to not engage with the traffic at all.

(Optional) Update Nameservers
Blocking all IPv6 traffic can interfere with how your server resolves things on the Internet. For example, this can affect how you use APT.

If you get errors like this when you try to run apt-get update:

Error
Err http://security.ubuntu.com trusty-security InRelease

Err http://security.ubuntu.com trusty-security Release.gpg
  Could not resolve 'security.ubuntu.com'

. . .
You should follow this section to get APT working again.

First, set your nameservers to outside nameservers. This example uses Google's nameservers. Open /etc/network/interfaces for editing:

sudo nano /etc/network/interfaces
Update the dns-nameservers line as shown:

/etc/network/interfaces
. . .
iface eth0 inet6 static
        address 2604:A880:0800:0010:0000:0000:00B2:0001
        netmask 64
        gateway 2604:A880:0800:0010:0000:0000:0000:0001
        autoconf 0
        dns-nameservers 8.8.8.8 8.8.4.4
Refresh your network settings:

sudo ifdown eth0 && sudo ifup eth0
The expected output is:

Output
RTNETLINK answers: No such process
Waiting for DAD... Done
Next, create a new firewall rule to force IPv4 when it's available. Create this new file:

sudo nano /etc/apt/apt.conf.d/99force-ipv4
Add this single line to the file:

/etc/apt/apt.conf.d/99force-ipv4
Acquire::ForceIPv4 "true";
Save and close the file. Now you should be able to use APT.

Implementing our Firewalls Using the IPTables Command
Now that you understand the general idea behind the policy we built, we will walk through how you could go about creating those rules using iptables commands. We will end up with the same rules that we specified above but we will create our policies by adding rules iteratively. Because iptables applies each of the rules immediately, rule ordering is very important (we leave the rules that deny packets until the end).

Reset your Firewall

We will start by resetting our firewall rules so that we can see how policies can be built from the command line. You can flush all of your rules by typing:

sudo service iptables-persistent flush
You can verify that your rules are reset by typing:

sudo iptables -S
You should see that the rules in the filter table are gone and that the default policy is set to ACCEPT on all chains:

output
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
Create Protocol-Specific Chains

We will start by creating all of our protocol-specific chains. These will be used to hold the rules that create exceptions to our deny policy for services we want to expose. We will create one for UDP traffic, one for TCP, and one for ICMP:

sudo iptables -N UDP
sudo iptables -N TCP
sudo iptables -N ICMP
We can go right ahead and add the exception for SSH traffic. SSH uses TCP, so we will add a rule to accept TCP traffic destined for port 22 to the TCP chain:

sudo iptables -A TCP -p tcp --dport 22 -j ACCEPT
If we wanted to add additional TCP services, we could do that now by repeating the command with the port number replaced.

Create General Purpose Accept and Deny Rules

In the INPUT chain, where all incoming traffic begins filtering, we need to add our general purpose rules. These are some common sense rules that set the baseline for our firewall by accepting traffic that's low risk (local traffic and traffic that's associated with connections we've already checked) and dropping traffic that is clearly not useful (invalid packets).

First, we will create an exception to accept all traffic that is part of an established connection or is related to an established connection:

sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
This rule uses the conntrack extension, which provides internal tracking so that iptables has the context it needs to evaluate packets as part of larger connections instead of as a stream of discrete, unrelated packets. TCP is a connection-based protocol, so an established connection is fairly well-defined. For UDP and other connectionless protocols, established connections refer to traffic that has seen a response (the source of the original packet will the destination of the response packet, and vice versa). A related connection refers to a new connection that has been initiated in association with an existing connection. The classic example here is an FTP data transfer connection, which would be related to the FTP control connection that has already been established.

We want to also allow all traffic originating on the local loopback interface. This is traffic generated by the server and destined for the server. It is used by services on the host to communicate with one another:

sudo iptables -A INPUT -i lo -j ACCEPT
Finally, we want to deny all invalid packets. Packets can be invalid for a number of reasons. They may refer to connections that do not exist, they may be destined for interfaces, addresses, or ports that do not exist, or they may simply be malformed. In any case, we will drop all invalid packets since there is no proper way to handle them and because they could represent malicious activity:

sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
Creating the Jump Rules to the Protocol-Specific Chains

So far, we have created some general rules in the INPUT chain and some rules for specific acceptable services within our protocol-specific chains. However, right now, traffic comes into the INPUT chain and has no way of reaching our protocol-specific chains.

We need to direct traffic in the INPUT chain into the appropriate protocol-specific chains. We can match on protocol type to send it to the right chain. We will also ensure that the packet represents a new connection (any established or related connections should already be handled earlier). For TCP packets, we will add the additional requirement that the packet is a SYN packet, which is the only valid type to start a TCP connection:

sudo iptables -A INPUT -p udp -m conntrack --ctstate NEW -j UDP
sudo iptables -A INPUT -p tcp --syn -m conntrack --ctstate NEW -j TCP
sudo iptables -A INPUT -p icmp -m conntrack --ctstate NEW -j ICMP
Reject All Remaining Traffic

If a packet that was passed to a protocol-specific chain did not match any of the rules within, control will be passed back to the INPUT chain. Anything that reaches this point should not be allowed by our firewall.

We will deny the traffic using the REJECT target, which sends a response message to the client. This allows us to specify the outbound messaging so that we can mimic the response that would be given if the client tried to send packets to a regular closed port. The response is dependent on the protocol used by the client.

Attempting to reach a closed UDP port will result in an ICMP "port unreachable" message. We can imitate this by typing:

sudo iptables -A INPUT -p udp -j REJECT --reject-with icmp-port-unreachable
Attempting to establish a TCP connection on a closed port results in a TCP RST response:

sudo iptables -A INPUT -p tcp -j REJECT --reject-with tcp-reset
For all other packets, we can send an ICMP "protocol unreachable" message to indicate that the server doesn't respond to packets of that type:

sudo iptables -A INPUT -j REJECT --reject-with icmp-proto-unreachable
Adjusting Default Policies

The last three rules we added should handle all remaining traffic in the INPUT chain. However, we should set the default policy to DROP as a precaution. We should also set this policy in the FORWARD chain if this server isn't configured as a router to other machines:

sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
Warning
With your policy set to DROP, if you clear your iptables with sudo iptables -F, your current SSH connection will be dropped! Flushing with sudo iptables-persistent flush is a better way to clear rules since it will reset the default policy as well.
To match our IPv6 policy of dropping all traffic, we can use the following ip6tables commands:

sudo ip6tables -P INPUT DROP
sudo ip6tables -P FORWARD DROP
sudo ip6tables -P OUTPUT DROP
This should replicate our rules set fairly closely.

Saving IPTables Rules

At this point, you should test your firewall rules and make sure they cover the block the traffic you want to keep out while not hindering your normal access. Once you are satisfied that your rules are behaving correctly, you can save them so that they will be automatically be applied to your system at boot.

Save your current rules (both IPv4 and IPv6) by typing:

sudo service iptables-persistent save
This will overwrite your /etc/iptables/rules.v4 and /etc/iptables/rules.v6 files with the policies you crafted on the command line.

Conclusion
By following this guide, either by pasting your firewall rules directly into the configuration files or by manually applying and saving them on the command line, you have created a good starting firewall configuration. You will have to add the individual rules to allow access to the services you want to make available.

The framework established in this guide should allow you to easily make adjustments and can help clarify your existing policies. Check out some of our other guides to see how to build out your firewall policy with some popular services:

Iptables Essentials: Common Firewall Rules and Commands
How To Set Up an Iptables Firewall to Protect Traffic Between your Servers
How To Forward Ports through a Linux Gateway with Iptables
How To Test your Firewall Configuration with Nmap and Tcpdump

-------------



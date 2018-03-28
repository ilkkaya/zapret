zapret v.18

What is it for?
-----------------

To bypass the web site blocking http.

How it works
----------------

ISPs in DPI have gaps. They happen from the fact that the DPI rules are written for
ordinary user programs, omitting all possible cases that are acceptable by standards.
This is done for simplicity and speed. There is no sense in catching hackers, which are 0.01%
because all the same these locks are quite easy even for ordinary users.

Some DPIs can not recognize the http request if it is divided into TCP segments.
For example, a query like "GET / HTTP / 1.1 \ r \ nHost: kinozal.tv ......"
we send in 2 parts: first goes "GET", then "/ HTTP / 1.1 \ r \ nHost: kinozal.tv .....".
Other DPI stumbles when the "Host:" header is written in a different register: for example, "host:".
In some cases, an additional white space is added after the method: "GET /" => "GET /"
or adding a dot at the end of the host name: "Host: kinozal.tv."

How to implement this in practice on a linux system
-----------------------------------------------

How do I get the system to split a request into parts? You can parse the entire TCP session
through transparent proxy, and you can replace the tcp window size field with the first incoming TCP packet with SYN, ACK.
Then the client will think that the server has set a small window size for it and the first data segment
will send no more than the specified length. In subsequent packages we will not change anything.
The further behavior of the system in choosing the size of the packets sent depends on the implemented
in it an algorithm. Experience shows that linux first packet always sends no more than specified
in the window size of the length, the remaining packets until some time sends no more than max (36, specified_size).
After a certain number of packages, the window scaling mechanism is triggered and starts
the scaling factor is taken into account, the size of the packets becomes no more than max (36, specified_ scale <factor_factor).
Not very elegant behavior, but since we do not influence the size of the packet inbound,
and the amount of data received via http is usually much higher than the amount sent, then visually
there will be only small delays.
Windows behaves in a similar case is much more predictable. The first segment
leaves the specified length, then window size changes depending on the value,
which is sent in new tcp packages. That is, the speed is almost immediately restored
up to a possible maximum.

To intercept a packet with SYN, ACK does not present any complexity by means of iptables.
However, the ability to edit packets in iptables is severely limited.
Just so you can not change window size with standard modules.
For this we use the NFQUEUE tool. This tool allows
Send packets for processing to processes running in user mode.
The process by accepting the package can change it, which is what we need.

iptables -t raw -I PREROUTING -p tcp --sport 80 --tcp-flags SYN, ACK SYN, ACK -j NFQUEUE --queue-num 200 --queue-bypass

Will give the necessary packages to the process, listening on the queue with the number 200.
It will override window size. PREROUTING will catch as packets addressed to the host itself,
and routed packets. That is, the solution works the same way both on the client,
and on the router. On a router based on PC or on the basis of OpenWRT.
In principle, this is enough.
However, with such an impact on TCP there will be a slight delay.
To avoid touching hosts that are not blocked by the provider, you can make this move.
Create a list of blocked domains or download it from rublacklist.
Resolve all domains in ipv4 addresses. Drive them into ipset with the name "zapret".
Add to rule:

iptables -t raw -I PREROUTING -p tcp --sport 80 --tcp-flags SYN, ACK SYN, ACK -m set --match-set zapret src -j NFQUEUE --queue-num 200 --queue-bypass

Thus, the impact will be only on ip addresses related to blocked sites.
The list can be updated through cron every few days.
If you update via rublacklist, it will take quite a while. More than an hour. But resources
This process does not take away, so it will not cause any problems, especially if the system
works constantly.

If the DPI does not bypass the segmentation of the query into segments, then sometimes a change occurs
"Host:" to "host:". In this case, we may not need to change the window size, so the chain
PREROUTING we do not need. Instead of it we hang on outgoing packets in the chain POSTROUTING:

iptables -t mangle -I POSTROUTING -p tcp --dport 80 -m set --match-set zapret dst -j NFQUEUE --queue-num 200 --queue-bypass

In this case, as additional points are possible. DPI can only catch the first http request, ignoring
subsequent requests in a keep-alive session. Then we can reduce the load by percents, refusing to process unnecessary packets.

iptables -t mangle -I POSTROUTING -p tcp -dport 80 -m connbytes -connbytes-dir = original -connbytes-mode = packets -connbytes 1: 5 -m set --match-set zapret dst -j NFQUEUE --queue-num 200 --queue-bypass

It happens that the provider monitors the entire HTTP session with keep-alive requests. In this case
it is not enough to restrict the TCP window when establishing a connection. It is necessary to send individual
TCP segments each new request. This task is solved through full proxy traffic through
transparent proxy (TPROXY or DNAT). TPROXY does not work with connections originating from the local system,
so this solution is only applicable on the router. DNAT works with local connections,
but there is a danger of entering infinite recursion, so the daemon starts under a separate user,
and for this user DNAT is disabled via "-m owner". Full proxying requires more resources
processor than manipulation with outgoing packets without the reconstruction of the TCP connection.

iptables -t nat -I PREROUTING -p tcp --dport 80 -j DNAT --to 127.0.0.1:1188
iptables -t nat -I OUTPUT -p tcp --dport 80 -m owner! --uid-owner tpws -j DNAT --to 127.0.0.1:1188

nfqws
-----

This program is a packet modifier and a NFQUEUE queue handler.
It takes the following parameters:
 --daemon; demonize the prog
 --qnum = 200; queue number
 --wsize = 4; change tcp window size to the specified size
 --hostcase; Change the "Host:" header register to "host:" by default.
 --hostnospace; Remove the space after "Host:" and move it to the end of the value of "User-Agent:" to save the length of the package
 --hostspell = HoST; exact writing of the Host header (can be "HOST" or "HoSt"). automatically includes --hostcase
The manipulation parameters can be combined in any combination.

tpws
-----

tpws is a transparent proxy.
 --bind-addr; on which address to listen. can be ipv4 or ipv6 address. if not specified, it listens on all ipv4 and ipv6 addresses
 --port = <port>; on which port to listen
 --daemon; demonize the prog
 --user = <username>; change the uid of the process
 --split-http-req = method | host; A method of separating http requests into segments: about a method (GET, POST) or near the Host header
 --split-pos = <offset>; Divide all sends into segments in the specified position. If the send is longer than 8Kb (receive buffer size), each block will be divided into 8Kb.
 --hostcase; change the "Host:" header register. by default to "host:".
 --hostspell = HoST; exact writing of the Host header (can be "HOST" or "HoSt"). automatically includes --hostcase
 --hostdot; adding a dot after the host name: "Host: kinozal.tv."
 --hosttab; adding tab after the host name: "Host: kinozal.tv \ t"
 --hostnospace; Remove the space after "Host:"
 --methodspace; add a space after the method: "GET /" => "GET /"
 - methodeol; add a line feed before the method: "GET /" => "\ r \ nGET /"
 --unixeol; convert 0D0A to 0A and use everywhere 0A
The manipulation parameters can be combined in any combination.
There are exceptions: split-pos replaces split-http-req. hostdot and hosttab are mutually exclusive.
 
Providers
----------

mns.ru: you need to replace window size with 3. mns.ru removes blocked domains from issuing your DNS servers. we change to third-party. uplink westcall ban on the IP address from the list of ILV where there is https

at-home.ru: with the default connection, everything was blocked by IP. after ordering an external IP (static NAT) are hacked by IP https addresses
 To work around the DPI, the windows size is replaced by 3, but instability and hanging have been observed. The best way is to split the request around the method during the entire http session.
 In https the certificate is substituted. If you are all blocked by IP, then there is no way, except proxy port 80 similar to 443.

beeline (corbina): you need to replace the register "Host:" throughout the entire http session. Since some time, "host" does not work, but other registers of letters work.

dom.ru: proxy HTTP sessions via tpws with the replacement of the register "Host:" and the separation of TCP segments on the host: "Host:".
  Ahtung! Domra blocks all subdomains of the blocked domain. IP addresses of all possible subdomains can not be found from the registry
  locks, so if suddenly on some site comes out a blocking banner, then go to the console firefox, the network tab.
  Download the site and see where the redirect goes. Then enter the domain in zapret-hosts-user.txt. For example, there are kinozal.tv
  2 requested subdomains: s.kinozal.tv and st.kinozal.tv with different IP addresses.
  Domra intercepts DNS requests and sticks in his false answer. This is bypassed through the false-response drop by means of iptables by the presence of the IP address of the stub or via dnscrypt.

sknt.ru: checked the work with tpws with the parameter "--split-http-req = method". Perhaps nfqueue will work as long as the possibilities
  check no

Rostelecom / tkt: helps to split the http request into segments, the settings of mns.ru are suitable
  TKT was bought by the Rostelecom, the filtration of Rostelecom is used.
  Since the DPI does not discard the incoming session, but only sticks in its packet, which comes before the response from the real server,
  blockages also do without the use of "heavy artillery" the following rule:
  iptables -t raw -I PREROUTING -p tcp --sport 80 -m string --hex-string "| 0D0A | Location: http://95.167.13.50 " --algo bm -j DROP --from 40 --to 200

tiera: Requires a split of http requests for the entire session.

Other providers
-----------------

The first step is to find out whether your DNS provider will replace it.
Look into what the blocked hosts from your ISP and through some web net tools can resolve, which can be nested a lot. Compare.
If the answers are different, then try to resolve the same hosts from the DNS server 8.8.8.8 through your ISP.
If the answer from 8.8.8.8 is normal, change the DNS. If the answer is abnormal, then the provider intercepts requests for third-party DNS.
Use dnscrypt.

Next, you need to find out which DPI workaround is working on your ISP.
The script https://github.com/ValdikSS/blockcheck will help you with this .
Choose which daemon you will use: nfqws or tpws.
Prepare the iptables rules manually for your case, execute them.
Run the daemon with the required parameters manually.
Check whether it works.
When you find a working variant, edit the init script for your system.
Uncomment ISP = custom. Add your code to the places "# PLACEHOLDER" by analogy with the sections for other providers for the found working combination.
For openwrt, put your code in /etc/firewall.user by analogy with the finished scripts.

Ways to get a list of blocked IPs
-------------------------------------------

1) Enter the blocked domains in ipset / zapret-hosts-user.txt and run ipset / get_user.sh
On the output you will receive ipset / zapret-ip-user.txt with IP addresses.

2) ipset / get_reestr.sh gets a list of domains from rublacklist and then resoles them to ip addresses
into the ipset / zapret-ip.txt file. In this list there are ready IP addresses, but judging by everything they are there exactly in the form,
which makes the registry RosKomPozor. Addresses can change, shame does not have time to update them, and providers rarely
banyat by IP: instead they hack http requests with "bad" header "Host:" regardless
from the IP address. Therefore, the script solves everything itself, although it takes a lot of time.
An additional requirement is the amount of memory in / tmp to store the downloaded file, the size of which
a few MB and continues to grow. On routers, openwrt / tmp is tmpfs, that is ramdisk.
In the case of a router with 32 MB of memory, it will not be enough, and there will be problems. In this case, use
the following script.

3) ipset / get_anizapret.sh. Fast and no load on the router receives a sheet with antizapret.prostovpn.org.

4) ipset / get_combined.sh. for providers that block by IP https, and the rest by DPI. IP https is entered in ipset ipban, the rest in ipset zapret.
Since a large list of ILVs is downloaded, the requirements for the location in / tmp are similar to 2)

All variants of the scripts considered automatically create and fill ipset.
Options 2-4 additionally cause option 1.

On routers it is not recommended to call these scripts more often than once in 2 days,
either to the internal flash memory of the router, or in the case of extroot - to the flash drive.
In both cases, too frequent recording can kill the flash drive, but if this happens to the internal
flash memory, then you just kill the router.

Forced update ipset executes the ipset / create_ipset.sh script

You can enter a list of domains in ipset / zapret-hosts-user-ipban.txt. Their ip addresses will be placed
in a separate ipset "ipban". It can be used to force all
connections to transparent proxy "redsocks" or to a VPN.


Installation example on debian 7
----------------------------
Debian 7 initially contains the kernel 3.2. It does not know how to do DNAT on localhost.
Of course, you can not bind tpws to 127.0.0.1 and replace iptables with "DNAT 127.0.0.1" with "REDIRECT",
but it is better to install a more recent kernel. It is in a stable repository:
 apt-get update
 apt-get install linux-image-3.16
Install packages:
 apt-get update
 apt-get install libnetfilter-queue-dev ipset curl
Copy the "zapret" directory to / opt.
Collect nfqws:
 cd / opt / zapret / nfq
 make
Collect tpws:
 cd / opt / zapret / tpws
 make
Copy /opt/zapret/init.d/debian7/zapret to /etc/init.d.
In /etc/init.d/zapret, select the "ISP" parameter. Depending on it, the necessary rules will be applied.
In the same place, select the SLAVE_ETH parameter corresponding to the name of the internal network interface.
Enable autostart: chkconfig zapret on
(optional) Manually the first time to get a new list of ip addresses: /opt/zapret/ipset/get_antizapret.sh
To pause the update job for a worksheet:
 crontab -e
 Create a line "0 12 * * * / 2 /opt/zapret/ipset/get_antizapret.sh". This means at 12:00 every 2 days to update the list.
Start the service: service zapret start
Try to go somewhere: http://ej.ru , http://kinozal.tv , http://grani.ru .
If it does not work, then stop the zapret service, add the rule to iptables manually,
run nfqws in the terminal under the root with the required parameters.
Try to connect to blocked sites, watch the output of the program.
If there is no response, then most likely the wrong queue number is specified or there is no ip destination in ipset.
If there is a reaction, but the blockage is not bypassed, then the bypass parameters chosen are not correct, or this means
Does not work in your case on your ISP.
Nobody said that it would work everywhere.
Try to remove the dump in wireshark or "tcpdump -vvv -X host <ip>", see if the first
The TCP segment goes short and the "Host:" register changes.

ubuntu 12.14
------------

There is a ready config for upstart: zapret.conf. It needs to be copied to / etc / init and configured in the same way as debian.
Starting the service: "start zapret"
Stop service: "stop zapret"
Show Posts: cat /var/log/upstart/zapret.log
Ubuntu 12, as well as debian 7, is equipped with the kernel 3.2. See the comment in the "debian 7" section.

ubuntu 16, debian 8
------------------

The process is similar to debian 7, however, it is required to register init scripts in systemd after copying them to /etc/init.d.
By default, lsb-core may not be installed.
apt-get update
apt-get -no-install-recommends install lsb-core

install: / usr / lib / lsb / install_initd zapret
remove: / usr / lib / lsb / remove_initd zapret
start: sytemctl start zapret
stop: systemctl stop zapret
status, output messages: systemctl status zapret

Other linux systems
--------------------

There are several basic systems for starting services: sysvinit, upstart, systemd.
The setting depends on the system used in your distribution.
A typical strategy is to find a script or configuration for starting other services and write your own by analogy,
if necessary, reading the documentation on the launch system.
The necessary commands can be taken from the proposed scripts.


Firewalls
---------

If you use some kind of firewall management system, then it can enter into conflict
with an existing startup script. In this case, the rules for iptables must be screwed
to your firewall separately from the startup script of tpws or nfqws.
This is how the question is solved in the case of openwrt, since there is a firewall management system.
When reapplying the rules, it could break the iptables settings made by the script from init.d.

What to do with openwrt / LEDE
-------------------------

Install additional packages:
opkg update
opkg install iptables-mod-extra iptables-mod-nfqueue iptables-mod-filter iptables-mod-ipopt ipset curl bind-tools
(for new LEDE) opkg install kmod-ipt-raw

The most important difficulty is compiling programs in C. This can be done on linux x64 using the SDK, which
can be downloaded from the official website openwrt or LEDE. But the process of cross compiling is always a challenge.
It's not enough to run make on a traditional linux system.
Therefore, binaries have ready-made static binaries for all the most common architectures.
Static assembly means that the binaric does not depend on the type libc (glibc, uclibc or musl) and the presence of installed so - it can be used immediately.
If only the type of CPU would fit. ARM and MIPS have several versions. Find the option that works on your system.
Most likely there is one. If not, you will have to collect it yourself.

Copy the directory "zapret" to / opt to the router.
Copy the running binaries nfqws to / opt / zapret / nfq, tpws to / opt / zapret / tpws.
Copy /opt/zapret/init.d/zapret to /etc/init.d.
In /etc/init.d/zapret, select the "ISP" parameter. Depending on it, the necessary rules will be applied.
/etc/init.d/zapret enable
/etc/init.d/zapret start
Depending on your provider, make the necessary entries in /etc/firewall.user.
/etc/init.d/firewall restart
View through iptables-L or through the luci tab "firewall" whether the necessary rules have appeared.
To pause the update job for a worksheet:
 crontab -e
 Create a line "0 12 * * * / 2 /opt/zapret/ipset/get_antizapret.sh". This means at 12:00 every 2 days to update the list.

Bypass https locking
----------------------

As a rule, tricks with DPI do not help to bypass https.
It is necessary to redirect traffic through an external host.
It is suggested to use a transparent redirect through socks5 via iptables + redsocks, or iptables + iproute + openvpn.
The configuration of the version with redsocks on openwrt is described in https.txt.

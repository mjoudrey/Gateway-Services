# Gateway-Services

Linux bootup config templates for DNS, DHCP, UFW/iptables, Squid, and Webmin using **VMware Workstation 15**.

# Topology 
![enter image description here](https://i.imgur.com/cIVlR2i.jpg)

# Before we begin      
|Static IP Addressing| 
|-|

Although most changes only happen within the host-only virtual LAN - which remains static most of the tutorial, you will want to account for when changing physical locations with different bridged-network IP addresses. These files include anything involving static IP referencing. 
```
Example 1 - /etc/network/interfaces
Example 2 - /etc/netplan/50-cloud-init.yaml
```
Depending on the scope of the network, there may be other files requiring your attention for deployments. 

With Ubuntu 18.04, most recommend only editing *50-cloud-init.yaml* for static IP, however, interfaces file alone worked for me.
|Hostname configuration| 
|-|
You can set or verify hostname settings at any point using 
```
> hostnamectl set-hostname DC1.markj.tech 
> hostnamectl
```
If you used Ubuntu Server live installation package (*ubuntu-18.04-live-server-amd64.iso*), you must edit **/etc/cloud/*cloud.cfg*** and set *perserve_hostname* to *true*.

![enter image description here](https://i.imgur.com/NfSfr4K.gif)

# Network Interface Cards
For the purposes of this Gateway Server, we will install two NICs inside of the virtual machine to prepare for DHCP, and other potential services. This will allow us to have further segmentation of WAN traffic and local-only traffic. I set the two virtual NICs as **Bridged** (connecting to the host machine's currently active LAN) and **Host-only** (connecting to a private virtual NIC located only inside of the host-machine and other virtual machines with the same host-only NIC interface active)
 
 |Virtual Network Editor - Host-Only settings| 
|-|
![enter image description here](https://i.imgur.com/KdIEif6.gif)

After installation was complete of the Ubuntu & CentOS, you would make appropriate changes to the interface config files to allow for static IP addresses for the Bridged(Outside) interface & Host-Only(virtual LAN) interface.

#### NIC 1 - Outside Connected
```
auto lo 
iface lo inet loopback
auto ens33
    iface ens33 inet static
    address 192.168.2.9
    netmask 255.255.255.0
```

#### NIC 2 - Virtual LAN
```
auto ens38
iface ens38 inet static
    address 10.0.0.250
    netmask 255.255.255.0
    dns-search markj.tech
    dns-nameserver 10.0.0.250
```
### ![#0000FF](https://placehold.it/15/0000FF/000000?text=+)  DNS -- Bind9 Installation & Configuration  ![#0000FF](https://placehold.it/15/0000FF/000000?text=+) 

********************************************************************
For DNS, bind9 is a staple DNS service many turn to for running their domain services.

Before we begin, you will want to restart networking services after making changes to your interfaces, sometimes this requires a **sudo reboot**.

Simply run 
```
> sudo apt-get update
> sudo apt-get upgrade
> sudo apt-get install bind9 bind9utils
```
This will create the directory */etc/bind/*. If you do not have your static IPs configured, now is the time.

If you haven't configured your hostname for your DNS server - now is the time. 

The format of the hostname is: ***servername.domain.TLD***

The command below is an example
```
> sudo hostnamectl set-hostname DCserver.techservices.com
```
You can confirm the hostname change by typing **hostnamectl** or **hostname**.

Open /etc/bind/named.conf.local file using your favorite text editor. This is where you will create your main bind9 configuration pointer references for your forward and reverse zones settings. This is also where you must make your decision of your domain and top-level domain(TLD). 

You will notice in my dns repo directory, in the *named.conf.local* file I used markj.tech as my domain and TLD. You can use this file as a template for creating your own bind9 domain.
```
zone "markj.tech" IN {
type master;
file "/etc/bind/forward.markj.tech";
};
 
zone "0.0.10.in-addr.arpa" IN {
type master;
file "/etc/bind/reverse.markj.tech";
};
```
Be aware: The changes you will need to make are:

 - First zone - Change the domain
 - Second zone - That is the network IP using a /24  CIDR, but in reverse and missing the last octet. If your server had an IP of 192.168.34.4, your first line would be:
```
zone "34.168.192.in-addr.arpa" IN {
```
You now want to begin creating your forward and reverse zone files. These are the files that tell Bind9 how to redirect IPs to domains(forward) and domains to IPs(reverse).

|*Forward and Reverse Zone Files*| 
|-|
Below are the forward zone and reverse zone config files. Use it as reference but be sure to maintain syntax. Notice the use of semi-colons, and periods at the end of the TLDs.

|Forward| 
|-|
![enter image description here](https://i.imgur.com/Jp6iGPC.gif)

|Reverse| 
|-|
![enter image description here](https://i.imgur.com/cq1F9eh.gif)

### ![#00FF00](https://placehold.it/15/00FF00/000000?text=+)  DHCP -- isc-dhcp-server Installation  & Configuration  ![#00FF00](https://placehold.it/15/00FF00/000000?text=+) 

********************************************************************
Simply run 
```
> sudo apt-get install isc-dhcp-server
```
After install, change your default NIC by editing the **/etc/default/*isc-dhcp-server*** file. In this scenario I want the DHCP server to exist on my host-only virtual interface - which is ens38.
![enter image description here](https://i.imgur.com/gIdAcYZ.gif)

For DHCP we want to add our scope. We do this by editing the */etc/dhcp/**dhpcd.conf*** file. Again - DHCP is pointed to the Host-only network, so I want to scope in on the 10.0.0.0 network, not the bridged network running a 192.168.2.x address.

![enter image description here](https://i.imgur.com/3OEVjbo.gif)

Now we start isc-dhcp-server service through terminal. 
```
> sudo systemctl start isc-dhcp-server
```
We can also check the status to verify it's running using systemctl as well.
![enter image description here](https://i.imgur.com/Yq1VgBQ.gif)

### ![#f03c55](https://placehold.it/15/f03c55/000000?text=+)  Firewall -- ufw(Uncomplicated Firewall) & iptables Installation  & Configuration  ![#f03c55](https://placehold.it/15/f03c55/000000?text=+) 

********************************************************************
Start with enabling ufw as it comes preinstalled with Ubuntu.
```

> sudo ufw enable
> sudo ufw status
```
Allowing all SSH, DNS, and HTTP traffic and blocking all SSH on client 192.168.2.4.
```
> ufw allow 22
> ufw allow 53
> ufw allow 80
> ufw deny proto tcp from 192.168.2.4 to any port 22
> ufw status
```
![enter image description here](https://i.imgur.com/kKagC9O.gif)

Enable iptables function Masquerade for client machines to ping the gateway of the outer public network
Edit /etc/sysctl.conf and uncomment these two lines
```
#net.ipv4.ip_forward=1
#net.ipv6.conf.default.forwarding=1
```
then execute *sudo sysctl -p*

Run the following command replacing the IP/Subnet CIDR number with your host-only network and your Internet-facing NIC. In this case, it was **ens33**.
```
> sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o ens33 -j MASQUERADE
```
Run to apply the final rule
```
> sudo iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
```
and to verify masquerading rules
```
> sudo iptables -t nat -L -n -v
```

### ![#4B0082](https://placehold.it/15/4B0082/000000?text=+)  Proxy Server -- Squid Installation  & Configuration  ![#4B0082](https://placehold.it/15/4B0082/000000?text=+) 

********************************************************************
Simply run 
```
> sudo apt-get install squid
```
Feel free to create a backup of the main **/etc/squid/*squid.conf*** file by typing in:
```
> sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.original
```
Next open the *squid.conf* as we will add the following:
```
# Host and Network Allow
acl localhost src 127.0.0.1/32
acl localnet src 192.168.2.0/24 <-- Bridged/Outside network
http_access allow localnet
http_access allow localhost
# Badsite redirector
acl badsites dstdomain .neverssl.com
deny_info https://google.com localnet
deny_info https://google.com localhost
http_reply_access deny badsites localnet
http_reply_access deny badsites localhost
# Proxy Port Assignments
http_port 3129
http_port 8888 intercept
http_port 3128 transparent
```

Next we want to forward our Squid ports through uncomplicated firewall by typing 
```
> ufw allow 3128
> ufw allow 3129
> ufw allow 8888
```

Restart Squid and ufw
```
> sudo service squid status
> sudo service squid restart
> systemctl restart squid.service
> /var/log/squid/access.log
> /var/log/squid/cache.log
> ufw reload
> systemctl restart ufw
```
After this process I had to wait about 2 minutes and then using port 3129 with the Proxy Server IP plugged into your web browser, you should have a working proxy that redirects client traffic when going to www.neverssl.com.

|Monitoring of Squid Proxy Server Traffic| 
|-|

![enter image description here](https://i.imgur.com/4aXaAlO.png)

|Neverssl Redirect| 
|-|

![enter image description here](https://i.imgur.com/gxv6juw.gif)

And that concludes the Gateway-Services setup! 

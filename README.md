# ROGER SKYLINE

## INSTALL VM
**Install VirtualBox**

Name : `roger`<br />
Type : `Linux`<br />
Version : `Debian (64-bits)`<br />
Memory size : `1024 Mb`<br />
Hard Disk : `create a virtual hard disk now`<br />
Hard Disk file type : `VDI (VirtualBox Disk Image)`<br />
Storage : `8 Gb & Fixed size`<br />
File location and size : `/sgoinfre/goinfre/Perso/xxx/roger.vdi`<br />
Set bridge connection :<br />
`Settings -> network -> bridge connection (connection between host and vm) (-> en0)`

**Install Debian**

Download debian <br />
https://www.debian.org/distrib/netinst<br />
Select: amd64<br />
Starting Iso

**Run the vm**

_Region:_ <br />
Language : `English`<br />
Country : `United States`<br />
Keymap to use : `American English`<br />

_Configure the network:_<br />
Hostname: `debian`<br /><br />
Domain name : `None`<br />
Select your timezone : `Eastern`<br />

_Partitions:_<br />
```
Manual
SCSI1 (sda)
Create new empty partition
pri/log
Create a new partition
4.5 Gb -> Primary -> Beginning -> Ext4
1.0 Gb -> Primary -> beginning -> Swap Area
The rest -> Primary -> beginning -> Ext4
Finish partitioning and write changes to disk -> yes
Use another CD -> no
```
_Package manager:_<br />
```
Debian archive mirror
United States
ftp.us.debian.org
```
_Configuration:_<br />
```
HTTP proxy: None
Install running
Participate in the survey -> no
Choose Software to Install
Select the last 2 softwares: [SSH server] [Standard system utilities] 
Install GRUB -> yes
```

## Login as root
```
apt-get update && apt-get upgrade
apt-get install sudo vim iptables-persistent fail2ban sendmail nginx portsentry
```
## Create and add user to sudo group
**Create user:**

`adduser login sudo`

**Add user to sudo:**

Edit /etc/sudoers : 
`sudo vim /etc/sudoers`
```# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults	env_reset
Defaults	mail_badpass
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root	ALL=(ALL:ALL) ALL
user	ALL=(ALL:ALL) PASSWD:ALL

# Allow members of group sudo to execute any command
%sudo	ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
```

## Static IP and Netmask /30

Edit /etc/network/interfaces :
`sudo vim /etc/network/interfaces`
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
	address	10.13.16.140
	netmask	255.255.255.252
	gateway	10.13.254.254
```
  
Edit /etc/resolv.conf : `sudo vim /etc/resolv.conf`
```
domain 42.fr
search 42.fr
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 10.13.16.140
```

## Configure SSH

Edit /etc/ssh/sshd_config :
`sudo vim /etc/ssh/sshd_config`
```
Port: 2222
Uncomment #PasswordAuthentification yes
Uncomment #PubkeyAuthentication yes
Uncomment #PermitRootLogin prohibit-password and replace by No
sudo reboot
```
Outside of the VM, open a terminal:
```
ssh-keygen
ssh-copy-id user@ip -p 'port number'
ssh -p 'port number' 'user@ip'
```
Edit /etc/ssh/sshd_config again:
`PasswordAuthentification yes" à no`

Restart SSH:
`sudo service ssh restart`

Check if root have access:
`ssh -p 'port number' 'root@ip'`

Should have the following output:
`Permission denied (publickey)`

## Configure Firewall

**Configure IPTABLES**

Create a file named firewall : 
`sudo vim /etc/network/if-pre-up.d/firewall`

Add the following lines to the file :
```
#!/bin/bash

#Flush iptables
sudo iptables -F
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -t mangle -F
sudo iptables -t mangle -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT

#Blocking all
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT DROP

#Droping all invalid packets
sudo iptables -A INPUT -m state --state INVALID -j DROP
sudo iptables -A FORWARD -m state --state INVALID -j DROP
sudo iptables -A OUTPUT -m state --state INVALID -j DROP

#Flooding of RST packets, smurf attack Rejection
sudo iptables -A INPUT -p tcp -m tcp --tcp-flags RST RST -m limit --limit 2/second --limit-burst 2 -j ACCEPT

#For SMURF attack protection
sudo iptables -A INPUT -p icmp -m icmp --icmp-type address-mask-request -j DROP
sudo iptables -A INPUT -p icmp -m icmp --icmp-type timestamp-request -j DROP

#Attacking IP will be locked for 24 hours (3600 x 24 = 86400 Seconds)
sudo iptables -A INPUT -m recent --name portscan --rcheck --seconds 86400 -j DROP
sudo iptables -A FORWARD -m recent --name portscan --rcheck --seconds 86400 -j DROP

#Remove attacking IP after 24 hours
sudo iptables -A INPUT -m recent --name portscan --remove
sudo iptables -A FORWARD -m recent --name portscan --remove

#PORT SCAN
sudo iptables -A INPUT -p TCP -m state --state NEW -m recent --set
sudo iptables -A INPUT -p TCP -m state --state NEW -m recent --update --seconds 1 --hitcount 10 -j DROP

#DOS - This rule limits the ammount of connections from the same IP in a short time
sudo iptables -I INPUT -p TCP --dport 8888 -i enp0s3 -m state --state NEW -m recent --set
sudo iptables -I INPUT -p TCP --dport 8888 -i enp0s3 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
sudo iptables -I INPUT -p TCP --dport 80 -i enp0s3 -m state --state NEW -m recent --set
sudo iptables -I INPUT -p TCP --dport 80 -i enp0s3 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
sudo iptables -I INPUT -p TCP --dport 443 -i enp0s3 -m state --state NEW -m recent --set
sudo iptables -I INPUT -p TCP --dport 443 -i enp0s3 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
sudo iptables -I INPUT -p TCP --dport 25 -i enp0s3 -m state --state NEW -m recent --set
sudo iptables -I INPUT -p TCP --dport 25 -i enp0s3 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
sudo iptables -I INPUT -p TCP --dport 53 -i enp0s3 -m state --state NEW -m recent --set
sudo iptables -I INPUT -p TCP --dport 53 -i enp0s3 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP

#Keeping connections already ESTABLISHED
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -m conntrack ! --ctstate INVALID -j ACCEPT

#Accept lo
sudo iptables -t filter -A INPUT -i lo -j ACCEPT
sudo iptables -t filter -A OUTPUT -o lo -j ACCEPT

#ACCEPT SSH
sudo iptables -A INPUT -p TCP --dport 8888 -j ACCEPT
sudo iptables -A OUTPUT -p TCP --dport 8888 -j ACCEPT

#ACCEPT HTTP
sudo iptables -A INPUT -p TCP --dport 80 -j ACCEPT
sudo iptables -A OUTPUT -p TCP --dport 80 -j ACCEPT

#ACCEPT HTTPS
sudo iptables -A INPUT -p TCP --dport 443 -j ACCEPT
sudo iptables -A OUTPUT -p TCP --dport 443 -j ACCEPT

#ACCEPT SMTP
sudo iptables -t filter -A INPUT -p tcp --dport 25 -j ACCEPT
sudo iptables -t filter -A OUTPUT -p tcp --dport 25 -j ACCEPT

#ACCEPT DNS
sudo iptables -t filter -A INPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -t filter -A OUTPUT -p tcp --dport 53 -j ACCEPT

# Lastly reject All INPUT traffic
sudo iptables -A INPUT -j REJECT

#ACCEPT PING
sudo iptables -t filter -A INPUT -p icmp -j ACCEPT
sudo iptables -t filter -A OUTPUT -p icmp -j ACCEPT

exit 0
```

Run the following commands to apply the iptables rules:
```
sudo chmod 700 /etc/network/if-pre-up.d/firewall
sudo sh /etc/network/if-pre-up.d/firewall
sudo reboot
```

**Configure FAIL2BAN**

Create a file named jail.local :
`sudo vim /etc/fail2ban/jail.local`

Add the following lines to the file :
```
[DEFAULT]
ignoreip = 127.0.0.1/8 ip

[sshd]
port    = port number
enabled = true
maxretry = 3
findtime = 30
bantime = 60
logpath = /var/log/auth.log

[sshd-ddos]
port    = port
logpath = /var/log/auth.log
enabled = true

[nginx-http-auth]
port     = http,https
logpath  = /var/log/nginx/error.log
enabled = true

[nginx-limit-req]
port     = http,https
logpath  = /var/log/nginx/error.log
enabled = true

[nginx-botsearch]
port     = http,https
logpath  = /var/log/nginx/error.log
enabled = true
maxretry = 300
findtime = 300
bantime = 60

[recidive]
enabled = true
```

Restart fail2ban : 
`sudo service fail2ban restart`

**Configure PORTSENTRY**

Follow the steps showed in the following link :<br />
https://fr-wiki.ikoula.com/fr/Se_prot%C3%A9ger_contre_le_scan_de_ports_avec_portsentry

Or follow the steps showed bellow :
`sudo /etc/init.d/portsentry stop`

Edit the /etc/default/portsentry :
`sudo vim /etc/default/portsentry`
```
TCP_MODE="atcp"
UDP_MODE="audp"
```

Edit /etc/portsentry/portsentry.conf :
`sudo vim /etc/portsentry/portsentry.conf`
```
BLOCK_UDP="1"
BLOCK_TCP="1"
```
Comment the current KILL_ROUTE and uncomment the following one :
`KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"`

Check with the following command : 
`sudo cat /etc/portsentry/portsentry.conf | grep KILL_ROUTE | grep -v '#'`

Restart portsentry : 
`sudo service portsentry restart`

## Ports Scans

**SLOWLORIS**

Follow the steps showed in the following link:<br />
https://github.com/llaera/slowloris.pl

**NMAP**

Enter the following command to tests your ports:
`nmap --script dos -Pn 10.13.16.140`

Go on this website to know more about Nmap:<br />
https://null-byte.wonderhowto.com/how-to/use-nmap-7-discover-vulnerabilities-launch-dos-attacks-and-more-016878

## Scripts

Open crontab:
`sudo crontab -e`

Enter the following lines:
```
@reboot { sudo apt-get update; sudo apt-get -y upgrade; } >> /var/log/update_script.log
0 4 * * 6 { sudo apt-get update; sudo apt-get -y upgrade; } >> /var/log/update_script.log
```

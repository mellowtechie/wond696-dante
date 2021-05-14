# Dante socks5 Proxy setup on OVHcloud Ubuntu VPS
I have build this as a step by step guide to building a socks5 proxy on an Ubuntu VPS

## Environment Details

OVHcloud Starter VPS https://us.ovhcloud.com/vps/cheap-vps/  
Latest Ubuntu Build 20.10  
Dante socks5 proxy https://www.inet.no/dante/  
Fail2ban https://www.fail2ban.org/wiki/index.php/Main_Page  

## Build Overview

1) Create VPS. The deante socks5 proxy is simple to use but as of the time of this writing has an issue with authentication in the Ubuntu repository.
2) Login to VPS and create a new user with sudo ability. 
3) Disable sshd login with the default 'ubuntu' account.
4) Enable ufw and allow ssh.
5) Install and configure fail2ban to protect sshd using the ufw firewall.
6) Download, compile, and install dante. 
7) Configure and test dante.

### Create VPS

Create Account https://www.ovh.com/auth/signup/#/  
Dante is very lightweight so I'm using the Starter VPS with Snapshots.
Cost is under $5 a month for 1 vCore, 2GB of Memory, 20GB of Disk, and 100Mbps of bandwidth. 
OVHcloud allows you to add up to 16 additional IPv4 IP's at a one time cost of $3 each, make sure you allocate and enable or they may recover the IP's. This is what I was told by support but I cannot guarantee the terms, talk to them yourself. 
If you think you need a higher grade VPS now is the time to do it.
Go to https://us.ovhcloud.com/vps/cheap-vps/ and buy a VPS.

### Login to VPS and create a new user with sudo ability. 

Once you recieve the email with the default login connect to your VPS over ssh. I'm on a mac using the terminal so its "ssh ubuntu@ip".

### Disable sshd login with the default 'ubuntu' account.

### Enable ufw and allow ssh.

### Install and configure fail2ban to protect sshd using the ufw firewall.

### Download, compile, and install dante. 

### Configure and test dante.

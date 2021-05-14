# Dante socks5 Proxy setup on OVHcloud Ubuntu VPS
I have build this as a setp by step guide to building a socks5 proxy on an Ubuntu VPS

## Environment Details

OVHcloud Starter VPS https://us.ovhcloud.com/vps/cheap-vps/  
Latest Ubuntu Build  
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

### Login to VPS and create a new user with sudo ability. 

### Disable sshd login with the default 'ubuntu' account.

### Enable ufw and allow ssh.

### Install and configure fail2ban to protect sshd using the ufw firewall.

### Download, compile, and install dante. 

### Configure and test dante.

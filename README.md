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
4) Update the distribution and apps
5) Enable ufw and allow ssh.
6) Install and configure fail2ban to protect sshd using the ufw firewall.
7) Download, compile, and install dante. 
8) Configure and test all extra IP's.
9) Configure and test dante.

### Create VPS

Create Account https://www.ovh.com/auth/signup/#/  
Dante is very lightweight so I'm using the Starter VPS with Snapshots.  
Cost is under $5 a month for 1 vCore, 2GB of Memory, 20GB of Disk, and 100Mbps of bandwidth.   
OVHcloud allows you to add up to 16 additional IPv4 IP's at a one time cost of $3 each, make sure you allocate and enable or they may recover the IP's. This is what I was told by support but I cannot guarantee the terms, talk to them yourself.  
If you think you need a higher grade VPS now is the time to do it.  
Go to https://us.ovhcloud.com/vps/cheap-vps/ and buy a VPS.

### Login to VPS and create a new user with sudo ability. 

Once you recieve the email with the default login connect to your VPS over ssh. I'm on a mac using the terminal so its `ssh ubuntu@xxx.xxx.xxx.xxx` but you can use any ssh client like Putty.  

Create a user, my example is newuser but make sure you use something you'll remember that's uncommon.
```
sudo adduser newuser
```

Follow the instructions to complete the process.
```
Adding user `newuser' ...
Adding new group `newuser' (1005) ...
Adding new user `newuser' (1004) with group `newuser' ...
Creating home directory `/home/newuser' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
Changing the user information for newuser
Enter the new value, or press ENTER for the default
 Full Name []: Name                
 Room Number []: 1234
 Work Phone []: 0123456789
 Home Phone []: 0987654321
 Other []: 
Is the information correct? [Y/n] Ysudo
```

Add this user to the sudo group.
```
sudo usermod -aG sudo newuser
```

Set the password for this user.

```
passwd newuser
```

Switch to newuser.

```
su - newuser
```

Test sudo permissions.

```
newuser@ubuntu:~$ sudo whoami
[sudo] password for newuser: 
root
newuser@ubuntu:~$ 
```

For extra security you could switch to private key authentication but I won't get into that.  

Next, use the same process to create a proxy user for later, but in this case we don't want a home directory or the ability to login. Change proxyuser to whatever username you want, it cannot be a generic system name such as proxy.

```
sudo useradd -s /usr/sbin/nologin -M proxyuser
```

Set the password for proxyuser

```
passwd proxyuser
```

### Disable sshd login with the default 'ubuntu' account.

In this next step we will disable the default ubuntu account from logging in via ssh. I do this because its a common brute force account attempt.  
I user the vi editor but you may find vim or nano easier.
https://www.shell-tips.com/cheat-sheets/vim-quick-references/

```
sudo vi /etc/ssh/sshd_config
```

Scroll to the end and hit `a` to enter insert mode, hit enter twice, and enter the following line.

```
DenyUsers ubuntu
```

Hit `esc`, type `:wq` and hit enter. Finish this by restarting sshd service.

```
sudo systemctl restart sshd
```

Test to make sure the ubuntu account is disabled.

```
me@localhost ~ % ssh ubuntu@xxx.xxx.xxx.xxx
ubuntu@xxx.xxx.xxx.xxx's password: 
Permission denied, please try again.
ubuntu@xxx.xxx.xxx.xxx's password: 

```

Close your session and relogin with your new account `ssh newuser@xxx.xxx.xxx.xxx`. 

```
me@localhost ~ % ssh newuser@xxx.xxx.xxx.xxx
newuser@xxx.xxx.xxx.xxx's password: 
Welcome to Ubuntu 21.04 (GNU/Linux 5.11.0-17-generic x86_64)
```

### Update the distribution and apps

```
sudo do-release-upgrade
```

Just follow the prompts and read the man page. https://manpages.ubuntu.com/manpages/precise/man8/do-release-upgrade.8.html. 

Run the update and upgrade sequences after the distribution is upgraded and you've logged back in.

```
me@localhost:~$ sudo apt update
[sudo] password for newuser: 
Get:1 http://security.ubuntu.com/ubuntu hirsute-security InRelease [101 kB]
Hit:2 http://nova.clouds.archive.ubuntu.com/ubuntu hirsute InRelease           
Get:3 http://nova.clouds.archive.ubuntu.com/ubuntu hirsute-updates InRelease [109 kB]
Hit:4 http://nova.clouds.archive.ubuntu.com/ubuntu hirsute-backports InRelease
Fetched 209 kB in 1s (274 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
```
```
me@localhost:~$ sudo apt upgrade
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
me@localhost:~$ 
```

### Enable ufw and allow ssh.

ufw is the default firewall on Ubuntu 2x.xx so that's what we will use.  

I already did this in the example but you'll want to make sure you allow ssh connections first

```
me@localhost:/etc/ssh$ sudo ufw allow ssh
[sudo] password for tmurray: 
Skipping adding existing rule
Skipping adding existing rule (v6)
me@localhost:/etc/ssh$ 
```

Enable ufw.

```
me@localhost:/etc/ssh$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
me@localhost:/etc/ssh$ 
```

Check status, it should say active.

```
me@localhost:/etc/ssh$ sudo systemctl status ufw
● ufw.service - Uncomplicated firewall
     Loaded: loaded (/lib/systemd/system/ufw.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2021-05-14 19:15:17 UTC; 1h 23min ago
       Docs: man:ufw(8)
    Process: 360 ExecStart=/lib/ufw/ufw-init start quiet (code=exited, status=0/SUCCESS)
   Main PID: 360 (code=exited, status=0/SUCCESS)

May 14 19:15:17 localhost systemd[1]: Finished Uncomplicated firewall.
Warning: journal has been rotated since unit was started, output may be incomplete.
```

### Install and configure fail2ban to protect sshd using the ufw firewall.

Install Fail2ban using apt. This package will watch for 5 failed ssh login attempts in 10 minutes and ban the origin IP for 1h.  
This is customizable in the jail.local file. https://www.fail2ban.org/wiki/index.php/Main_Page

```
sudo apt install fail2ban
```

Now we need to configure Fail2ban to use the ufw firewall, enable sshd monitoring, and validate it's working.  
Package updates may overwrite the deafult config so we will create a local conf file.

Using `vi` as above lets create the jail.local file.
```
sudo vi /etc/fail2ban/jail.local
```

Hit `i` to insert and paste in the following.  

```
[DEFAULT]
# Instruct fail2ban to use the ufw firewall
banaction = ufw
# Ban hosts for one hour:
bantime = 3600

# Make sure we don't ban local IP's. Make sure you add the IP of your VPS to this list.
ignoreip = 127.0.0.1/8 ::1 123.123.123.123 192.168.1.0/24

[sshd]
enabled = true
           
```

Hit `esc`, type `:wq`, and hit enter to save.

Enable and start the fail2ban service.

```
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Verify the service is running.

```
sudo systemctl status fail2ban
```

You should see the following.

```
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2021-05-14 19:15:22 UTC; 1h 41min ago
       Docs: man:fail2ban(1)
    Process: 709 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 722 (fail2ban-server)
      Tasks: 5 (limit: 2191)
     Memory: 16.4M
     CGroup: /system.slice/fail2ban.service
             └─722 /usr/bin/python3 /usr/bin/fail2ban-server -xf start

May 14 19:15:22 vps-1d98edda systemd[1]: Starting Fail2Ban Service...
May 14 19:15:22 vps-1d98edda systemd[1]: Started Fail2Ban Service.
May 14 19:15:25 vps-1d98edda fail2ban-server[722]: Server ready
```

Wait about 5 minutes and check the firewall status, you may have bans already.

```
sudo ufw status
```

You should see the following.

```
Status: active

To                         Action      From
--                         ------      ----
Anywhere                   REJECT      223.220.251.232           
Anywhere                   REJECT      129.211.94.30             
Anywhere                   REJECT      114.113.225.111           
Anywhere                   REJECT      106.13.236.133            
Anywhere                   REJECT      167.172.190.11            
Anywhere                   REJECT      180.76.235.75             
Anywhere                   REJECT      43.129.28.88              
Anywhere                   REJECT      106.75.50.173             
Anywhere                   REJECT      111.229.176.234           
Anywhere                   REJECT      81.182.254.124            
Anywhere                   REJECT      1.15.186.85               
Anywhere                   REJECT      47.250.46.46              
Anywhere                   REJECT      45.158.199.156            
Anywhere                   REJECT      139.59.81.146             
Anywhere                   REJECT      157.230.42.76             
Anywhere                   REJECT      143.110.176.216           
Anywhere                   REJECT      47.74.92.9                
Anywhere                   REJECT      27.128.229.118            
Anywhere                   REJECT      139.155.252.205           
Anywhere                   REJECT      106.13.113.143            
Anywhere                   REJECT      210.14.142.85             
22/tcp                     ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6)             
```

### Download and install dante. 

Now we get to the install and configuration of dante 

```
sudo apt install dante-server
```

At one point there was an issue with the apt package dante-server 1.4.2 package where authentication was broken but it seems to be fixed.  
If you need to compile from source there are many guides out there.


### Configure and test all extra IP's.

If you are binding multiple IP's to your VPS pause and go do that first. For OVHcloud use the following guide and make sure you can ping each IP before proceeding. https://support.us.ovhcloud.com/hc/en-us/articles/360014248820-How-to-Configure-IP-Aliasing-on-a-VPS. You may also need to review this to see how to assign multiple addresses on Ubuntu 20.10 with netplan.  

It took some trial and error to get the netplan configuration correct but here is how I ended up doing it. 

Edit the netplan configuration.

```
sudo vi /etc/netplan/50-cloud-init.yaml
```

Rather than creating ton's of duplicates I added all my IP's including the default (first in order) to the ens3 section by adding an `addresses:` section with IP lists. IP's and MAC address are obscured but you get the idea, you can copy/paste the `addresses:` section and below. Read this for help. https://netplan.io/examples/#using-multiple-addresses-on-a-single-interface

```
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: true
            match:
                macaddress: xx:xx:xx:xx:xx:xx
            mtu: 1500
            set-name: ens3
            addresses:
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                - xxx.xxx.xxx.xxx/32
                                      
```

Make sure you commit changes before moving on.

```
sudo netplan apply
```

### Configure and test dante.

First rename the default conf file and then we will create a new one. Using `vi` as above lets create the danted.conf file.
```
sudo mv /etc/danted.conf /etc/danted.conf.old && sudo vi /etc/danted.conf
```

Hit `i` to insert and paste in the following.  
Make sure you put in your public IP where xxx.xxx.xxx.xxx is. If you have multiple IP's bound duplicate the `internal:` and `external:` lines for each IP.

```
errorlog: socks.errlog
logoutput: socks.log

internal: xxx.xxx.xxx.xxx port = 1080
external: xxx.xxx.xxx.xxx

clientmethod: none
socksmethod: username
user.privileged: root
user.unprivileged: nobody
#user.libwrap: nobody
external.rotation: same-same

client pass {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: error connect disconnect
}
client block {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: connect error
}
socks pass {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: error connect disconnect
}
socks block {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: connect error
}


```

Hit `esc`, type `:wq`, and hit enter to save.  

Enable and start the dante service

```
sudo systemctl enable danted
sudo systemctl start danted
```

Check to make sure the service started, if it failed read through the errors at the bottom. 

```
sudo systemctl status danted
```

You should see the following.

```
● danted.service - SOCKS (v4 and v5) proxy daemon (danted)
     Loaded: loaded (/lib/systemd/system/danted.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2021-05-17 02:02:04 UTC; 4s ago
       Docs: man:danted(8)
             man:danted.conf(5)
    Process: 9073 ExecStartPre=/bin/sh -c       uid=`sed -n -e "s/[[:space:]]//g" -e "s/#.*//" -e "/^user\.privileged/{s/[^:]*://p;q;>
   Main PID: 9077 (danted)
      Tasks: 20 (limit: 2281)
     Memory: 8.3M
     CGroup: /system.slice/danted.service
             ├─9077 /usr/sbin/danted
             ├─9078 danted: monitor
             ├─9079 danted: negotia
             ├─9080 danted: request
             ├─9081 danted: request
             ├─9082 danted: request
             ├─9083 danted: request
             ├─9084 danted: request
             ├─9085 danted: request
             ├─9086 danted: request
             ├─9087 danted: request
             ├─9088 danted: request
             ├─9089 danted: request
             ├─9090 danted: request
             ├─9091 danted: request
             ├─9092 danted: request
             ├─9093 danted: request
             ├─9094 danted: request
             ├─9095 danted: request
             └─9096 danted: io-chil

May 17 02:02:04 vps-ad57bd8f systemd[1]: Starting SOCKS (v4 and v5) proxy daemon (danted)...
May 17 02:02:04 vps-ad57bd8f systemd[1]: Started SOCKS (v4 and v5) proxy daemon (danted).
May 17 02:02:04 vps-ad57bd8f danted[9077]: May 17 02:02:04 (1621216924.226008) danted[9077]: warning: openlogfile(): could not open o>
May 17 02:02:04 vps-ad57bd8f danted[9077]: May 17 02:02:04 (1621216924.226599) danted[9077]: alert: configparsing(): could not (re)op>
May 17 02:02:04 vps-ad57bd8f danted[9077]: May 17 02:02:04 (1621216924.229192) danted[9077]: warning: checkconfig(): more than one ex>
```

Add a firewall rule for here, if you chose a port other than 1080 then use it here.

```
sudo ufw allow 1080/tcp
sudo ufw status
```

If everything is ready you should see something similar.

```
Status: active

To                         Action      From
--                         ------      ----
Anywhere                   REJECT      34.87.160.10              
Anywhere                   REJECT      46.151.212.38             
Anywhere                   REJECT      159.89.199.80             
Anywhere                   REJECT      174.138.28.36             
Anywhere                   REJECT      200.111.120.180           
22/tcp                     ALLOW       Anywhere                  
1080/tcp                   ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6)             
1080/tcp (v6)              ALLOW       Anywhere (v6)       
```

Now, with everything done let's test. As usual replace the username, password, and IP with your configuration. Run this against each IP you have and it should return the IP that you sent each request to. This will confirm what others will see when you use your proxy.

```
curl -v -x socks5://proxyuser:password@xxx.xxx.xxx.xxx:1080 https://ifconfig.me/all
```

At this point you should be working, if you want to setup Ubuntu to automatically stay updated here is a guide. https://www.cyberciti.biz/faq/how-to-set-up-automatic-updates-for-ubuntu-linux-18-04/

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
8) Configure and test dante.

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

### Download, compile, and install dante. 

### Configure and test dante.

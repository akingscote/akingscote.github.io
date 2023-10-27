---
title: "Debian 9 Integrated with Windows Server 2012 Active Directory"
date: "2018-12-29"
categories: 
  - "networking"
tags: 
  - "active-directory"
  - "authentication"
  - "debian"
  - "linux"
  - "windows-server"
coverImage: "featured_image.png"
---

I recently tried to get Debian integrated into Windows Server 2012 Active Directory but found myself frustrated with a lack of detailed/current documentation. There is a guide on the Debian website, but its quite outdated [https://wiki.debian.org/AuthenticatingLinuxWithActiveDirectory](https://wiki.debian.org/AuthenticatingLinuxWithActiveDirectory).

This post details the steps to get a Debian 9 virtual machine authenticated with active directory hosted on a Windows Server 2012 virtual machine.

The first thing you'll need to do it set the environment up - i just created two Virtual machines. The debian 9 image used was taken from [here](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.6.0-amd64-netinst.iso). I didnt set up static IP addressing (something that bit me later on) but its not a problem, its actually a good learning piece. You just need to configured the DHCP to no pull down DNS information.

On the Windows 2012 Server, I changed the PC name to AKINGSCOTEDC then added the Active Directory Domain Services & DNS  Server roles. When configuring the Active Directory Role, i selected the option to create a new forrest and called it "akingscote.com". I then opened up Active Directory Users and Computers, selected users and created a new test account called "testuser". I also made sure that the account i was using (not the testuser) was part of the domain admins just in case. I hopped onto the debian VM machine and made sure that i can ping the windows 2012 server.

So you should now be in a situation where you have a Debian 9 VM and a windows 2012 setup with DNS  & Active Directory installed and they can ping each other. The first thing to do is to configure the Debian 9 VM to use the Windows 2012 server as its DNS server. This will save a lot of problems down the road if you do it as follows. Do not add the server into the /etc/hosts file, that is not the same as setting it as the DNS server!

As i was using a virtual machine, i had to create the interface in the network interfaces file. Use "ip address" to check the interface name (it should be ens33 if using a VM). Enter the IP address of the server and the domain name as follows:

```
sudo nano /etc/network/interfaces
auto ens33
iface ens33 inet dhcp
    dns-nameservers 192.168.161.131
    dns-search akingscote.com
```


**IMPORTANT**

As i was (lazilly) using the default DHCP, i needed to update the configuration file to not pull down DNS information. Remove the dns-name-servers from the request section of /etc/dhcp/dhclient.conf. It should look like the following:

```
request subnet-mask, broadcast-address, time-offset, routers, host-name,
dhcp6.name-servers, dhcp6.domain-search, dhcp6.fqdn, dhcp6.sntp-servers,
netbios-name-servers, netbios-scope, interface-mtu,
rfc3442-classless-static-routes, ntp-servers;
```

 

Restart the network services/machine. On reboot you should be able to ping the domain name directly (make sure there **is not** an entry in the host file).
![](/images/ping_domain.png)

If you have problems, try downloading resolvconf and restarting everything.

```
sudo apt-get install resolvconf
...
systemctl restart ifup@ens33 resolvconf
```
 

 

## Integration

Now that the environment is setup properly, we can start on the actual integration.

Download samba, kerbaros and winbind

```sudo apt-get install winbind samba Krb5-user Libpam-krb5 libpam-winbind libnss-winbind```

When installing kerbaros, dont worry too much about the realm config it makes you do. We are going to tweak the config file later anyway.

### Kerbaros Configuration

The first thing to do is to configure kerbaros. You must be able to resolve the domain name to your windows 2012 server first.

Edit the file at /etc/krb5.conf. Add in your domain as a Kerbaros realm and make sure you add in the domains. As you can see from my configuration, Kerbaros wants any mention of the domain name to be in CAPITALS.

```
ashley@debian:~$ cat /etc/krb5.conf
[libdefaults]
default_realm = AKINGSCOTE.COM
 
# The following krb5.conf variables are only for MIT Kerberos.
kdc_timesync = 1
ccache_type = 4
forwardable = true
proxiable = true
 
# The following encryption type specification will be used by MIT Kerberos
# if uncommented.  In general, the defaults in the MIT Kerberos code are
# correct and overriding these specifications only serves to disable new
# encryption types as they are added, creating interoperability problems.
#
# The only time when you might need to uncomment these lines and change
# the enctypes is if you have local software that will break on ticket
# caches containing ticket encryption types it doesn't know about (such as
# old versions of Sun Java).
 
#    default_tgs_enctypes = des3-hmac-sha1
#    default_tkt_enctypes = des3-hmac-sha1
#    permitted_enctypes = des3-hmac-sha1
 
# The following libdefaults parameters are only for Heimdal Kerberos.
fcc-mit-ticketflags = true
 
[realms]
AKINGSCOTE.COM = {
kdc = AKINGSCOTE.COM:88
admin_server = AKINGSCOTE.COM:464
default_domain = AKINGSCOTE.COM
}
 
[domain_realm]
.akingscote = AKINGSCOTE.COM
akingscote.com = AKINGSCOTE.COM
ashley@debian:~$
```

Once you have updated the config file, you should be able to request a kerbaros ticket for a user that is on the server. As  a precaution, i made sure to use a user that i know is in the domain admin user group.

`sudo kinit Administration@AKINGSCOTE.COM`

If you dont get anything in the console afterwards, the chances are that it was succesful. You can verify that it was by using the klist command as a super user.

![](/images/kerbaros-ticket.png)

Note the Default principal account is the one we just specified.

## Samba Configuration

Now that kerbaros is up and running, the next stage is to setup the samba share and configure winbind.

I found the samba configuration is very versatile, but very tempremental. It is important that the DNS resolution is working properly (not through hosts file) otherwise the winbind service wont start properly (took far too long to debug!).

Edit the saba configuration file at /etc/samba/smb.conf.

The configuration file has been re-organised in the later versions. I ran "samba -V" and found that i was using "Version 4.5.12-Debian". Here is a dump of my updated configuration file. The notable changes are that i added a bunch of values to the \[global\] section and i changed the "wins support" to yes. As well as this, i updated the syslog configuration section to actually write to syslog. Why on earth they would ship it then complain about obsolete config items i will never know.  I also commented out the printer section although i dont think thats need (i was trying some debug).

A really useful command is the "testparm" command which will check the configuration file. It will complain that the uid & gid are obsolete but it still works.

If you are unsure of your workstation name, you can use "net config workstation" on the windows machine.

![](/images/windows-workstation.png)

```
ashley@debian:~$ cat /etc/samba/smb.conf
#
# Sample configuration file for the Samba suite for Debian GNU/Linux.
#
#
# This is the main Samba configuration file. You should read the
# smb.conf(5) manual page in order to understand the options listed
# here. Samba has a huge number of configurable options most of which
# are not shown in this example
#
# Some options that are often worth tuning have been included as
# commented-out examples in this file.
#  - When such options are commented with ";", the proposed setting
#    differs from the default Samba behaviour
#  - When commented with "#", the proposed setting is the default
#    behaviour of Samba but the option is considered important
#    enough to be mentioned here
#
# NOTE: Whenever you modify this file you should run the command
# "testparm" to check that you have not made any basic syntactic
# errors.
 
#======================= Global Settings =======================
 
[global]
security = ADS
realm = AKINGSCOTE.COM
workgroup = AKINGSCOTE
dns forwarder = 192.168.161.131
idmap config * : backend = tdb
idmap config *:range = 50000-1000000
template homedir = /home/%D/%U
template shell = /bin/bash
winbind use default domain = true
winbind offline logon = false
winbind nss info = rfc2307
winbind enum users = yes
winbind enum groups = yes
netbios name = debian
vfs objects = acl_xattr
map acl inherit = Yes
store dos attributes = Yes
 
 
# Windows Internet Name Serving Support Section:
# WINS Support - Tells the NMBD component of Samba to enable its WINS Server
wins support = yes
 
# WINS Server - Tells the NMBD components of Samba to be a WINS Client
# Note: Samba can be either a WINS Server, or a WINS Client, but NOT both
;   wins server = w.x.y.z
 
# This will prevent nmbd to search for NetBIOS names through DNS.
dns proxy = no
 
#### Networking ####
 
# The specific set of interfaces / networks to bind to
# This can be either the interface name or an IP address/netmask;
# interface names are normally preferred
;   interfaces = 127.0.0.0/8 eth0
 
# Only bind to the named interfaces and/or networks; you must use the
# 'interfaces' option above to use this.
# It is recommended that you enable this feature if your Samba machine is
# not protected by a firewall or is a firewall itself.  However, this
# option cannot handle dynamic or non-broadcast interfaces correctly.
;   bind interfaces only = yes
 
 
 
#### Debugging/Accounting ####
 
# This tells Samba to use a separate log file for each machine
# that connects
log file = /var/log/samba/log.%m
 
# Cap the size of the individual log files (in KiB).
max log size = 1000
 
# If you want Samba to only log through syslog then set the following
# parameter to 'yes'.
#   syslog only = no
 
# We want Samba to log a minimum amount of information to syslog. Everything
# should go to /var/log/samba/log.{smbd,nmbd} instead. If you want to log
# through syslog you should set the following parameter to something higher.
#   syslog = 0
logging = syslog@1 /var/log/samba/log.%m
 
# Do something sensible when Samba crashes: mail the admin a backtrace
panic action = /usr/share/samba/panic-action %d
 
 
####### Authentication #######
 
# Server role. Defines in which mode Samba will operate. Possible
# values are "standalone server", "member server", "classic primary
# domain controller", "classic backup domain controller", "active
# directory domain controller".
#
# Most people will want "standalone sever" or "member server".
# Running as "active directory domain controller" will require first
# running "samba-tool domain provision" to wipe databases and create a
# new domain.
server role = standalone server
 
# If you are using encrypted passwords, Samba will need to know what
# password database type you are using.
passdb backend = tdbsam
 
obey pam restrictions = yes
 
# This boolean parameter controls whether Samba attempts to sync the Unix
# password with the SMB password when the encrypted SMB password in the
# passdb is changed.
unix password sync = yes
 
# For Unix password sync to work on a Debian GNU/Linux system, the following
# parameters must be set (thanks to Ian Kahan <<kahan@informatik.tu-muenchen.de> for
# sending the correct chat script for the passwd program in Debian Sarge).
passwd program = /usr/bin/passwd %u
passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
 
# This boolean controls whether PAM will be used for password changes
# when requested by an SMB client instead of the program listed in
# 'passwd program'. The default is 'no'.
pam password change = yes
 
# This option controls how unsuccessful authentication attempts are mapped
# to anonymous connections
map to guest = bad user
 
########## Domains ###########
 
#
# The following settings only takes effect if 'server role = primary
# classic domain controller', 'server role = backup domain controller'
# or 'domain logons' is set
#
 
# It specifies the location of the user's
# profile directory from the client point of view) The following
# required a [profiles] share to be setup on the samba server (see
# below)
;   logon path = \\%N\profiles\%U
# Another common choice is storing the profile in the user's home directory
# (this is Samba's default)
#   logon path = \\%N\%U\profile
 
# The following setting only takes effect if 'domain logons' is set
# It specifies the location of a user's home directory (from the client
# point of view)
;   logon drive = H:
#   logon home = \\%N\%U
 
# The following setting only takes effect if 'domain logons' is set
# It specifies the script to run during logon. The script must be stored
# in the [netlogon] share
# NOTE: Must be store in 'DOS' file format convention
;   logon script = logon.cmd
 
# This allows Unix users to be created on the domain controller via the SAMR
# RPC pipe.  The example command creates a user account with a disabled Unix
# password; please adapt to your needs
; add user script = /usr/sbin/adduser --quiet --disabled-password --gecos "" %u
 
# This allows machine accounts to be created on the domain controller via the
# SAMR RPC pipe.
# The following assumes a "machines" group exists on the system
; add machine script  = /usr/sbin/useradd -g machines -c "%u machine account" -d /var/lib/samba -s /bin/false %u
 
# This allows Unix groups to be created on the domain controller via the SAMR
# RPC pipe.
; add group script = /usr/sbin/addgroup --force-badname %g
 
############ Misc ############
 
# Using the following line enables you to customise your configuration
# on a per machine basis. The %m gets replaced with the netbios name
# of the machine that is connecting
;   include = /home/samba/etc/smb.conf.%m
 
# Some defaults for winbind (make sure you're not using the ranges
# for something else.)
idmap uid = 10000-20000
idmap gid = 10000-20000
template shell = /bin/bash
 
# Setup usershare options to enable non-root users to share folders
# with the net usershare command.
 
# Maximum number of usershare. 0 (default) means that usershare is disabled.
;   usershare max shares = 100
 
# Allow users who've been granted usershare privileges to create
# public shares, not just authenticated ones
usershare allow guests = yes
 
#======================= Share Definitions =======================
 
[homes]
comment = Home Directories
browseable = no
 
# By default, the home directories are exported read-only. Change the
# next parameter to 'no' if you want to be able to write to them.
read only = yes
 
# File creation mask is set to 0700 for security reasons. If you want to
# create files with group=rw permissions, set next parameter to 0775.
create mask = 0700
 
# Directory creation mask is set to 0700 for security reasons. If you want to
# create dirs. with group=rw permissions, set next parameter to 0775.
directory mask = 0700
 
# By default, \\server\username shares can be connected to by anyone
# with access to the samba server.
# The following parameter makes sure that only "username" can connect
# to \\server\username
# This might need tweaking when using external authentication schemes
valid users = %S
 
# Un-comment the following and create the netlogon directory for Domain Logons
# (you need to configure Samba to act as a domain controller too.)
;[netlogon]
;   comment = Network Logon Service
;   path = /home/samba/netlogon
;   guest ok = yes
;   read only = yes
 
# Un-comment the following and create the profiles directory to store
# users profiles (see the "logon path" option above)
# (you need to configure Samba to act as a domain controller too.)
# The path below should be writable by all users so that their
# profile directory may be created the first time they log on
;[profiles]
;   comment = Users profiles
;   path = /home/samba/profiles
;   guest ok = no
;   browseable = no
;   create mask = 0600
;   directory mask = 0700
 
#[printers]
#   comment = All Printers
#   browseable = no
#   path = /var/spool/samba
#   printable = yes
#   guest ok = no
#   read only = yes
#   create mask = 0700
 
# Windows clients look for this share name as a source of downloadable
# printer drivers
#;[print$]
#;   comment = Printer Drivers
#;   path = /var/lib/samba/printers
#;   browseable = yes
#;   read only = yes
#;   guest ok = no
# Uncomment to allow remote administration of Windows print drivers.
# You may need to replace 'lpadmin' with the name of the group your
# admin users are members of.
# Please note that you also need to set appropriate Unix permissions
# to the drivers directory for these users to have write rights in it
;   write list = root, @lpadmin
```
 

Next, update the nsswitch config to use winbind as well.

```
sudo nano /etc/nsswitch.conf

....
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.
 
passwd:         compat winbind
group:          compat winbind
shadow:         compat winbind
gshadow:        files
 
hosts:          files mdns4_minimal [NOTFOUND=return] dns myhostname
networks:       files
 
protocols:      db files
services:       db files
ethers:         db files
rpc:            db files
 
netgroup:       nis
ashley@debian:~$
```

After changing the samba configuration, youll need to restart the services. I had a lot of problems restarting the winbind service which I think was down to me not having the DNS setup properly. Once i added the DNS to the interface like i specified here then it worked.

You can restart the services in the correct order by using these commands:

```
sudo systemctl restart smbd nmbd winbind
sudo systemctl stop samba-ad-dc
sudo systemctl enable smbd nmbd winbind
```

If you are having trouble with winbind, you can run it directly in the console with

`sudo winbindd -i`

## Testing

Finally we should be able to join the domain and log into the debian machine as an Active Directory user!

Join the domain with the following command:

```
ashley@debian:~$ sudo net ads join -U Administrator
[sudo] password for ashley:
Enter Administrator's password:
Using short domain name -- AKINGSCOTE
Joined 'DEBIAN' to dns domain 'akingscote.com'
No DNS domain configured for debian. Unable to perform DNS Update.
DNS update failed: NT_STATUS_INVALID_PARAMETER
```

It looks like an error, but it isnt.

Use the following command to list all users on the domain.

```
ashley@debian:~$ wbinfo -u
administrator
guest
ashley
krbtgt
testuser
ashley@debian:~$
```

Finally, change users to one which is on the domain (testuser!)

`su testuser`

 

![](/images/successful-login.png)

I even changed the password of testuser to verify that it 10000% was working. Obviously it still needs a lot of work to tweak and make production ready, but that is the hardest bit done!

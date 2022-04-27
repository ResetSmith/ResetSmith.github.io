---
title: "Using VSFTPD to setup FTP access to remote servers"
date: 2021-10-12
hero: 3567818.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: VSFTPD
    identifier: svr_mgmt-vsftp
    parent: svr_mgmt
    weight: 10
draft: true
---

This tutorial will cover installing VSFTPD to handle FTP connections to your server. File Transfer Protocol (FTP) is a commonly used protocol for accessing remote servers and exchanging files, however FTP is considered no longer secure, instead you should use VSFTPD. Very Secure File Transfer Protocol Daemon (VSFTPD) is a tool for handling FTP connections that allows for user/password logins over TLS or via SSL.

For this particular tutorial I will be configuring VSFTPD on an Ubuntu 20.04 server and using the FileZilla FTP client to test the FTP connection settings.

## Install VSFTPD
Update your package list
```
sudo apt update
```

Install VSFTPD
```
sudo apt install vsftpd
```
### Allow incoming connections in the firewall
open ports 20,21, 990/tcp
```
sudo ufw allow 20,21, 990/tcp
```
Open passive port range
This is done to allow support for PASV commands. You can find more information about this from a good article by [TitanFTP Here](https://titanftp.com/2018/08/23/what-is-the-difference-between-active-and-passive-ftp).
```
sudo ufw allow 40000:50000/tcp
```
## Creating a User Directory
Create a user for FTP access
```
sudo adduser $USERNAME
```
Create a folder for FTP files
```
sudo mkdir /home/$USERNAME/ftp
```
Update the FTP folder ownership
```
sudo chown nobody:nogroup /home/$USERNAME/ftp
```
Remove write permissions
```
sudo chmod a-w /home/$USERNAME/ftp
```
Verify the permissions
```
sudo ls -la /home/$USERNAME/ftp
```

Now create a folder for file uploads
```
sudo mkdir /home/$USERNAME/ftp/files
```
Give ownership of the files folder to the user
```
chown $USERNAME:$USERNAME /home/$USERNAME/ftp/files
```

## Configure FTP setings
Make a backup copy of the configuration file before we start
```
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.old
```
Open the vsftpd.conf file to update the settings
```
sudo vim vsftpd.conf
```
Check the following settings
```
anonymous_enable=NO
local_enable=YES
```
```
write_enable=YES
```
```
chroot_local_user=YES
```
```
user_sub_token=$USER
local_root=/home/$USER/ftp
```
```
pasv_min_port=40000
pasv_max_port=50000
```
```
userlist_enable=YES
userlist_file=/etc/vsftpd.users
userlist_deny=NO
```
```
rsa_cert_file=/etc/ssl/private/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.pem
```
```
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH
```
```
sudo systemctl restart vsftpd.service
```

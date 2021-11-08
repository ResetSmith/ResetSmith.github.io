---
title: Ubuntu Notes
menu:
  notes:
    name: Ubuntu
    identifier: notes-ubuntu
    weight: 10
---

<!--General Stuff-->
{{< note title="General stuff" >}}
Change hostname
```
hostnamectl set-hostname $HOSTNAME
```
Must also check the following files and update if necessary
```
vim /etc/hostname
vim /etc/hosts
```
Restart server to take effect
Set servertime
```
timedatectl
```
sets server time to local 'Pacific' time
```
sudo timedatectl set-timezone America/Los_Angeles
```
Turns on colored propmts in terminal
```
sed -i 's/#force_color_prompt=yes/force_color_prompt=yes/g' ~/.bashrc
```
{{< /note >}}


<!--Custom Prompts-->
{{< note title="~/.bashrc custom prompts" >}}
For users:
```
export PS1="\u @\[$(tput sgr0)\]\[\033[38;5;10m\]\H\[$(tput sgr0)\] \A\n[\[$(tput sgr0)\]\[\033[38;5;10m\]\w\[$(tput sgr0)\]] \\$ \[$(tput sgr0)\]"
```
For root:
```
export PS1="\[$(tput bold)\]\[\033[38;5;10m\]\u\[$(tput sgr0)\] @\[$(tput sgr0)\]\[$(tput bold)\]\[\033[38;5;10m\]\H\[$(tput sgr0)\] \A\n[\[$(tput sgr0)\]\[\033[38;5;10m\]\w\[$(tput sgr0)\]] \\$ \[$(tput sgr0)\]"
```
if you see a permission denied error, update permissions of .bashrc
```
chmod 644 ~/.bashrc
```
Set bash as the default shell on ssh login
```
chsch -s /bin/bash
```
need to create a ~/.bash_profile file to load custom prompts with the following
```
[ -f "$HOME/.bashrc" ] && source "$HOME/.bashrc"
```
or through root
```
usermod -s /bin/bash $USERNAME
```
{{< /note >}}


<!--Creating Users-->
{{< note title="Creating users" >}}
Create user 'home' and '.ssh' folders
```
sudo mkdir -p /home/$USERNAME/.ssh
```
Create 'authorized keys' file
```
sudo touch /home/$USERNAME/.ssh/authorized_keys
```
Create user
```
sudo useradd -d /home/$USERNAME $USERNAME
```
Add user to 'sudo' group (alllows sudo privileges)
```
sudo usermod -aG sudo $USERNAME
```
Give user permission over own home folder
```
sudo chown -R $USERNAME:$USERNAME /home/$USERNAME
```
Give root permission of users home folder
```
sudo chown root:root /home/$USERNAME
```
Update permissions on '.ssh' folder and 'authorized_keys' file
```
sudo chmod 700 /home/$USERNAME/.ssh
sudo chmod 644 /home/$USERNAME/.ssh/authorized_keys
```
Set user 'sudo' password for user
```
passwd $USERNAME
```
{{< /note >}}


<!--Copying SSH Key to a Remote Server-->
{{< note title="Copying SSH key to server" >}}
Need to enable 'password authentication' on the server by opening sshd_config
```
sudo vim /etc/ssh/sshd_config
```
Look for 'Passwordauthentication' and update 'no' to 'yes'
Reload sshd
```
sudo service sshd restart
```
On LOCAL MACHINE run
```
ssh-copy-id $USERNAME@$DOMAIN_NAME
```
Alternatively if you get errors you may need to specify the public key file
```
ssh-copy-id -i /home/$USER/.ssh/id_rsa.pub $USER@$DOMAIN_NAME
```
You will be prompted for the 'sudo' password defined earlier
Need to disable 'password authentication'
```
sudo vim /etc/ssh/sshd_config
```
Update 'Passwordauthentication' back to 'no'
Reload sshd
```
sudo service sshd restart
```
{{< /note >}}


<!--Setting up SFTP Access-->
{{< note title="Setting up VSFTPD for FTP access" >}}
```
apt install vsftpd
cp /etc/vsftpd.conf /etc/vsftpd.conf.old
ufw allow 20/tcp
ufw allow 21/tcp
```
configure FTP access
```
vim /etc/vsftpd.conf
```
Update to match the following
```
# Allow anonymous FTP? (Disabled by default).
anonymous_enable=NO
# Uncomment this to allow local users to log in.
local_enable=YES
write_enable=YES
chroot_local_user=YES
```
*ADD*
```
user_sub_token=$USER
local_root=/home/$USER # set to '/' for unlimited access
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
```
restart VSFTP
```
systemctl restart vsftpd
```
Creating users for FTP access
```
adduser $USER
mkdir /home/$USER/
chown $GROUP:$GROUP /home/$USER/
chmod a-w /home/$USER/
```
add user to userlist file
```
echo "$USER" | tee -a /etc/vsftpd.userlist
```
{{< /note >}}


<!--Auth.log Notes-->
{{< note title="Auth.log Notes" >}}
search auth.log for mentions of a particular IP
```
grep "$IP" /var/log/auth.log
```
search auth.log for the term "Connection closed" and make a list of the IP addresses
```
grep "Connection closed" /var/log/auth.log | grep -Po "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | sort | uniq -c
```
{{< /note >}}

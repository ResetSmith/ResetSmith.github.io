---
title: Apache Notes
weight: 10
menu:
  notes:
    name: Apache Notes
    identifier: notes-apache-apache_notes
    parent: notes-apache
    weight: 10
---

{{< note title="General" >}}
Show active Vhost configuration in Apache
```
apache2ctl -S
```
Enable vhost in apache
```
a2ensite $VHOST.conf
```
Disable vhost in apache
```
a2dissite $VHOST.conf
```
Enable SSL in apache
```
a3enmod ssl
```
Check for errors in apache
```
apache2ctl configtest
```
{{< /note >}}

{{< note title="Basic Auth with Apache Virtualhost" >}}
*Note This will NOT work while using wordpress with .htaccess file*
Install Apache utils package
```
apt-get update
apt-get install apache2-utils
```
create password file and user
```
htpasswd -c /etc/apache2/.htpasswd $USERNAME
```
LEAVE OFF -c for additional users
```
htpasswd /etc/apache2/.htpasswd $USERNAME2
```
*Must be added to the 'Directory' section of the Apache VirtualHost file*
```
  AuthType Basic
  AuthName "Restricted Content"
  AuthUserFile /etc/apache2/.htpasswd
  Require valid-user
```
check apache configuration
```
apache2ctl configtest
```
restart apache
```
systemctl restart apache
```
if seeing misconfiguration errors, may need to update ownership of .htpasswd
```
chown www-data:www-data /etc/apache2/.htpasswd
```
{{< /note >}}

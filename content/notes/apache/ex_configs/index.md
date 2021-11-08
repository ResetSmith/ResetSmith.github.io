---
title: Apache Conf Examples
weight: 20
menu:
  notes:
    name: Apache Conf Ex
    identifier: notes-apache-ex_configs
    parent: notes-apache
    weight: 20
---

{{< note title="HTTP Vhost for WordPress" >}}
```
<VirtualHost *:80>
  ServerAdmin  $EMAIL
  ServerName   $FQDN
  ServerAlias  www.$FQDN
  DocumentRoot /var/www/$HOSTNAME
  <Directory /var/www/$HOSTNAME/>
    Options FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
ErrorLog  ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
{{< /note >}}


{{< note title="HTTPS Vhost for WordPress" >}}
```
<VirtualHost *:80>
  ServerName  $FQDN
  ServerAlias www.$FQDN
  Redirect permanent / https://$FQDN/
</VirtualHost>

<VirtualHost *:443>
  ServerAdmin $EMAIL
  DocumentRoot /var/www/$HOSTNAME
  ServerName  $FQDN
  ServerAlias www.$FQDN
  SSLEngine on
  SSLCertificateFile    $PATH_TO_CERTIFICATE
  SSLCertificateKeyFile $PATH_TO_PRIVATE_KEY
  <Directory /var/www/$HOSTNAME/>
    Options FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
ErrorLog  ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Will need to enable the Apache SSL Mod for HTTPS to work
```
sudo a2enmod ssl
```
{{< /note >}}


{{< note title="HTTP Proxy Vhost" >}}
This VHost file will accept requests at $FQDN and then redirect them to the address listed in Proxy/ProxyPass and then applies the $FQDN in the address bar.
```
<VirtualHost *:80>
  ServerName $FQDN
  ProxyRequests Off
  <Location />
    ProxyPreserveHost On
    ProxyPass         http://137.184.11.237:8080/
    ProxyPassReverse  http://137.184.11.237:8080/
  </Location>
ErrorLog  ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Will need to enable the Apache Proxy Mod for this to work
```
sudo a2enmod proxy proxy_http
```
{{< /note >}}


{{< note title="HTTPS Proxy Vhost" >}}
This works similarly to above except it will appply a SSL certificate as well.
```Apache
<VirtualHost *:80>
ServerName storage.casat.org
Redirect Permanent / https://storage.casat.org/
</VirtualHost>

<VirtualHost *:443>
  ServerName storage.casat.org
  SSLEngine On
  SSLCertificateFile    /etc/letsencrypt/live/storage.casat.org/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/storage.casat.org/privkey.pem
  ProxyRequests Off
  <Location />
    ProxyPreserveHost On
    ProxyPass         http://137.184.11.237:8080/
    ProxyPassReverse  http://137.184.11.237:8080/
  </Location>
  # uncomment for SSL
  SSLProxyEngine On  
ErrorLog  ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
This requires a few Apache Mods to be enabled to work
```
sudo a2enmod ssl proxy proxy_http
```
{{< /note >}}


{{< note title="HTTPS Vhost w/ Basic Authentication" >}}
*Note This will NOT work while using wordpress with .htaccess file*
Install Apache utils package
```
apt-get update
apt-get install apache2-utils
```
create password file and user
```
htpasswd -c /etc/apache2/htpasswd.users $USERNAME
```
LEAVE OFF -c for additional users
```
htpasswd /etc/apache2/htpasswd.users $USERNAME2
```
*Must be added to the 'Directory' section of the Apache VirtualHost file*
```
  AuthType Basic
  AuthName "Restricted Content"
  AuthUserFile /etc/apache2/htpasswd.users
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
chown www-data:www-data /etc/apache2/htpasswd.users
```
{{< /note >}}

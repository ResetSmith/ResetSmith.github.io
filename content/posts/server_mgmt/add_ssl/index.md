---
title: "Enabling SSL with Apache"
date: 2021-05-01
hero: development.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Enabling SSL
    identifier: svr_mgmt-add_ssl
    parent: svr_mgmt
    weight: 60
---

# How we have been installing SSL certificates on our Apache servers

## Obtain certificate using Certbot

Adapted from the [Certbot website](https://certbot.eff.org/lets-encrypt)

1. Update snapd
```
sudo snap install core; sudo snap refresh core
```

2. Install Certbot app
```
sudo snap install --classic certbot
```

3. Prepare the Certbot command
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

4. If Apache is currently running, disable it so Certbot can use the port
```
sudo systemctl stop apache2.service
```

5. Test Certbot
```
sudo certbot certonly --dry-run
```
- when prompted, select 'Spin up a temporary webserver'
- when entering the domain name, you can list multiple targets
    - ie: casat.org, www.casat.org, *.casat.org

6. If the test run passes without errors you should be good to run the 'live' command
```
sudo certbot certonly
```  
- If you followed the above commands the certificates should be saved to the certbot default location: /etc/letsencrypt/live/$DOMAIN_NAME/

## Create new Apache Vhost file for SSL

For Apache to serve HTTPS content we must update the VirtualHost file with support for connections over port 443, and references to the locations of the SSL certificates. For this we will keep our existing VirtualHost file in tact and create a new .conf that supports SSL connections instead.

1. Create new Vhost .conf file
```
sudo vim /etc/apache2/sites-available/$DOMAIN_NAME-ssl.conf
```
So in practice that location should read something like: /etc/apache2/sites-available/casat-ssl.conf

2. Fill in Vhost information
```
<VirtualHost *:80>
ServerName  $FQDN
ServerAlias www.$FQDN
Redirect permanent / https://$FQDN/
</VirtualHost>

<VirtualHost *:443>
ServerAdmin webmaster@casat.org
DocumentRoot /var/www/html/$HOSTNAME
ServerName  $FQDN
ServerAlias www.$FQDN
SSLEngine on
SSLCertificateFile    /etc/letsencrypt/live/$FQDN/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/$FQDN/privkey.pem
    <Directory /var/www/html/$HOSTNAME/>
    Options FollowSymLinks
    AllowOverride All
    Require all granted
    </Directory>

ErrorLog  ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

3. Enable SSL support in Apache
```
sudo a2enmod ssl
```

4. Disable any current Vhost files and enable the new SSL file
```
sudo a2dissite $CURRENT_VHOST.conf
sudo a2ensite $NEW_VHOST-ssl.conf
```

5. Check that Apache doesn't have any problems with your syntax in the Vhost file
```
sudo apache2ctl configtest
```

6. Restart the Apache service to activate the changes
```
sudo systemctl restart apache2.service
```

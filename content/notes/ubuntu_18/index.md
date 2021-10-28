---
title: Ubuntu 18 Notes
menu:
  notes:
    name: Ubuntu 18
    identifier: notes-ubuntu_18
    weight: 90
---

{{< note title="Updating PHP" >}}
I have had problems getting PHP 7.2 to update to 7.4 on Ubuntu 18.04. Here's what I did to get it working.

1. Manually add PHP repository and check again
```
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt upgrade
```

2. Check PHP version again
```
php -v
```

If PHP still shows 7.2 then 7.4 needs to be manually installed

3. Install PHP 7.4
```
sudo apt install php7.4
```

4. Enable PHP 7.4
Enable PHP 7.4 and then disable PHP 7.2 and then restart Apache
```
sudo a2enmod php7.4
sudo a2dismod php7.2
sudo systemctl restart apache2.service
```

Check PHP version again
```
php -v
```
It should now show PHP 7.4
{{< /note >}}

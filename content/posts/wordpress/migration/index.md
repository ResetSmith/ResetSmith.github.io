---
title: "Migrating a WordPress website"
date: 2021-04-05T10:03:55-07:00
hero: post.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: WordPress Migrations
    identifier: wordpress-migration
    parent: wordpress
    weight: 20
---

When migrating a WordPress website from one server/VPS to another there are two things we need to make sure we capture; the MySql database and the wp-content folder. The instructions below will walkthrough copying both the MySql database and the wp-content folders from an existing WordPress installation to another 'blank' WordPress installation on a different VPS.

## Copying the data

*This should be done on the server you want to copy.*

First thing we are going to do is create a folder in the /tmp/ directory to work out of. Navigate to the /tmp/ directory and make a folder, I usually use the website domain name in the folder name, ie: casat_copy.
```
cd /tmp
sudo mkdir $HOST_COPY
```

Now we will copy the wp-content folder from our WordPress installation into our $HOST_COPY folder in the /tmp/ directory.
```
sudo cp -avr /var/www/$DOMAIN_NAME/wp-content /tmp/$HOST_COPY
sudo mysqldump -u root -p $DATABASE_NAME > /tmp/$HOST_COPY/database.sql
```

Next we will make a copy of the MySql database and put it in the $HOST_COPY folder. Historically I've had trouble getting the database permissions correct for this command to run as a user. So before running the command to export the MySql database we will first use a command to elevate ourselves to the 'root' user on the server (su), then run our command (mysqldump...), and then exit the 'root' user back our own user (exit).
```
su
mysqldump -u root $DATABASE_NAME > /tmp/$HOST_COPY/database.sql
exit
```

{{< alert type="info" >}}
**Database Name**\
If you don't know what the database name is you can find it by viewing the wp-config.php file. It can be found in the WordPress root directory, commonly /var/www/$DOMAIN/wp-config.php.
{{< /alert >}}

---

## Exporting the data

Now that we have the data copied we need to package it together to make it easier to move. The following command will compress the folder together into a single 'tarball' file.
```
tar -czvf $HOST_COPY.tar.gz /tmp/$HOST_COPY
```

With the tarball file made, we'll use 'scp' (secure copy) to send the files from one server to another. The command itself is rather simple, however you will likely have to make some adjustments on the new server to allow for a less secure connection to make this work. The scp command is listed below.
```
scp $HOST_COPY.tar.gz $USER@$NEW_SERVER:~
```
An example would look like this `scp casat.tar.gz jmarks@123.56.89.111:~`

{{< alert type="warning" >}}
**Allowing Less Secure Connections**\
If the 'scp' command is failing, it's likely that the target server (new server) only allows connections with verified SSH keys (as it should). In order to get around this you will likely need to enable 'PasswordAuthentication' to let you scp into the server using your 'sudo' password.

To enable 'PasswordAuthentication' first connect to the server you want to enable it on, and then open the sshd_config file.
```
sudo vim /etc/ssh/sshd_config
```
From there you will want to find the line
```
PasswordAuthentication no
```
Update the 'no' to a 'yes' and then save the file. We then need to reload the sshd module for the changes to take affect.
```
sudo systemctl restart sshd.service
```
Now we should be able to use the 'scp' command to move our file from our old server to the new. When prompted for a password during the 'scp' you should use your 'sudo' password on the target server.

**Once the file is copied you need to disable PasswordAuthentication.**\
Reopen the sshd_config file and change PasswordAuthentication back to no, and then run the following command to apply the change
```
sudo systemctl restart sshd.service
```
This disables PasswordAuthentication and restricts access to SSH only again.
{{< /alert >}}

After you have confirmed the files are on the new server you should clean up the files you left in /tmp/. Running the following command from within /tmp/ on the old server will clear both the tarball file and the temporary copy folder we made.
```
sudo rm -rf /tmp/$HOST_COPY.tar.gz $HOST_COPY
```

---

## Importing the data

*This should be done on the new server*

The final steps on the new server are to unpack the tarball file we copied from the old server, import the MySql database onto our current one, and then replace the content of our existing /wp-content folder with the copy. After all that we will go back to the MySql database and update any URL references to point to the correct domain name.

Navigate to the folder with the copied tarball file, if you folowed the steps above the copy should be in /home/$USER/. First we will copy the file to /tmp/, then navigate to that folder, then decompress the tarball
```
sudo mv $HOST_COPY.tar.gz /tmp/
cd /tmp
tar -xzvf $HOST_COPY.tar.gz
```

Take the copy MySql database and overwrite the existing database
```
sudo mysql -u root -p wp_$EXISTING_DB < /tmp/$HOST_COPY/database.sql
```

Next we need to remove the current /wp-content folder from our WordPress installation, then we copy the /wp-content folder from $HOST_COPY
```
sudo rm -r /var/www/$DOMAIN_NAME/wp-content/
sudo cp -ar /tmp/$HOST_COPY/wp-content /var/www/$DOMAIN_NAME/
```

Since this folder is being copied from another source we need to update the permissions so the www-data user can continue to access them
```
sudo chown -R www-data:www-data /var/www/$DOMAIN_NAME/
sudo chmod -R 755 /var/www/html/$DOMAIN_NAME/
```

---

## Updating the URL in the database

If the website that we copied the data *from* had a different URL from the one we are copying the data *to* then we need to update the URL in the MySql database. The easiest and most comprehensive way to do this is using the wp-cli app. Alternatively, in some cases, it can just be done from within MySql.

### Updating the URL with wp-cli

wp-cli is a powerful tool for managing WordPress website from the Terminal. It can do and allow you to automate all sorts of functions. For this particular case we will be using wp-cli's 'search and replace' feature. If you followed the WordPress server installation [on here](http://hugo.casatdev.com/it_notes/new_server_config/LAMP_03/#installing-wp-cli), then wp-cli should already be installed. Otherwise it is [available here.](https://wp-cli.org)

In order to update the URL for the website, we first navigate to the WordPress root folder, /var/www/$DOMAIN_NAME/. From there we will run the following command
```
wp search replace $OLD_DOMAIN $NEW_DOMAIN --dry-run
```
Make sure you replace the domain name variables with the complete 'fully qualified domain name' (FQDN). The $OLD_DOMAIN needs to be whatever the current URL of the server is, typically that will either be an IP address or a 'dev' subdomain. Here's a couple of examples
```
wp search replace 123.567.89.211 https://casat.org --dry-run
wp search replace http://dev.casat.org http://casat.org --dry-run
```
The --dry-run flag at the end, is there to say that this is a test. We want to run the command and then check what the output is prior to running the command in 'production'. Run the command and check the result, if you have any error notices fix those and then test again. Once you have the command working cleanly, run it without the --dry-run flag and that will update the URLs across the entire site.

### Updating the URL with MySql

It is also possible to update the URL directly within MySql itself, however this should only be done as a last resort if needed.

{{< alert type="danger" >}}
**Updating MySql Manually**\
Because of the way that WordPress stores information within the MySql database it's possible that this can cause additional issues. Many plugins use 'serialized data' within the database, this means that the data is spaced in a way to create sections. When updating the MySql URL manually it is possible to screw up the spacing of those sequences and potentially break other things. In order to avoid this its recommended to use the wp-cli search and replace function instead.
{{< /alert >}}

Connect to the MySql app using the root account for MySql
```
mysql -u root -p
```
Once we are in MySql we first need to connect to the WordPress database
```
USE wp_$HOST_NAME;
```
Then we will update the URL in two specific places
```
UPDATE wp_options SET option_value = 'https://$DOMAIN_NAME' WHERE option_name = 'siteurl';
UPDATE wp_options SET option_value = 'https://$DOMAIN_NAME' WHERE option_name = 'home';
```
After that we can leave MySql
```
exit;
```
Then finally restart the MySql database for the changes to take affect.
```
systemctl restart mysql
```

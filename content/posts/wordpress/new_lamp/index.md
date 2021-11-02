---
title: "Setting up a LAMP server for WordPress"
date: 2021-04-04T12:18:47-07:00
hero: post.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Deploying a LAMP server
    identifier: wordpress-new_lamp
    parent: wordpress
    weight: 10
---

This tutorial will walkthrough the process of installing Apache, MySql, and PHP on an Ubuntu 20.04 DigitalOcean Droplet. The server created here will eventually host a WordPress website, I will indicate when settings are specific to WordPress and you can alternate from there if you are doing something different. These directions are for DigitalOcean but should be similar for most VPS providers. Before starting this process you should obtain an [Ubuntu 20.04 droplet](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ubuntu-20-04-server-on-a-digitalocean-droplet) from DigitalOcean and make sure your [SSH access is working.](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/)

{{< alert type="info" >}}
**Ubuntu Versions**\
I recommend starting with an Ubuntu 20.04 (LTS) server instead of other versions because the LTS version is the 'Long Term Support' version of Ubuntu, meaning it is supported with at least 5 years of updates. Non-LTS releases are only guaranteed to receive 9 months of updates.
{{< /alert >}}

## Setting up a LAMP server

By starting out with a Linux (Ubuntu) server from DigitalOcean we already have the 'L' for our 'LAMP' server, now we'll move onto the 'A'. Apache is the most common web server in use with WordPress websites, and the one we'll be using for this tutorial. To begin with we'll need to download and install Apache onto our server. The first step is to make sure you are connected to your webserver [over SSH](https://docs.digitalocean.com/products/droplets/how-to/connect-with-ssh/) or through the ['Droplet Console'](https://docs.digitalocean.com/products/droplets/how-to/connect-with-console/).

### Installing Apache

Once you are connected to your server, the first thing you should do is check for updates and then run any updates. You will check for updates with the following command:
```
sudo apt update
```
Once that is down checking, we'll run the waiting updates with
```
sudo apt upgrade
```
If you are asked to confirm any updates, just go with 'yes' or whatever default is already checked. Since it is a new server there are no settings we need to worry about preserving.

With the server updated we can get Apache installed. We'll run this command to download it
```
sudo apt install apache2
```

And when once that is completed you can navigate to the IP address of the server in your web browser and should be greeted with the Apache2 Default Page. There is still some work to be done in order to get Apache to properly serve up a webpage. But this is all we need for now, we'll create VirtualHosts and configure it on a later step.

### Installing MySql

Next up for our LAMP is the 'M', MySql. MySql is a Database application that help us organize our content for our future website. There are many Database options, but MySql is the most common for WordPress so that's what we'll use here.

First up is to download the mysql-server app by running
```
sudo apt install mysql-server
```
Once MySql is downloaded we should go through the initial setup. Run this command to have MySql walk you through the initial settings
```
mysql_secure_installation
```

You will be prompted through a series of questions, the first should be if you would like to enforce password strength requirements (I recommend yes). Next you will be prompted to define a 'root' password, make sure you copy this down you will most likely need it in the future. After you set the password you will be asked to remove anonymous users, you should. You should also disallow remote root logins, and finally go ahead and reload the privilege tables when prompted which updates the currently logged in accounts to match the new settings.

After that we'll be done with MySql for now.

### Installing PHP

The final piece of our LAMP server is the 'P', for PHP. PHP is a programming scripting language that WordPress and many other Web applications are built on. Since this tutorial is about getting WordPress installed we'll be installing PHP and some necessary add-ons for WordPress.

```
sudo apt install php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip
```

These are some of the commonly needed PHP extensions. It's possible that you may need other or additional extensions for particular WordPress plugins, make sure to check your documentation.

---

## Configuring the LAMP server

Now that there is a LAMP server setup we need to configure Apache, Mysql, and PHP for WordPress. For these steps it's best if you already have a web address registered (www.google.com, casat.org, etc...) and DNS already setup to point to the correct server. From here on the web address will be referred to as the FQDN (fully qualified domain name).

### Configuring the MySql database

The first thing we will do is setup a MySql database and a user for WordPress to access the database from. For this particular section you will be providing 3 variables, $WPDBNAME is the name for the WordPress database we'll be creating, I like them to reference the website to keep it simple (something like wp_google), $WPUSERNAME is the name for the user that WordPress uses to interact with the database and again I typically reference the FQDN to keep it simple (ex: user_google), last you will need to come up with a password for that user account it will be referred to as $WPPASSWORD. When you see one of these variables in the commands you should replace it with the correct information.

Connect to your server via SSH or the Droplet Console, and then connect to the MySql app by typing in
```
sudo mysql -u root
```
If you are prompted for a password, it is looking for your Ubuntu 'Sudo' password.

Now we'll create the database that WordPress will store all of your website data and users. Make sure to replace $WPDBNAME with an appropriate name for your WordPress website database.
```
CREATE DATABASE $WPDBNAME DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
```
This will create a database in MySql named '$WPDBNAME' and then set the Character Set and Collate options to utf8 which is what WordPress needs.

Then we will create a user that WordPress can use to access the database. For this command you will need to replace the variables $WPUSERNAME and $WPPASSWORD with a Username and Password respectively. Make sure to copy down what the Username and Password are as they will be used later in the WordPress installation.
```
CREATE USER '$WPUSERNAME'@'localhost' IDENTIFIED WITH mysql_native_password BY '$WPPASSWORD';
```
This creates a user named $WPUSERNAME and only allows that account to connect to the database Locally, which prevents someone from connecting to the database remotely using that account. It then sets the user's password to $WPPASSWORD. The mysql_native_password modifier tells MySql to create the user with the older, less secure, password hashing format that is necessary for WordPress as of right now.

Next we will grant permission for the user account we just made to access the database. For this you will need the database name you created, and the username.
```
GRANT ALL ON $WPDBNAME.* TO '$WPUSERNAME'@'localhost';
```

For the final MySql steps we will reset the user permissions and then exit the MySql interface and return to the Ubuntu terminal using the following commands
```
FLUSH PRIVILEGES;
EXIT;
```
Now you should be back in the regular Ubuntu terminal. Next we'll fix our PHP installation to serve up WordPress content.

### Creating an Apache VirtualHost

In order to configure Apache for WordPress we will create a new VirtualHost file, enable that VirtualHost, check our Apache setup for errors and then enable the new Apache configuration.

First thing is to create a new VirtualHost file for the website. The file can be named whatever you want it to be, as long as it ends with the .conf suffix. I prefer to use the name of the website to keep it simple. In this example replace the $VHOSTNAME variable with your website (ie: google.conf, digitalocean.conf)
```
sudo touch /etc/apache2/sites-available/$VHOSTNAME.conf
```

Next we will fill the content of the VirtualHost file by opening it with a text editor. Personally I like to use vim, but nano works just as well if that's what you are familar with. Use this command to open the VirtualHost file, replace vim with the text editor of choice if you'd like to use something different, and $VHOSTNAME with the name of the file you just created.
```
sudo vim /etc/apache2/sites-available/$VHOSTNAME.conf
```

Now we will create the VirtualHost content. I'm including a very basic VirtualHost example below that can be copy and pasted. However there are several variables that need to be edited to work with your respective website. In the top section the variable $ADMIN_EMAIL should be replaced with an active email address. The $DOMAIN_NAME variable should be replaced with your domain name (ie: google.com).

{{< alert type="info" >}}
**www vs non-www**\
In my example the website is being setup for non-www, if you would prefer your site to be www then you should put www.$DOMAIN_NAME in the ServerName field and the non-www in the ServerAlias field.
{{< /alert >}}

The other variable you need to define is $WPFOLDER. This folder does not exist yet, we will create it later when we do our actual WordPress installation. Typically I name this folder the same as my website which helps keeps things straight if you will eventually host multiple sites from the same server. If this is not a concern for you, you can just name this folder wordpress. Whatever you choose make sure you remember what it is, as we will need to reference it later.
```
<VirtualHost *:80>
ServerAdmin  $ADMIN_EMAIL
ServerName   $DOMAIN_NAME
ServerAlias  www.$DOMAIN_NAME
DocumentRoot /var/www/$WPFOLDER

 <Directory /var/www/$WPFOLDER/>
  Options FollowSymLinks
  AllowOverride All
  Require all granted
 </Directory>

ErrorLog  ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Once we have our VirtualHost file created we need to tell Apache to enable that VirtualHost.
```
sudo a2ensite $VHOSTNAME.conf
```

Then we need to disable the default VirtualHost file that Apache comes with.
```
sudo a2dissite 000-default.conf
```

Before proceeding further you can check the syntax of your Apache Virtualhost file by running this command. If you receive any errors you will need to correct them before moving on.
```
sudo apache2ctl configtest
```

You will get a warning that the current DocumentRoot folder does not exist. That is correct, we will create that folder in a later step. Now we will enable the rewrite module in Apache that is required for WordPress.
```
sudo a2enmod rewrite
```

And finally we will restart Apache to apply the new settings.
```
sudo systemctl restart apache2
```

At this point we are now ready to install WordPress.

---

## Installing WordPress

For the final step in getting WordPress installed on our DigitalOcean LAMP VPS we need to install WordPress.

### Download and unpack WordPress

To download WordPress, we will first navigate to the /tmp folder on the server and then download
```
cd /tmp
curl -O https://wordpress.org/latest.tar.gz
```

WordPress comes downloaded in a compressed form (tar.gz) so we will unpack it with tar and leave it in it's own folder in /tmp
```
tar xzvf latest.tar.gz -C /tmp/.
```

With that you will have a new /tmp/wordpress folder and all the WordPress files and folders are in there. Now we will create a couple of files that are necessary for WordPress and make a copy of one that WordPress provides a template for.
update files and folders
```
sudo touch /tmp/wordpress/.htaccess
sudo mkdir /tmp/wordpress/wp-content/upgrade
sudo cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php
```

### WordPress config file

Now that we have copied the WordPress wp-config.php template we need to do a bit of setup and configure it to talk to the MySql database that we created previously. For this you will need to know the WordPress database name, and the username and password we created in the previous steps, specifically the ones we used for $WPDBNAME, $WPUSERNAME, and $WPPASSWORD.

Open wp-config.php with a text editor
```
sudo vim /tmp/wordpress/wp-config.php
```
From there we'll begin replacing things, first thing is the MySql database name, $WPDBNAME. Find the line that matches below and replace database_name_here with your MySql WordPress database name (leave the '' ticks).
```
define( 'DB_NAME', 'database_name_here' );
```

Next we'll do the same with the MySql database user we created, $WPUSERNAME. Scroll down wp-config.php and find the matching line and replace username_here with your MySql user
```
define( 'DB_USER' 'username_here' );
```

And we'll do the same with the MySql user password, $WPPASSWORD, we created earlier as well. Find the matching line and replace password_here with the actual password
```
define( 'DB_PASSWORD', 'password_here' );
```

With that done we need to do one more thing for wp-config and that is download the Salts. Salts are what WordPress uses to cryptographically randomize yours and your users passwords when they are stored in memory. For our purposes you will run the command below and then copy the output which we will then paste into our wp-config.php file.
```
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```
With the salt contents copied, we will open up the wp-config.php file one more time in our text editor
```
sudo vim /tmp/wordpress/wp-config.php
```
And then scroll down till you find the following section
```
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );
```
Delete these 8 lines and replace them with the salt contents you copied earlier and then save the changes.

### Copying WordPress to its proper location

Our WordPress files are now ready to be copied to our live location. In the previous step when setting up our Apache VirtuaHost file we defined a DocumentRoot location, that is where you need to copy the WordPress files to. The command should look something like this
```
sudo cp -a /tmp/wordpress/. /var/www/$WPFOLDER
```

Once the files have been copied we need to update them to have the correct permissions. Run the following commands to update the permissions
```
sudo chown -R www-data:www-data /var/www/$WPFOLDER/
sudo chmod -R 755 /var/www/$WPFOLDER/
sudo find /var/www/$WPFOLDER/ -type d -exec chmod 750 {} \;
sudo find /var/www/$WPFOLDER/ -type f -exec chmod 640 {} \;
sudo find /var/www/$WPFOLDER/ -type d -exec chmod g+s {} \;
```

Now you should be able to navigate to your domain and see the WordPress welcome page.

---

## Installing wp-cli

wp-cli is a tool for managing WordPress from the terminal. We use it to manage WordPress when the web interface is inaccessible and to apply bulk changes to the MySQL database after a server migration. You can find more information on the [wp-cli website.](https://wp-cli.org/)

{{< alert type="info" >}}
**Running commands**\
To use wp-cli you navigate to the WordPress web root folder, for our websites that should be /var/www/$DOMAIN_NAME/. From there you can run the wp-cli commands in the terminal. You can find a list of commands on the [WordPress website.](https://developer.wordpress.org/cli/commands/)
{{< /alert >}}

Download the wp-cli app
```
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
```

Make the wp-cli app executable
```
sudo chmod +x wp-cli.phar
```

Move the app to the proper location for the app to be run by all users.
```
mv wp-cli.phar /usr/local/bin/wp
```

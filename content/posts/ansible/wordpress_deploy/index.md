---
title: "Automate Deploying WordPress with Ansible"
date: 2022-10-03
hero: 504_Error_Gateway_Timeout-rafiki.png
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: WordPress Ansible Deployment
    identifier: ansible-wordpress
    parent: ansible
    weight: 15
#draft: true
---


Ansible is a framework for automating deployment and maintenance of remote systems. This tutorial will go through the creation of my own WordPress playbook for Ansible. You can find all of my WordPress playbooks in Github in my [Playbooks repository](https://github.com/ResetSmith/playbooks).

## wp_install.yml playbook

The WordPress installer playbook I created is the wp_install.yml playbook. The playbook handles the installation and configuration of Apache, MySQL, PHP, and WordPress on top of an Ubuntu server (Tested on 22.04). This playbook is friendly to being run on multiple targets at once as long as the prerequisites have been completed.

This tutorial will go through the playbook in sections, but you can find a complete copy of the [playbook in Github](https://github.com/ResetSmith/playbooks/blob/main/wordpress/wp_install.yml).

---

## The Header

Starting from the top there is the section containing the Inventory File and Variable definitions along with the first directive to run the playbook as Sudo.

```yaml
---
- hosts: all
  become: true
  vars_files:
    - ../secrets/mysql_vars.yml
```

So for this playbook I have the hosts set to 'all'. I do have my Ansible hosts file setup for different groups but by-and-large the bulk of my servers are WordPress servers so I leave the hosts directive set to 'all' so this playbook will happily run on any server I target regardless of which group within the hosts file it belongs to.

The 'become: true' directive is about running commands as Sudo, and running it at this level applies Sudo to every task below. This is one of my first playbooks I wrote, and since then have gotten away from applying Sudo at this level, and instead prefer to add this directive to the particular tasks that actually need it. At some point I will revisit this playbook and update the Sudo method but for now this works, and you just need to recognize that all the tasks below will be performed as Sudo. 

The 'vars_files:' directive looks at a Variable file I have saved in another location. If you are following my same Playbook folder organization the 'mysql_vars.yml' file should be located in /playbooks/secrets/mysql_vars.yml, which you will need to create if you have cloned by playbooks from github.  The vars file contains the variables needed for the Geerlingguy.mysql role we'll be running later. But below is an example of what that file should look like, I'll go into more detail further on.

```yaml
# These are the variables needed for the GeerlingGuy MySQL role

# MySQL root access password
mysql_root_password: $MYSQL_PASSWORD
# Uncomment the line below if you need to overwrite an existing root user password
# mysql_root_password_update: true
mysql_enabled_on_startup: true

# This creates the database needed for WordPress
mysql_databases:
  - {name: 'wordpress', encoding: 'utf8', collation: 'utf8_general_ci'}

# This creates the user WordPress will use to access the database
mysql_users:
  - {name: '$WP_USER_NAME', password: '#WP_USER_PASSWORD', host: '%', priv: 'wordpress.*:ALL'}
```

That's it for the 'header' section of the wp_install, next up is the 'Tasks' section. The Tasks section of the playbook is list of directives that are actually taking some kind of action. This will become more clear as you review the directives themselves.

---

## Initial Tasks

```yaml
  tasks:
# Checks for server updates and runs them
    - name: Updating apt repo and cache on all servers
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: Updating packages
      apt: upgrade=dist force_apt_get=yes
```

This is the start of our Ansible playbooks tasks section, as you can tell by the 'tasks:' directive and underneath that the tasks are listed out in order that they should run. The first of these two tasks checks for updates, the 'cache_valid_time' directive says that if the last check for updates occurred in the last 3600 seconds (60 minutes) then do not pull updates from the web/repos. The second task here simply installs the updates found by the previous task.

```yaml
# Installs dependencies needed for WordPress: Apache, MySQL and PHP
    - name: Installing Apache and PHP
      apt: name={{ item }} update_cache=yes state=latest
      loop: [ 'apache2', 'python3-pymysql', 'php', 'php-mysql', 'libapache2-mod-php' ]
# Installs PHP extensions needed for WordPress
    - name: Installing PHP extensions
      apt: name={{ item }} update_cache=yes state=latest
      loop: [ 'php-curl', 'php-gd', 'php-mbstring', 'php-xml', 'php-xmlrpc', 'php-soap', 'php-intl', 'php-zip', 'php-imagick' ]
# Restarts the system prior to running the geerlingguy.mysql role
    - name: Restarting system prior to running Geerlingguy.MySQL role
      reboot:
        connect_timeout: 5
        reboot_timeout: 300
        post_reboot_delay: 30
        test_command: uptime
```

Now that we have the updates out of the way the next two tasks take care of installing some of the necessary dependencies for a WordPress installation. The first task installs Apache2 and PHP, along with the python3 MySQL Driver, the PHP MySQL module, and the PHP module for Apache2. The second task installs the PHP plugins needed by WordPress. The final task in this sequence reboots the server prior to moving on to the next tasks. I've found this restart is necessary to prevent errors further into the installation.

---

```yaml
# Uses the pre-created role by GeerlingGuy to install mysql
# The role is installed by Ansible in the '/etc/ansible/galaxy.roles/' folder
# Variables needed for this role are defined in '/etc/ansible/playbooks/secrets/mysql_vars.yml'
    - name: Using geerlingguy.mysql role to install and configure mysql
      include_role:
        name: geerlingguy.mysql
```

After the restart the next Task is to install MySQL. I had quite a bit of trouble getting the MySQL installation to work correctly via Ansible, and eventually settled on using an Ansible Role for MySQL installations created by [Jeff Geerling](https://github.com/geerlingguy) who has written many Ansible Roles for public use. The specific role I am using here is [Ansible-role-mysql](https://github.com/geerlingguy/ansible-role-mysql). You will need to have downloaded and installed this role prior to running this playbook, along with creating the necessary variable file created at the start of this tutorial. The MySQL role has quite a few tasks to go through, so just let it do it's thing and once completed the playbook will move on the next tasks.

---

```yaml
# Apache Configuration
# Creates Apache VirtualHost file, using the ansible hostname
# The ansible hostname is determined automatically from the server's hostname
# The server's hostname is set during creation in DigitalOcean or by using the 'hostnamectl' command on the server prior to running this playbook
    - name: Creating the Apache VirtualHost file
      template:
        src: "files/apache.conf.j2"
        dest: /etc/apache2/sites-available/wordpress.conf
      notify: Reload Apache
# Enables the Apache rewrite module, necessary for WordPress
    - name: Enabling the Apache rewrite module
      shell: /usr/sbin/a2enmod rewrite
      notify: Reload Apache
# Enables the new Apache VirtualHost file and disables the default one
    - name: Enabling the new site in Apache
      shell: /usr/sbin/a2ensite wordpress.conf
      notify: Reload Apache

    - name: Disabling the default Apache site
      shell: /usr/sbin/a2dissite 000-default.conf
      notify: Restart Apache
```

---

## Apache Configuration Tasks

Now that we have most the of the necessary pieces installed, the next tasks will focus on editing or creating the configuration files for previously installed applications.

```yaml
# Apache Configuration
# Creates Apache VirtualHost file, using the ansible hostname
# The ansible hostname is determined automatically from the server's hostname
# The server's hostname is set during creation in DigitalOcean or by using the 'hostnamectl' command on the server prior to running this playbook
    - name: Creating the Apache VirtualHost file
      template:
        src: "files/apache.conf.j2"
        dest: /etc/apache2/sites-available/wordpress.conf
      notify: Reload Apache
# Enables the Apache rewrite module, necessary for WordPress
    - name: Enabling the Apache rewrite module
      shell: /usr/sbin/a2enmod rewrite
      notify: Reload Apache
# Enables the new Apache VirtualHost file and disables the default one
    - name: Enabling the new site in Apache
      shell: /usr/sbin/a2ensite wordpress.conf
      notify: Reload Apache

    - name: Disabling the default Apache site
      shell: /usr/sbin/a2dissite 000-default.conf
      notify: Restart Apache
```

The first set of tasks in the 'configuration' section focuses on Apache. The initial task creates a simple Apache VirtualHost file to help route web traffic requests to our eventual WordPress website. The VirtualHost file is copied from an existing file that can be found on Github within my [WordPress playbooks files](https://github.com/ResetSmith/playbooks/blob/main/wordpress/files/apache.conf.j2) folder (It would also be downloaded automatically if you clone this playbook from github). Within the conf file you should update the 'ServerAdmin' directive to your own email address, the rest of the conf file should be left alone. 

{{< alert type="info" >}}
**Ansible Facts**\
Be aware that the 'ServerName' and 'ServerAlias' directives are being set programmatically by Ansible based on what your Server's hostname is set to. Make sure to have the server hostname defined properly prior to running this playbook. 
{{< /alert >}}

After creating the new wordpress.conf file, the next task enables the Apache rewrite module required by WordPress. Then Ansible goes on to tell Apache to enable the newly created wordpress.conf and disables the 000-default.conf file that Apache ships with.

```yaml
# Adds Apache connections to the list for Fail2Ban to monitor
# This task checks for a Fail2Ban installation in the default location
    - name: Checks for Fail2Ban installation
      stat:
        path: /usr/bin/fail2ban-client
      register: f2b_check

# The following task is skipped if no Fail2Ban installation is found
    - name: Adding Apache connections to Fail2Ban firewall monitoring list
      when: f2b_check.stat.exists
      blockinfile:
        path: /etc/fail2ban/jail.local
        block: |
          # Apache jail
          #
          [apache]
          enabled   = true
          port      = http,https
          filter    = apache-auth
          logpath   = /var/log/apache*/*error.log
          maxretry  = 3

          [apache-overflows]
          enabled   = true
          filter    = apache-overflows
          logpath   = /var/log/apache*/*error.log

          [apache-badbots]
          enabled   = true
          filter    = apache-badbots
          logpath   = /var/log/apache*/*access.log
      notify: Restart Fail2ban
```

This block is only relevant if you are using Fail2Ban to help manage your Firewall. The initial task in this section checks if Fail2Ban is installed on the server (in the default location). If it is then it will add a few rules for monitoring Apache traffic. If a Fail2Ban installation is not found then nothing is changed.

## Installing WordPress

The next several tasks are all for the installation of WordPress.

```yaml
# Install WordPress
# If an existing WordPress installation is found at /var/www/wordpress then the next several tasks are skipped
    - name: Checks for existing WordPress installation
      stat:
        path: /var/www/wordpress
      register: wp_check

# Creates WordPress root folder and assigns the correct permissions
    - name: Creating the WordPress root directory
      when: not wp_check.stat.exists
      file:
        path: /var/www/wordpress
        state: directory
        owner: "www-data"
        group: "www-data"
        mode: '0755'

# Downloads WordPress and installs the files to the wordpress root directory
    - name: Downloading and unpacking the latest version of WordPress
      when: not wp_check.stat.exists
      unarchive:
        src: https://wordpress.org/latest.tar.gz
        dest: /var/www/
        remote_src: yes
```

The initial task in this sequence checks for an existing WordPress installation in the same location (/var/www/wordpress), if one is found then the next several tasks are skipped. The next task creates the wordpress folder assigns the correct permissions and gives ownership to the Apache www-data user account. The final task in that sequence downloads WordPress into the /var/www/wordpress folder.

```yaml
# Copies the 'wp-config.php' and '.htaccess' files to the wordpress root directory
# Variables used in the wp-config.php file are pulled from /secrets/mysql_vars.yml
    - name: Copying updated wp-config.php from file
      when: not wp_check.stat.exists
      template:
        src: "files/wp-config.php.j2"
        dest: /var/www/wordpress/wp-config.php

    - name: Updating the MySQL database reference in wp-config.php
      when: not wp_check.stat.exists    
      replace:
        path: /var/www/wordpress/wp-config.php
        regexp: 'MYSQL_DBNAME'
        replace: "{{ item.name }}"
      with_items: "{{ mysql_databases }}"

    - name: Updating the MySQL user name in wp-config.php
      when: not wp_check.stat.exists      
      replace:
        path: /var/www/wordpress/wp-config.php
        regexp: 'MYSQL_UNAME'
        replace: "{{ item.name }}"
      with_items: "{{ mysql_users }}"

    - name: Updating the MySQL user password in wp-config.php
      when: not wp_check.stat.exists       
      replace:
        path: /var/www/wordpress/wp-config.php
        regexp: 'MYSQL_PASS'
        replace: "{{ item.password }}"
      with_items: "{{ mysql_users }}"
# Copies .htaccess
    - name: Copy .htaccess from file
      when: not wp_check.stat.exists
      template:
        src: "files/htaccess.j2"
        dest: /var/www/wordpress/.htaccess
```

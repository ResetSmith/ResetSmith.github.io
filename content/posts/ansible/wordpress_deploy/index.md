---
title: "Deploying WordPress with Ansible"
date: 2022-04-13
hero: 504_Error_Gateway_Timeout-rafiki.png
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Ansible Backups
    identifier: ansible-backup
    parent: ansible
    weight: 15
#draft: true
---


Ansible is a framework for automating deployment and maintenance of remote systems. This tutorial will go through the creation of my own WordPress playbook for Ansible. You can find all of my WordPress playbooks in Github in my [Playbooks repository](https://github.com/ResetSmith/playbooks).

## wp_install.yml playbook

The WordPress installer playbook I created is the wp_install.yml playbook. The playbook handles the installation and configuration of Apache, MySQL, PHP, and WordPress on top of an Ubuntu server (Tested on 22.04). This playbook is friendly to being run on multiple targets at once as long as the prerequisites have been completed.

This tutorial will go through the playbook in sections, but you can find a complete copy of the [playbook in Github](https://github.com/ResetSmith/playbooks/blob/main/wordpress/wp_install.yml).

Starting from the top there is the section containing the Inventory File and Variable definitions along with the first directive to run the playbook as Sudo.

```
---
- hosts: all
  become: true
  vars_files:
    - ../secrets/mysql_vars.yml
```

So for this playbook I have the hosts set to 'all'. I do have my Ansible hosts file setup for different groups but by-and-large the bulk of my servers are WordPress servers so I leave the hosts directive set to 'all' so this playbook will happily run on any server I target regardless of it's group within the hosts file. The 'become: true' directive is about running commands as Sudo, and running it at this level applies Sudo to every task below. This is one of my first playbooks I wrote, and since then have gotten away from applying Sudo at this level, and instead prefer to add this directive to the particular tasks that actually need it. At some point I will revisit this playbook and update the Sudo method but for now this works, and you just need to recognize that all the tasks below will be performed as Sudo. The 'vars_files:' directive looks at a Variable file I have saved in another location. If you are following my same Playbook folder organization the 'mysql_vars.yml' file should be located in /playbooks/secrets/mysql_vars.yml, which you will need to create if you have cloned by playbooks from github.  The vars file contains the variables needed for the Geerlingguy.mysql role we'll be running later. But below is an example of what that file should look like, I'll go into more detail further on.

```
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

```
  tasks:
# Checks for server updates and runs them
    - name: Updating apt repo and cache on all servers
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: Updating packages
      apt: upgrade=dist force_apt_get=yes
```

This is the start of our Ansible playbooks tasks section, as you can tell by the 'tasks:' directive and underneath that the tasks are listed out in order that they should run. The first of these two tasks checks for updates, the 'cache_valid_time' directive says that if the last check for updates occurred in the last 3600 seconds (60 minutes) then do not pull updates from the web/repos. The second task here simply installs the updates found by the previous task.

```
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

```
# Uses the pre-created role by GeerlingGuy to install mysql
# The role is installed by Ansible in the '/etc/ansible/galaxy.roles/' folder
# Variables needed for this role are defined in '/etc/ansible/playbooks/secrets/mysql_vars.yml'
    - name: Using geerlingguy.mysql role to install and configure mysql
      include_role:
        name: geerlingguy.mysql
```

After the restart the next Task is to install MySQL. I had quite a bit of trouble getting the MySQL installation to work correctly via Ansible, and eventually settled on using an Ansible Role for MySQL installations created by [Jeff Geerling](https://github.com/geerlingguy) who has written many Ansible Roles for public use. The specific role I am using here is [Ansible-role-mysql](https://github.com/geerlingguy/ansible-role-mysql). You will need to have downloaded and installed this role prior to running this playbook, along with creating the necessary variable file created at the start of this tutorial. The MySQL role has quite a few tasks to go through, so just let it do it's thing and once completed the playbook will move on the next tasks.

```
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




---
title: Ansible Notes
weight: 10
menu:
  notes:
    name: Ansible Notes
    identifier: notes-ansible-ansible_notes
    parent: notes-ansible
    weight: 10
---

{{< note title="WordPress Backups" >}}
Backup WordPress servers.
Assumes WordPress is installed at /var/www/wordpress
```

- hosts: all
  become: true
  vars_files:
    - $PATH_TO_VARS_FILE

  tasks:
    - name: Update repositories
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: Check if server is a WordPress host
      stat:
        path: /var/www/wordpress
      register: wp_check

    - name: Install prerequisites python3-pip
      apt:
        name: python3-pip
        state: present
      when: wp_check.stat.exists

    - name: Install prerequisites boto3
      pip:
        name:
          - boto3
          - botocore

    - name: Create temporary folder for backups on source
      file:
        path: "/tmp/{{ ansible_host }}"
        state: directory
        mode: '0755'
      tags: [ backup ]

    - name: Copy WordPress wp-content folder to backup folder
      copy:
        src: /var/www/wordpress/wp-content
        dest: "/tmp/{{ ansible_host }}"
        mode: '0755'
        remote_src: yes
      tags: [ backup ]

    - name: Do a MySQL dump to copy the database
      mysql_db:
        name: wordpress
        state: dump
        login_user: root
        login_password: "{{ mysql_root_pw }}"
        target: "/tmp/{{ ansible_host }}/backup.sql"

    - name: Compress backup directory
      archive:
        path: "/tmp/{{ ansible_host }}"
        dest: "/tmp/{{ ansible_host }}.{{ ansible_date_time.date }}.tar.gz"

    - name: Copy backup file to storage
      aws_s3:
        aws_access_key: "{{ do_access_key }}"
        aws_secret_key: "{{ do_secret_key }}"
        region: sfo3
        s3_url: "$DO_SPACES_URL_GOES_HERE"
        bucket: "backups"
        src: "/tmp/{{ ansible_host }}.{{ ansible_date_time.date }}.tar.gz"
        object: "{{ ansible_date_time.date }}/{{ ansible_host }}.tar.gz"
        mode: put

    - name: Cleanup temporary files on source
      shell: rm -rf /tmp/{{ ansible_host }} ; rm -rf /tmp/{{ ansible_host }}.*
      args:
        warn: false

```
{{< /note >}}

---
title: "Automating WordPress Backups with Ansible"
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
draft: true
---

Creating backups for WordPress websites using Ansible is a pretty simple process. This article will detail my process for backing up my WordPress websites with Ansible and copying the backups to a DigitalOcean Spaces storage server. Putting the backups in a DO Space allows them to be browsed through the DigitalOcean interface and from there you can download copies of the backups directly or share download links with partners if you ever need to share a WordPress site with a colleague or contractor.

## Prerequisites

To complete this tutorial you will need:

- A server with a WordPress website that will be the target of our backups

- Ansible installed on either your local machine or another server with access to the WordPress target

- A DigitalOcean account with a Space created
    - And an associated [Spaces access key](https://www.digitalocean.com/community/tutorials/how-to-create-a-digitalocean-space-and-api-key#creating-an-access-key)

## Creating the Playbook

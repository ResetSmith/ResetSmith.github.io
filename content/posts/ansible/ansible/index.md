---
title: "Ansible: Getting started"
date: 2022-10-03
hero: post.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Why Ansible
    identifier: ansible-ansible
    parent: ansible
    weight: 10
draft: False
---

Ansible is an open-source tool for automating infrastructure and application deployment. I came across Ansible repeatedly while searching for a better way to manage my numerous bash scripts that I had been using for automating my deployments. Ansible allows for 'infrastructure as code'(IAC) management techniques which in turn make systems more standardized and easier to manage.

## Why Ansible

There are a number of IAC tools in circulation today, and I know I was overwhelmed with options when initially trying to select one to use. Chef, Terraform, SaltStack, and Puppet are all competitors to Ansible that are all apparently widely used and seem to have decent documentation available (as well as all being open source). Ultimately I ended up with Ansible as somewhat of a dartboard toss, more or less closing my eyes, picking a IAC tool and then just going for it.

I have been happy so far with my choice, Ansible has been relatively easy and painless to get started with. First I greatly appreciate that I only need to install Ansible on a single host and nothing extraneous needs to be installed on my remote servers. Ansible is an 'agent-less' tool, so there is nothing to be installed on remote servers. Ansible simply connects over SSH through your existing user account, so right off the bat it's ready to work.

Originally I was searching for a method for running commands simultaneously on multiple remote servers, which Ansible does. But when I figured out how easy it was to string commands together to make playbooks it became immediately apparent how this was going to change my deployment method. In Ansible you combine together lists of 'tasks' (commands), or 'roles' (specialized tasks) into 'playbooks'. It's even modular enough that playbooks can be made of Tasks, Roles, or other Playbooks or even a combination of all three. This allows you to automate infrastructure from provisioning to deployment, and when done correctly helps enforce idempotence in a way that is very difficult to recreate with scripting alone.

## Getting started

The [Ansible Docs Website](https://docs.ansible.com/ansible/latest/getting_started/index.html) has a great 'Getting started with Ansible' section, so I won't bother recreating the wheel. Rather, this section is a quick overview of what you will need to get started with Ansible if you want to follow along with my Playbooks I will be covering in this blog. To get started using Ansible to manage WordPress you will need:

- An Ansible Host 
    - This can be a computer or a Server/VM
    - With the latest version of [Ansible installed](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installation-guide)
    - With a user account with SSH access to the target devices
        - For my purposes I have an 'ansible' user account that has a [passphrase-less SSH key](https://www.redhat.com/sysadmin/passwordless-ssh) for a matching user account on the target servers
    - An Ansible ['Inventory File'](https://docs.ansible.com/ansible/latest/getting_started/get_started_inventory.html) that lists the potential targets for your commands

- A target Server or VM
    - With a user account with the matching public key from your Ansible Host
    - Is listed in the Ansible inventory file


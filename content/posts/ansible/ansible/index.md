---
title: "Ansible: Why"
date: 2022-04-13
hero: 504_Error_Gateway_Timeout-rafiki.png
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Why Ansible
    identifier: ansible-ansible
    parent: ansible
    weight: 10
draft: true
---

Ansible is an open-source tool for automating infrastructure and application deployment. I came across Ansible repeatedly while searching for a better way to manage my numerous bash scripts that I had been using for automating my deployments. Ansible allows for 'infrastructure as code'(IAC) management techniques which in turn make systems more standardized and easier to manage.

## Why Ansible

There are a number of IAC tools in circulation today, and I know I was overwhelmed with options when initially trying to select one to use. Chef, Terraform, SaltStack, and Puppet are all competitors to Ansible that are all apparently widely used and seem to have decent documentation available (as well as all being open source). Ultimately I ended up with Ansible as somewhat of a dartboard toss, more or less closing my eyes, picking a IAC tool and then just going for it.

I have been happy so far with my choice, Ansible has been relatively easy and painless to get started with. First I greatly appreciate that I only need to install Ansible on a single host and nothing extraneous needs to be installed on my remote servers. Ansible is an 'agentless' tool, so there is nothing to be installed on remote servers. Ansible simply connects over SSH through your existing user account, so right off the bat it's ready to work.

Originally I was searching for a method for running commands simultaneously on multiple remote servers, which Ansible does. But when I figured out how easy it was to string commands together to make playbooks it became immediately apparent how this was going to change my deployment method. In Ansible you combine together lists of 'tasks' (commands), or 'roles' (specialized tasks) into 'playbooks'. It's even modular enough that playbooks can be made of Tasks, Roles, or other Playbooks or even a combination of all three. When done correctly this allows you to automate infrastructure from provisioning to deployment, and when done correctly helps enforce idempotence in a way that is very difficult to recreate with scripting alone.

## Ansible Caveats

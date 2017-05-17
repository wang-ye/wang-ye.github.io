---
layout: post
title:  "Flask-base Virtual Environment with Ansible"
date:   2017-05-15 21:35:03 -0800
---

[flask-base](https://github.com/hack4impact/flask-base) is a flask application template with extensive boilerplate code. I found it annoying to configure the local environment by following tedious instructions. Basically, you need to manually install the right Python version, and various services such as postgres, redis and ruby. To simplify the process, I thus spent some time building a virtual development environment using with Vagrant, Virtualbox and Ansible. This post talks about my configuration scripts.

## Prerequisite

To build this virtual environment, you first need to install 
[Virtualbox](https://www.virtualbox.org/wiki/Downloads), [Vagrant](https://www.vagrantup.com/downloads.html) and Ansible. 
Ansible is a Python package and can be installed with:

```shell
pip install ansible
```

Simply put, Vagrant serves as the commander (guest machine on/off/status, ssh into guest machine); Virtualbox is the actual container (bare guest OS holder), while Ansible is the provisioner (installing necessary softwares and start services). A simple hello-world example with the above tools can be found [here](https://gist.github.com/drgarcia1986/386a7546fa9c8ba47f39).

## Detailed Config

The configration mainly contains two pieces - a Vagrantfile and Ansible YML configuration. The Vagrantfile is as follows:

```ruby
# Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "ubuntu/xenial64"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  config.vm.network "forwarded_port", guest: 5000, host: 7000

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder ".", "/home/ubuntu/flask-base"

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision 'ansible' do |ansible|
    ansible.playbook = 'site.yml'
    ansible.verbose = 'v'
    ansible.extra_vars = { ansible_user: 'vagrant', ansible_python_interpreter: '/usr/bin/python3' }
  end 
end
```

Vagrant takes advantage of synced folder to easily share code and data between guest and host machines. It uses Ansible to provision the virtual machines. The detailed provision instructions are listed in *site.yml*. It is worth noting that [Ansible enables Python3](https://docs.ansible.com/ansible/python_3_support.html) with

```
ansible_python_interpreter: '/usr/bin/python3'
```

Now, let's jump to Ansible configuration details:

```yml
- name: Configure application
  hosts: all
  become: true
  become_method: sudo
  vars:
      repository_path: /home/ubuntu/flask-base

  tasks:
    - name: Install packages
      apt: update_cache=yes name={{ item }} state=present
      with_items:
        - git
        - libpq-dev
        - python-dev
        - python3-pip
        - redis-server
        - python3-venv
        - ruby

    - name: Install sass
      gem: name={{ item }} state=present
      with_items:
        - sass

    - name: Install requirements
      pip: requirements='{{ repository_path }}/requirements.txt'

    - name: Run App
      shell: cd /; nohup python3 /home/ubuntu/flask-base/manage.py runserver --host 0.0.0.0 > /home/ubuntu/flask-base/web_log 2>&1 &
```

It does the following:

1. Run the tasks with sudo permission.
2. Install necessary packages, such as python, redis, ruby, sass and postgres db.
3. Install requirements with pip.
4. Start the App in dev mode.

**It is worth noting that to access guest machine app from host, you have to set the host to '0.0.0.0'. I spent hours debugging this issue ...**

The full script can be found [here](https://github.com/wang-ye/code/tree/master/flask-base-ansible). The flask-base commit hash I worked on is [dcf02d8](https://github.com/hack4impact/flask-base/commit/dcf02d809ab76f47a78dc99c35c9c84bc22cebf6).

## How To Use
You can first download the [full scripts](https://github.com/wang-ye/code/tree/master/flask-base-ansible). To start guest machine, just run

```shell
vagrant up --provision
```

Once guest machine is up and running, you can access the app in *localhost:7000*.

If you want to enter the guest machine to fine-tune the commands, logging into the machine is as simple as
 
```shell
vagrant ssh
```

## Summary
This posts discusses the virtual environment configuration for flask-base. It provides a python3-compliant, easy-to-use virtual environment. Some future enhancements include:

1. We can have virtualenv set for the guest machine. This way we can precisely specify the desired Python version 
2. Enable it to work in production environment.
3. Make the script more modular with roles.

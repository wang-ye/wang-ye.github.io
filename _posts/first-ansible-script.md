I was playing with [flask-base](https://github.com/hack4impact/flask-base) recently, and found it annoying to configure the local environment.

Ideally, I want to run dev-mode flask-base in a container. I played with Vagrant + Virtualbox + Ansible and finally get a working version. This is a personal note on the configurations.

The commit hash is dcf02d809ab76f47a78dc99c35c9c84bc22cebf6

The full script can be found at : https://github.com/wang-ye/code/tree/master/flask-base-ansible

Simply put, Vagrant serves as the commander (guest machine on/off/status, ssh into guest machine), Virtualbox is the actual container (bare guest OS holder), while Ansible is the provisioner (install necessary softwares and start services).

To play with it,
First, you will need to install necessary softwares.
Virtualbox,
Ubuntu:
https://www.virtualbox.org/wiki/Downloads

sudo apt-get install virtualbox
Mac: 

For Vagrant,
Download from here: https://www.vagrantup.com/downloads.html

For Ansible,
it is a Python package, you can use:
pip install ansible

```ruby
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

Note that, flask-base uses Python3. To enable Python3, we use:
ansible_python_interpreter: '/usr/bin/python3'

Another thing worth noting is we applied synced folder, which can easily share the source and data between guest and host.

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

What is does:
1. Run the tasks with sudo permission.
2. Install necessary packages, such as python, redis, ruby, sass and postgres db.
3. Install requirements with pip.
4. Start the App in dev mode.

It is worth noting that to access guest machine app from host, you have to set the host to '0.0.0.0'. I spent hours on it.

More TODOs:
1. We can have virtualenv set for the guest machine. This way we can precisely specify the desired Python version 
2. Enable it to work in production environment.
3. Make the script more modular with roles.

https://gist.github.com/drgarcia1986/386a7546fa9c8ba47f39

---
layout: post
title:  "Using Ansible to provision AWS EC2 instances with Docker"
categories: hacking
tags:
- Ansible
- Docker
- AWS EC2
- S3
---

## Background

In AWS’s China (Beijing) region, both [EC2 Container Service](https://aws.amazon.com/ecs/) and [EC2 Container Registry](https://aws.amazon.com/ecr/) are not supported yet (as of July 2016). However, since I personally believe that docker is going to be huge, esp. with the imminent release of 1.12 which introduces an [autonomous swarm cluster mode](https://docs.docker.com/swarm/), I decided to plan our infrastructure with docker installed on EC2 instances anyway.

The obvious way to automate the process is to use [ansible](https://www.ansible.com/), which has extensive support for EC2 as [dynamic inventory](http://docs.ansible.com/intro_dynamic_inventory.html) and [docker modules](http://docs.ansible.com/docker_container_module.html). I like ansible’s simplicity, given that it has the minimal requirement of only ssh daemons on target machines (well, in fact you’ll also need Python and docker-py module installed if you want to use ansible’s docker module).

In this post I’ll share how I setup my ansible repository to create and provision EC2 instances with docker. Note that ansible is a config-heavy tool, and since that YMMV, please don’t simply copy and paste the config files.

## Setup

I followed the [suggested directory layout from ansible](http://docs.ansible.com/ansible/playbooks_best_practices.html), and created a `site.yml` file, a `group_vars` folder, a `roles`  folder for putting roles (so that different groups of machines can share common tasks). I also downloaded both [ec2.ini](https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini) and [ec2.py](https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.py) to make use of EC2 based dynamic inventory and put them under the folder `inventory` .

So the final directory structure looks like:

```
.
├── README.md
├── ansible.cfg
├── backends.yml
├── group_vars
│   ├── all.yml
│   ├── tag_class_backends.yml
│   └── tag_container_docker.yml
├── inventory
│   ├── ec2.ini
│   └── ec2.py
├── provisions.yml
├── requirements.txt
├── roles
│   ├── docker
│   │   └── tasks
│   │       └── main.yml
│   ├── launch
│   │   └── tasks
│   │       └── main.yml
│   └── dataserver
│       └── tasks
│           └── main.yml
└── site.yml

9 directories, 14 files
```

It is a good practice to put `ansible.cfg` in the root folder, to configure how ansible behaves:

```
[defaults]
remote_user = ec2-user
host_key_checking = False
inventory = ./inventory/ec2.py

[ssh_connection]
# see http://docs.ansible.com/ansible/intro_configuration.html#control-path
control_path = %(directory)s/%%h-%%r
```

Since I am using Amazon’s default Linux distribution, the default username is `ec2-user` , and the `inventory` value tells ansible to use the EC2 dynamic inventory. The `control_path` is to make sure the limit of unix socket name is not reached.

The `requirements.txt` is to setup a python virtualenv so that my teammates can reproduce the exact environment to use this ansible playbook folder. Basically you should install `boto` to connect to AWS and `ansible` itself.

Apart from `site.yml` I also created `provisions.yml` and `backends.yml` , the first playbook is for creating new instances on EC2, and the second one is for creating our backends. You can also create others like `webserver.yml` for example. I’ll focus on `provisions.yml` for this post.

## Creating instances

The content of top-level `provisions.yml` is rather simple since most of the meat is in `roles` for possible reuse:

```yaml
---
# provisions.yml
# this brings up machines and provision them with
# docker installs.

- hosts: localhost
  connection: local
  gather_facts: no
  roles:
    - role: launch
      name: "{{ name }}"
      class: "{{ class }}"
      iam_profile: DockerHost_EC2_Role

- hosts: "tag_class_{{ class }}"
  become: True
  become_user: root
  become_method: sudo
  roles:
    - docker
```

Note that there are two plays: the first one targets localhost because itself will create new EC2 instances, and the second one targets group `tag_class_{{ class }}` which needs more explanations later.

Let’s take a look at the launch task which is used by first play:

```yaml
---
# roles/launch/tasks/main.yml
- name: Launch new instance
  ec2:
    region: "{{ region }}"
    keypair: "{{ keypair }}"
    zone: "{{ zone }}"
    group: "{{ security_groups }}"
    image: "{{ image }}"
    instance_profile_name: "{{ iam_profile }}"
    instance_type: "{{ instance_type }}"
    instance_tags:
      Name: "{{ name }}" # capitalize because it's AWS convention
      class: "{{ class }}"
      container: docker
    volumes: "{{ volumes }}"
    wait: yes
  register: ec2

- name: Add new instances to host group
  add_host:
    name: "{{ item.public_ip }}"
    groups: "tag_class_{{ class }}"
    ec2_id: "{{ item.id }}"
  with_items: "{{ ec2.instances }}"

- name: Wait for instance to boot
  wait_for:
    host: "{{ item.public_ip }}"
    port: 22
    delay: 30
    timeout: 300
    state: started
  with_items: "{{ ec2.instances }}"
```

it uses ansible’s ec2 module to create instances using a set of variables. Basically you can tweak a lot of things here:

- region, availability zones, subnets, etc.
- keypair, security group, instance profile, etc.
- instance type, disk volumes, etc.
- instance tags, which includes one or more user defined tags

These variables can have a global default setup, and you can put it in `group_vars/all.yml` :

```yaml
---
# group_vars/all.yml

region: cn-north-1
zone: cn-north-1a
keypair: mykey
security_groups: ['dev-server']
instance_type: t2.micro # cheapest as a failsafe, overriden by group

image: ami-8e6aa0e3 # Amazon Linux AMI 2016.03.3 (HVM), SSD Volume Type
group_name: test
iam_profile: "noaccess" # no access as a failsafe, overriden by group

# this uses douban's pip source, because official repo is too slow in China
pip_extra_args: '-i https://pypi.doubanio.com/simple'

volumes: []
```

Note that I applied a `class` tag above, and then in second step, the newly created instances are added to the `tag_class` group respectively. This makes it easier to identify what classes (or types/kinds) of instances you have and apply different provisioning logics to them separately. This tag is also used in the second play of `provisions.yml` . More detailed explanations can be found on ansible’s docs about dynamic inventory.

After the instances are created and added to groups, the task will wait for their SSH to be available in order to proceed.

## Provisioning docker

The second step is to install and start docker daemon on each machine:

```yaml
---
# roles/docker/tasks/main.yml
- name: Install python setuptools and docker
  yum:
    name: "{{ item }}"
    update_cache: yes
  with_items:
    - python-setuptools
    - docker

- name: Start docker
  service:
    name: docker
    state: running

- name: Install pypi
  easy_install: name=pip

- name: Install docker-py to latest
  pip:
    name: "{{ item }}"
    state: latest
    extra_args: "{{ pip_extra_args }}"
  with_items:
    - docker-py
```

Amazon’s Linux AMI comes with yum (and a fast intranet source list). You might want to use `apt` if you are using ubuntu or debian. Docker officially provided https://get.docker.com script for installation like `curl https://get.docker.com | sh` , but since you already know what’s inside there, you might just use  `yum` .

The script also installs `easy_install` , `pip`  and then `docker-py` in order, for using ansible’s docker container module.

With this done, you are all set! Go and run `provisions.yml` playbook and you’ll get an instance of EC2 running with docker installed on it.


## Running docker containers

To showcase how you can then run docker containers, see below:

```yaml
---
# roles/dataserver/tasks/main.yml

- name: login to registry
  docker_login:
    username: '{{ docker_registry_user }}'
    password: '{{ docker_registry_pass }}'
    email: '{{ docker_registry_email }}'

- name: postgres data container
  docker_container:
    name: pg_data
    image: busybox
    state: present
    volumes:
      - /var/lib/postgresql

- name: start the local postgres for demo
  docker_container:
    name: postgres
    image: postgres
    state: started
    volumes_from:
      - pg_data
    log_driver: awslogs
    log_options:
      awslogs-region: "{{ region }}"
      awslogs-group: "{{ awslogs_group }}"
      awslogs-stream: postgres
    env:
      POSTGRES_USER: "{{ PG_USER }}"
      POSTGRES_PASSWORD: "{{ PG_PASSWORD }}"
      POSTGRES_DB: "{{ PG_DATABASE }}"
    restart_retries: 10
    restart_policy: always

# more goes on
```

You can pretty much use ansible to do many cool things with docker, like login to dockerhub and pull private images, telling docker containers to log to a specific CloudWatch log stream, and create docker containers to talk to each others within different network interfaces. For more detailed information you can see [the docs](http://docs.ansible.com/ansible/docker_container_module.html). Once you have these credentials stored as group_var files, you can use [ansible-vault](http://docs.ansible.com/ansible/playbooks_vault.html) to encrypt them so that you don’t have to store passwords in cleartext in git.


## One more thing about docker storage driver

If you came from ubuntu like I did, it might never occur to you that docker used to only run on [aufs](https://docs.docker.com/engine/userguide/storagedriver/aufs-driver/) (like wtf is that). Since Amazon’s Linux is CentOS based, if you run `docker info` it might tell you that it is based on `devicemapper` and it uses local loopback which is not ideal for production. Turns out there is a lot going on in the docker storage driver discussion, like [this](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/) and [this](http://www.projectatomic.io/blog/2015/06/notes-on-fedora-centos-and-docker-storage-drivers/).

Long story short, if possible, try to use a better config for docker storage driver if you plan to use it for non-trivial tasks.

Luckily there is already some handy script that does the whole `lvm` managing for you. My plan in this is to create and mount another EBS disk for the EC2 instance for docker’s use, and use the script to reconfigure docker to use `direct-lvm` mode.

I modified `group_vars/all.yml` in add a disk volume to each instance:

```yaml
volumes:
  # this is for docker storage
  - device_name: /dev/sdb
    device_type: gp2
    volume_size: 8
    delete_on_termination: true

# name of the docker volume group
docker_volume_group: docker_vol
```

and then in `roles/docker/tasks/main.yml` to include these additional steps after installing docker and before starting the docker daemon:

```yaml
- name: Create the volume group for docker volume
  shell: vgcreate "{{ docker_volume_group }}" "{{ volumes[0].device_name }}"

- name: Install docker-storage-setup
  yum: name=docker-storage-setup.noarch

- name: Configure docker-storage-setup
  lineinfile:
    dest: /etc/sysconfig/docker-storage-setup
    state: present
    create: yes
    line: "{{ item }}"
  with_items:
    - 'VG={{ docker_volume_group }}'
    - 'DEV={{ volumes[0].device_name }}'

- name: Remove the old thin pool logical volume
  shell: rm -rf /var/lib/docker

- name: Run docker-storage-setup
  shell: docker-storage-setup
```

The script will create and update the logical volume on the block device and write configuration to `/etc/sysconfig/docker-storage`  so that docker daemon can pick up the new config.

Now that when you run `docker info` there will be no warning.

## What’s more

I plan to try out ansible + docker swarm after the official release of docker 1.12, and see what changes it can bring. Stay tuned.

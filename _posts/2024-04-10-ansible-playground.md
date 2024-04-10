---
layout: post
title: Ansible Playground
date: 2024-04-10 12:00:00 +0300
categories: [blog, Ansible]
---

# A Playground for Ansible
*Create an ansible playground using docker containers*

## Problem and scope

So I recently learned a little about Ansible and started playing around with it, and wanted to come up with a simple lab environment I can work my new muscle in.

There are labs that you can setup using cloud systems, like spawning up a few small AWS instances and using systems manager to run playbooks, but seemed like too much.

## A possible solution

I decided to spring up a quick docker file to build a container that could act as Ansible-child machines (Managed nodes) that I could do all the tricks on.  

I will try to layout the process I went to and why certain things are in place the way they are. Here is the GitHub Repository for the project in case you want to checkout.

**Github Repo** :- https://github.com/dhwanil-thakkar/ansible-playground

## The Process 

So we wamt to create docker container that can act as ansible-child machines (they are officially called managed nodes but I like to call the ansible_child machines, I do not know why :-P).

For a machine to take commands from ansible , it requires nothing but an ssh connection setup, that is the beauty of ansible. In real world scenario a script like this does not make sense as you already probably have ssh connections setup always, so this only makes sense in a developing or testing environments for developers to quickly sprung up a few docker containers to test out the code locally.

You would also need a few more things setup apart from just installing the SSH server and flipping the switch, we need to have a user in place for ansible to run playbooks as. you need to setup a ssh key-pair that ansible can authenticate using and probably a configuration file for setting up the SSH service (this provides a flexibility to make changes easily before the image is built to accomodate for needs).

### Code

So we start with an empty container , you can basically pick anything , I have chosen Ubuntu here to be as close as to most production workloads.

- Now lets update the apt cache and install a few packages onto that basic image, we will install the *open-ssh server* (the ssh server) *python3-apt* (this controls the debian package manager and is used by ansible apt module.), *sudo*, *vim* (just because :P). That would look something like this 

```
FROM ubuntu:latest
RUN apt  update && apt install openssh-server python3-apt vim sudo -y
```
<br>

- Now we need to create a user, give it sudo permission for ansible to work on, and give it a password (this is optional, but can help with troubleshooting).
- Then let's take a copy of the sshd_config file and put it as the configuration file, you can edit this file before building the container for customization if needed.
  
```
RUN useradd -rm -d /home/john -s /bin/bash -g root -G sudo -u 1000 john
RUN echo "john:doe" | chpasswd
COPY sshd_config /etc/ssh/sshd_config
```
<br>

- So now we have a user ready as well as the SSH server installed, the next part is to setup the authentication part. So lets create a RSA key-pair using ssh-keygen ( [How to create a SSH key-pair](https://man7.org/linux/man-pages/man1/ssh-keygen.1.html) ).
- Next we need to copy the public key to the user's ssh folder , under the file "authorizedkeys", we also need correct ownership and specific permissions on those folders. A owner read-write on the file (600) and a owner read-write-execute on the folder (700).

```
RUN mkdir /home/john/.ssh
COPY id_rsa.pub /home/john/.ssh/authorized_keys
RUN chmod 700 /home/john/.ssh
RUN chown john:root /home/john/.ssh
RUN chmod 600 /home/john/.ssh/authorized_keys
RUN chown john:root /home/john/.ssh/authorized_keys
```

- Now all that is left to do is expose the SSH server port and start the service.

```
RUN service ssh start
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

That is the Docker file in all essence, of course it is open to be edited and be highly customizable , this is just a starting point.

### Usage

You can find the code and instruction on my GitHub repository [here](https://github.com/dhwanil-thakkar/ansible-playground) but I will briefly touch on how to make this Docker file work to create a Lab environment.

So once we have the Docker file, SSH config file, and the public key file  ready , we can build the image using the command below. 

```bash
sudo docker build -t ansible_child .  
```

Once the image is built, let's create a container from the image using the command below.

```bash
sudo docker run -d --name="child2" ansible_child
```

You can obviously can run multiple copies of the container by simply running the above command multiple times and changing the *name* attribute.

Next You need to get the IP Address of the managed nodes you deployed and list them in an inventory file.
The last part is to create a ansible config file of the exact name "ansible.cfg" and tell which inventory file to use and what private key file should be used to authenticate.

### Testing

Once you have launched the containers, you can run a simple ansible ad-hoc command to test the connection using the ping module by running the command as follows. The second command uses the inventory file and runs the check on machines specified in inventory.

```bash
ansible <ip-address> -m ping
```
```bash
ansible all -m ping
```
---
layout: post
title: "Vagrant and Docker, repeatable development"
date: 2015-10-07 20:00:00
categories: vagrant docker development
---

## Repeatable Development Environments

Wanted to talk briefly today about repeatable development environments.  Repeatability
is a very good trait for a development environment to have.  I have been plagued for 
years and years of not giving myself a way to repeatably build a development 
environment.  We get lazy right?  You get used to just installing everything you need
for development globally on your development machine, until it gets to a point where 
you can't sanely reproduce what your production environment looks like, and you end up
going with the nuclear option of destroying everything and starting over... 

## Enter Vagrant

Vagrant is an abstraction layer on Virtual Machine Engines that allows you to build 
a complete development environment in the virtual machine of your choosing.  Vagrant 
is very much focused on automation and allows some very cool and exciting mechanisms 
to create repeatable builds of virtual machines with very minimal effort.  In-fact, to 
get started, you real just need to do the following:

1. download and install a virtual machine engine such as VirtualBox or VMWare
2. download and [install Vagrant][install-vagrant]
3. create a Vagrantfile, explained next

## Vagrant File

Vagrant needs instructions on what to do to make you a virtual machine, so below is a snipit of a Vagrantfile:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.hostname = "sandbox"
  config.vm.box = "debian/jessie64"

  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get install apt-transport-https
    sudo apt-get update
  SHELL

  config.vm.provision "docker" do |d|
      d.run "debian:jessie",
          name: "bash_container",
          cmd: "bash",
          daemonize: true
  end
#...
{% endhighlight %}

You can see in this Vagrantfile we are setting some particulars about the virtual 
machine that is being created, the name and what base operating system image we want
to run on this virtual machine.

Next we actually start provisioning our virtual machine.  This is the good stuff. 
Vagrant comes with a ton of provisioner such as chef, salt, ansible, shell, docker, 
and many more.  This allows someone to very easily set up a repeatable environment.

The docker provisioning above actually installs docker on the virtual machine, starts
the docker service and then performs the following:

{% highlight bash %}
	docker run -d -n bash_container debian:jessie /bin/bash
{% endhighlight %}

This command starts up a bash session in a container in daemon mode.

Pretty cool right?

# There is more?

Vagrant also has the concept of volume mounting, allowing the vm to access a part of 
the host OS's file system.  What this allows me to do is mount my code bases inside of 
the virtual machine.  This way I can have a virtual machine that contains all of my
production environment installed packages and moreover, have a repeatable mechanism
by which I can reproduce that environment.

What is even cooler is, you can take it a step further and volume mount the file system
of the vagrant virtual machine host inside of docker containers you spin up... 

I wonder what happens if you sshfs mount back to the physical host's file system ;)

Anyhow, this is truly exciting for me to be able to provision a vm with all the development and build tools I need all the while allowing it to be repeatable and 
portable.

[install-vagrant]: https://www.vagrantup.com/downloads.html 
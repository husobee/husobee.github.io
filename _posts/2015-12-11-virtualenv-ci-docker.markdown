---
layout: post
title: "Building Python With Docker in CI"
date: 2015-12-11 08:00:00
categories: python docker ci
---

# Build System

A while back I started using, and looking deeply into [drone.io][drone] as a 
continuous integration system.  The basic tldr; of drone.io is a build system
that uses docker instances to build your code.  At work we are running a server
with drone.io running on it that performs builds of our go code.

Anyhow, [very early on][docker-drone] we felt that drone could be enhanced, by 
allowing the repositories to specify volume mounts from the drone build system
file system into our drone build container.  This allows us to do wacky stuff
like volume mount the docker socket to the running build container and use 
the host's docker daemon to perform `docker build` commands from within the
build container without having to run docker within a docker container.

This proved so useful that [drone adopted this idea][drone-build] into their 
product.

# ... Segue to Testing

[As I have mentioned before][behavior] we write numerous behavior tests. These
tests work great for our QA department, and ease the communication gap between
developers and QA and Product.  I really like this style of testing due to the
clearer communication.  So a little while back we had merely a repository per
micro-service of python based behavior tests, which were run when someone felt 
the need to run them.  Not very Continuous.

We decided that pushes to these testing repos should be treated just like our
code, and we should build *testing artifacts* which could be run within our 
automated pipeline.  As these testing repos are not golang, we had to figure out
how to build and install dependencies in our drone scheme, not to mention
figure out how to pull requirements from private repositories with drone.

# Building our Python Code

Lets start with a drone build script:

{% highlight yaml %}
build:
  image: private.registry/library/python-build
  volumes: 
    - /var/run/docker.sock:/var/run/docker.sock
  commands:
    - docker build -t private.registry/library/servicea-tests .
{% endhighlight %}

With a Dockerfile that looks like this:

{% highlight text %}
FROM private.registry/library/debian:jessie

RUN apt-get update \
	&& apt-get install -y python python-virtualenv python-pip

RUN mkdir /app
ADD . /app

RUN pip install -r requirements.txt

CMD["lettuce"]

{% endhighlight %}

The above is pretty simple for base use cases, where you just want to create a 
docker container that has python, and pip install this repositories repos.  But
what if your `requirements.txt` contains a private repo location such as 
`git+ssh://git@private.repo/my-private-repo@master#egg=my-private-repo`?

Well, drone.io volume mount and python virtualenv to the rescue!

Check out the modified drone build script below:

{% highlight yaml %}
build:
  image: private.registry/library/python-build
  volumes: 
    - /var/run/docker.sock:/var/run/docker.sock
    - /opt/read_only_ssh.key:/read_only_ssh.key
  commands:
    - build.sh
    - docker build -t private.registry/library/servicea-tests .
{% endhighlight %}

Below is the new build.sh script:

{% highlight bash %}
#!/bin/bash

eval ssh-agent && ssh-add /read_only_ssh.key
# create a virtual env called .bdd
virtualenv .bdd
# go into said virtualenv
source .bdd/bin/activate
# install all of the requirements in the virtual env
pip install -r requirements.txt
ssh-agent -k
{% endhighlight %}

And our new Dockerfile looks like this:

{% highlight text %}
FROM private.registry/library/debian:jessie

RUN apt-get update \
	&& apt-get install -y python python-virtualenv python-pip

RUN mkdir /app
ADD . /app

#ugly hack, sorry
mkdir -p /var/lib/drone/cache/ /var/lib/drone/cache/
ln -s /app/.bdd /var/lib/drone/cache/.bdd 

CMD["VIRTUAL_ENV=/app/.bdd PATH=$VIRTUAL_ENV/bin:$PATH lettuce"]

{% endhighlight %}

The result of this is a python docker image that runs our behavior 
tests when this docker container is run.  The build.sh script uses
the volume mounted ssh key to pip install any private repositories
we need (like a common utilities library).

[drone]: https://drone.io
[drone-build]: https://github.com/drone/drone/blob/master/docs/build/build.md
[docker-drone]: https://github.com/drone/drone/pull/773#issuecomment-99604130
[behavior]: http://husobee.github.io/automate/behavior/testing/2015/06/13/automate-behaviors.html

---
layout: post
title: "docker-compose container orchestration"
date: 2016-02-04 08:00:00
categories: docker compose
---

docker-compose is a python project, run by the people at docker, that aims to 
give docker users a cleaner project/service/container orchestration layer.  

## overview of compose
The main gist is, with a defined project `.yaml` file you are able to describe the
service state you want to be running in docker.  Brief example of a project file:

{% highlight yaml %}
service_a:
  image: ubuntu:latest
  volumes:
    - /tmp/configs/service_a.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "running"; sleep 1; done
service_b:
  image: ubuntu:latest
  volumes:
    - /tmp/configs/service_b.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "running"; sleep 1; done
{% endhighlight %}

In our example we have two services defined, that volume mount configuration
`.json` files.  With this file named `service-a-b.yml` we can now run the 
following command to bring up our services:

    docker-compose -f service-a-b.yml up -d

This will create two containers, `$DIRNAME_service_a_1` and `$DIRNAME_service_b_1`
where $DIRNAME is the name of your "project" directory.  Great right, docker 
containers get spun up quickly.

To bring your services down, stopping and removing your containers, you can run
the following command:

    docker-compose -f service-a-b.yml down

## cool features of compose

Just the basics of compose is really cool in of itself, the ability to create a
service manifest for a project which coordinates many docker containers.  But, 
compose offers us some goodies too:

1.) [Stacking Config Files][#stacking config files]
2.) [Smart Container State][#smart container state]
3.) [Wrappers around common docker commands][#wrapping docker]


### stacking config files

If you noticed in previous commands I used the `-f` flag to specify the compose
config file I wanted to use.  Compose has the concept of `ConfigExtender` where 
compose is smart enough to know the order of the `-f` flags passed in.  Below 
are two config files `service_a_b.yaml` and `service_b_c.yaml` in that order:

{% highlight yaml %}
service_a:
  image: ubuntu:latest
  volumes:
    - /tmp/configs/service_a.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "running"; sleep 1; done
service_b:
  image: ubuntu:latest
  volumes:
    - /tmp/configs/service_b.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "running"; sleep 1; done
{% endhighlight %}

{% highlight yaml %}
service_b:
  image: ubuntu:latest
  volumes:
    - /tmp/configs/service_b.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "waiting"; sleep 1; done
service_c:
  image: ubuntu:latest
  volumes:
    - /tmp/configs/service_a.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "running"; sleep 1; done
{% endhighlight %}

Running the following command:

    docker-compose -f service_a_b.yaml -f service_b_c.yaml up -d

Will work exactly how you would imagine it would work.  The end state there
are three containers, `$DIRNAME_service_a_1` `$DIRNAME_service_b_1` 
`$DIRNAME_service_c_1`.  This is because the union of the two configuration
files enumerate three services in total.  When looking at the output of 
`$DIRNAME_service_b_1` you should also note that it says `waiting` instead of
`running`.  This is due to how configuration extending works in compose, 
it is basically an "extends" operation on the project configuration dictionary.

By stacking compose configuration files you gain flexibility, if for example in 
a certain run you wished to run your service in [coverage mode][external-coverage]
to get external test coverage of your service.

### smart container state

Compose takes the current service state into account before trying to provision
more containers to accommodate the service.  This is nice as you don't want to 
rebuild or recreate the containers again if there is no state change often.

### wrapping docker

Compose has various nice docker wrapping commands that take the entire project 
into accounts, such as `docker-compose logs` will dump out a clean colorized 
log of the entire project based on the individual container docker logs.

Other wrapping commands can be seen from the `--help` option of compose:

      build              Build or rebuild services
      config             Validate and view the compose file
      create             Create services
      down               Stop and remove containers, networks, images, and volumes
      events             Receive real time events from containers
      help               Get help on a command
      kill               Kill containers
      logs               View output from containers
      pause              Pause services
      port               Print the public port for a port binding
      ps                 List containers
      pull               Pulls service images
      restart            Restart services
      rm                 Remove stopped containers
      run                Run a one-off command
      scale              Set number of containers for a service
      start              Start services
      stop               Stop services
      unpause            Unpause services
      up                 Create and start containers
      version            Show the Docker-Compose version information

## contribution

I am working on [this fork][compose-fork] currently, attempting to get a few new
commands added to the configuration file schema: `pre_create` and `pre_start`. I
have found often I need to perform some preparatory work to the container host, 
such as template population of configuration files, checking state of certain 
exposed ports, etc.  Here is an example of my fork's modifications:

{% highlight yaml %}
service_a:
  image: ubuntu:latest
  pre_create:
    - sed -i "s/{{hostname}}/$(hostname -f)/g" /tmp/configs/service_a.json
  pre_start:
    - bash -c "while true; nc -z 0.0.0.0 3601; if [[ $? -eq 0 ]]; then break; fi; sleep 1; done"
  volumes:
    - /tmp/configs/service_a.json:/config.json:ro
  environment:
    - CONFIG=/config.json
  command: while true; do echo "running"; sleep 1; done
{% endhighlight %}

In the above with `pre_create` you are able to specify something to happen before
the image is created by compose, and `pre_start` will allow you to run something
before the service is started.  In this example, in `pre_create` I am replacing
{{hostname}} with the host's hostname. In `pre_start` I am waiting for mysql to 
be available before running this container.

Seems like a nice bit of extra flexibility.  Will PR soon!

[external-coverage]: http://husobee.github.io/golang/test/coverage/2015/11/17/external-test-coverage.html
[compose-fork]: https://github.com/husobee/compose

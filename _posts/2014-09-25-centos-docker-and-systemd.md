---
layout: post
title: "CentOS, Docker, and Systemd"
description: ""
category: CentOS
tags: [CentOS, Docker, systemd]
---
{% include JB/setup %}

Over the last few weeks, we've been asked about using systemd inside the
CentOS-7 Docker containers for more complex operation. Because systemd offers
a number of rather nice features, I can completely understand why people want
to use it rather than pulling in outside tools like supervisord to recreate
what already exists in centos by default. Unfortunately it's just not that
easy. 

There are a couple major reasons why we don't include systemd by default in
the base Docker image. Dan Walsh covered these pretty completely in a [blog
post](http://developerblog.redhat.com/2014/05/05/running-systemd-within-docker-container/), 
but to recap where we are currently lets hit the highlights

 * systemd requires the `CAP_SYS_ADMIN` capability. This means running docker with `--privileged`. Not good for a base image.
 * systemd requires access to the cgroups filesystem. 
 * systemd has a number of unit files that don't matter in a container, and they cause errors if they're not removed

It's for these reasons that we ship with fakesystemd in the default image. The
fakesystemd package provides dependency resolution and the proper directory
structure so that packages install normally, and individual apps can be run
inside the container by default. The fakesystemd package isn't really an
elegant fix, but it's currently the best we can do in the base images. As soon
as we're able to rip it out and ship a proper systemd package, we will. 

If you're okay with running your container with `--privileged`, then you can
follow the steps below to create your systemd enabled docker image from the
CentOS-7 base image.

### Dockerfile for systemd base image

    FROM centos:centos7
    MAINTAINER "you" <your@email.here>
    ENV container docker
    RUN yum -y swap -- remove fakesystemd -- install systemd systemd-libs
    RUN yum -y update; yum clean all; \
    (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i ==
    systemd-tmpfiles-setup.service ] || rm -f $i; done); \
    rm -f /lib/systemd/system/multi-user.target.wants/*;\
    rm -f /etc/systemd/system/*.wants/*;\
    rm -f /lib/systemd/system/local-fs.target.wants/*; \
    rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
    rm -f /lib/systemd/system/basic.target.wants/*;\
    rm -f /lib/systemd/system/anaconda.target.wants/*;
    VOLUME [ "/sys/fs/cgroup" ]
    CMD ["/usr/sbin/init"]

This Dockerfile swaps out fakesystemd for the real deal, but deletes a bunch
of the unit files we don't need. Building this image gives us a usable base to
start from. 

    docker build --rm -t centos7-systemd . 

### systemd enabled app container

Once this new base image is built, we can move on to building stuff that
actually needs systemd. In this instance we'll use httpd as an example. 

    FROM centos7-systemd
    RUN yum -y install httpd; yum clean all; systemctl enable httpd.service
    EXPOSE 80
    CMD ["/usr/sbin/init"]

We build once again:

    docker build --rm -t centos7-systemd/httpd

To put this all together and run httpd with systemd in Docker, we do this:

    docker run –privileged -ti -v /sys/fs/cgroup:/sys/fs/cgroup:ro -p 80:80 centos7-systemd/httpd

This container is running with systemd in a limited context, but it must
always be run as a privileged container with the cgroups filesystem mounted.


There are plenty of rumors circulating that this will be addressed in the
future, either in system or in a systemd-like subpackage. As soon as we're
able to ship systemd in the base image, we will do so. 



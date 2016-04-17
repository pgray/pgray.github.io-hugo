+++
date = "2016-04-17T16:59:40-04:00"
description = ""
draft = false
keywords = []
title = "cron runner"

+++
## Why?
Cron is awesome and its use is widespread. Sometimes though, one might hit minor hiccups in creating jobs because of environment variables, dependencies, or any other multitude of reasons. One could argue that complicated cron jobs should not be cron jobs at all, but it might not make sense for a business or sole developer to create a more sophisticated solution at the cost of taking time away from the project or product they are working on.
Also, if the machine one picks just happens to be running [CoreOS](https://coreos.com/) (or if one just feels like it), it might make more logical sense to configure an instance or machine with SystemD timers instead of trying to set up a traditional cron runner of sorts inside a container or via some other means.
## Config
I will be running CoreOS and creating a timer unit and service unit with a cloud-config. The cloud-config  is the user_data one specifies for an `aws_instance` in a tool like [terraform](https://terraform.io/), or the input for the [User Data](https://www.digitalocean.com/company/blog/automating-application-deployments-with-user-data/) textbox on Digital Ocean's Droplet creation interface. Cloud-configs can be used to configure VPSs (virtual private servers) and work across many cloud providers. AWS and DigitalOcean are the main services/providers I use. Most likely one's cloud/infrastructure provider has some way of providing a CoreOS instance a cloud-config, and if not there's no shortage of other providers.
```
#cloud-config
coreos:
  units:
    - name: docker.service
      enable: true
    - name: job.timer
      command: start
      content: |
        [Unit]
        Description=A timer that starts the job service and runs it every Friday at 12 pm (noon)
        Requires=docker.service

        [Timer]
        OnCalendar=Fri *-*-* 12:00:00
    - name: job.service
      content: |
        [Unit]
        Description=The job service, which just runs a container
        Requires=docker.service

        [Service]
        TimeoutSec=0
        Type=simple
        ExecStartPre=/usr/bin/docker pull registry.example.com/job:latest
        ExecStart=/usr/bin/docker run --rm registry.example.com/job:latest
```
## Explanation:
Above I've created a cloud-config that:

* enables docker
* creates the job.timer unit file
* configures the job.timer to start
* creates the content of the job.timer with [`Fri *-*-* 12:00:00`](http://www.freedesktop.org/software/systemd/man/systemd.time.html) referring to every Friday at noon 
* creates the job.service unit file, which:
 * requires `docker.service`
 * sets the timeout to infinity (e.g. never timeout)
 * sets the type to simple (this is good for units that exit)
 * pulls the docker image associated with the job every time the service starts
 * starts the container with `--rm` which ensures the container will be cleaned up upon exit

Some of these choices in the cloud-config, like the time the job executes and executing `docker run` with `--rm` are easily configurable options that change based upon situation, but others like `TimeoutSec=0` and `Requires=docker.service` are good choices when your service runs a container specifically because of the time it takes to download an image, and the explicit need of a docker daemon to do anything locally with docker.

## Conclusion
I started writing this post because I wasn't sure if the timers worked exactly like cron jobs and only executed at the times that the `OnCalendar=` field in the timer unit that matched (e.g. on boot and at 12pm?), but from working on a small project and doing some troubleshooting at the beginning, it looks like most of that is handled by more involved fields specified in the [documentation](http://www.freedesktop.org/software/systemd/man/systemd.timer.html). I hope the description of all the pieces above prove helpful though to anyone that might stumble across it.

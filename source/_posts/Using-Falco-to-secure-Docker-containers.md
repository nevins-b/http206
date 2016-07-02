---
title: Using Falco to secure Docker containers
date: 2016-07-01 21:28:56
tags:
  - security
  - Docker
  - Falco
---
I've been looking for an excuse to use [Sysdig](http://www.sysdig.org/) since I first heard about it at a bsides BOS conference a couple of years ago. This came to a head recently while at [DockerCon](http://2016.dockercon.com/) in Seattle.

While there I had a chance to look at the new Sysdig monitoring for Docker which is pretty damn cool. The more interesting piece though was their newly opensourced tool [Falco](http://www.sysdig.org/falco/). Falco, in a nut shell, lets to create rules that will monitor for and alert on basically anything that is happening on your linux system. So now I needed to find a reason to use it.

This brings us back to Docker. The security of Docker is something that is very interesting to me. One feature which I've looked at before but always had a hard time figuring out how to use well is [Seccomp](https://github.com/docker/docker/blob/master/docs/security/seccomp.md). Seccomp profiles with Docker allow you to limit what syscalls your Docker container is allowed to make to the underlying system. By default Docker disables about 40 out of 300 syscalls. That's pretty good, but lets see if we can leverage Falco to do better.

Let's start with Falco. Falco is container aware which means we can create a rule really simply.
```yaml
- rule: container_syscalls
  desc: Capture syscalls for any docker container
  priority: informational
  condition: container.id != host and syscall.type exists
  output: %container.id:%syscall.type
```
This rule simply states, if you see a container making a syscall log the container's id and the syscall that is being made. With that rule in place, and Falco configured to log JSON, we can start the Falco daemon and running a Nginx Docker container gives us the following output:

```json
{"put":"stuff here"}
{"put":"stuff here"}
```

That's awesome! We now have a log of all the syscalls being made by that Docker container. So what do we do with this?

To help with this task I've released [falco2seccomp](https://github.com/nevins-b/falco2seccomp). This is a pretty simple Go project written to parse the output from Falco and generate a ready to go seccomp profile. Here's example usage:

```shell
falco2seccomp -log events.log -container-id <something>
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "name": "accept",
            "action": "SCMP_ACT_ALLOW",
            "args": []
        },
        {
            "name": "accept4",
            "action": "SCMP_ACT_ALLOW",
            "args": []
        },
        ...
    ]
}
```

This is a ready to use out of the box seccomp profile for our nginx container, limited to just the syscalls we actually saw the container using. Instead of blacklisting 40 syscalls, we're only allowing 44. Again, awesome.

So how can you use this in the real world? One thought would be to integrate it into a CI/CD pipeline. If your builds are being done in a Docker container your tests should be exercising your code enough to generate the full list of syscalls required. boom.

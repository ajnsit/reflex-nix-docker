# reflex-nix-docker

[Reflex-Platform](https://github.com/reflex-frp/reflex-platform) depends on Nix which is poorly supported on non-Linux platforms like MacOS (See [State of Nix on OSX](https://github.com/NixOS/nixpkgs/issues/18506)). However it's possible to have a fairly comfortable dev environment with [Docker](https://docker.com).

## Setup a *case sensitive* filesystem

For best results, put your source code on a case sensitive filesystem. On MacOS you can use the *Disk Utility* to create a new partition, and then format it as a case-sensitive filesystem. The following will assume that the sources are in `/Volumes/Dev/myproj`.

## Setup Docker

Install Docker - [Docker for Mac](https://docs.docker.com/docker-for-mac/).

## Build the Base Nix image

It's recommended to have one one Dockerfile for building a reusable nix image, and separate Dockerfile for each project that uses that nix image.

To build the base image -

```
$ docker build -f Dockerfile.basenix -t basenix .
```

## Build the project image

Create a Dockerfile for your project which builds FROM basenix, and downloads and builds your sources with nix. Here we take the example of building the reflex platform -

```
$ docker build -f Dockerfile.reflex -t reflex .
```

## Create a container

Run the docker image with the command below. This will start a new container from that image, and mount the local code directory into `/home/snow/myproj` (the default user created is `snow`).

```
$ docker run -i -t -v /Volumes/Dev/myproj:/home/snow/myproj -P reflex /sbin/my_init -- bash -l
```

The first time you log into the container (using docker run command above), you would see something like -

```
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
No SSH host key available. Generating one...
Creating SSH2 RSA key; this may take some time ...
2048 SHA256:uTaKjimoj8FLGoYZMhr69lNNR+puL61iR+5SiguaQcY root@17aca8342da2 (RSA)
Creating SSH2 DSA key; this may take some time ...
1024 SHA256:iNtUfUnPxFdzJ20pYTp470lU28ZOuyXk/kPQZn0CVik root@17aca8342da2 (DSA)
Creating SSH2 ECDSA key; this may take some time ...
256 SHA256:2SNQ6XODFP6KNAgdIc6co/NCspaCxd90HNYjeghY1Po root@17aca8342da2 (ECDSA)
Creating SSH2 ED25519 key; this may take some time ...
256 SHA256:FredhjrNilydfrB2EanAD1qrVGOejaP8TUKoS9KAsaw root@17aca8342da2 (ED25519)
invoke-rc.d: could not determine current runlevel
invoke-rc.d: policy-rc.d denied execution of restart.
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 78
*** Running bash -l...
Aug  2 07:42:25 17aca8342da2 syslog-ng[88]: syslog-ng starting up; version='3.5.6'
root@17aca8342da2:/home/snow/myproj#
```

`17aca8342da2` is the hash of the new container created.

When you exit, the container is stopped but the state is saved -

```
root@17aca8342da2:/home/snow/myproj# logout
*** bash exited with status 0.
*** Shutting down runit daemon (PID 78)...
*** Killing all processes...

$ docker ps -a
CONTAINER ID        IMAGE        COMMAND                  CREATED             STATUS                          PORTS               NAMES
17aca8342da2        reflex       "/sbin/my_init -- bas"   18 minutes ago      Exited (0) About a minute ago                       peaceful_turing
```

Once the container has been created, you can reuse the same container instead of creating
new containers off the image.

```
$ docker start 17aca8342da2
17aca8342da2

$ docker ps
CONTAINER ID        IMAGE        COMMAND                  CREATED             STATUS              PORTS                     NAMES
17aca8342da2        reflex       "/sbin/my_init -- bas"   20 minutes ago      Up 11 seconds       0.0.0.0:32770->8000/tcp   peaceful_turing

$ docker exec -i -t 17aca8342da2 bash
root@17aca8342da2:~/myproj#
```

## Use the container for development

**IMPORTANT NOTE:*** You should always switch to the unprivileged user for development -

```
root@17aca8342da2:~/myproj# su snow
snow@17aca8342da2:~/myproj$
```

The code resides on the *host* machine at `/Volumes/Dev/myproj` and can be edited
using any native editor. After making changes, you can run the build inside the container -

```
snow@17aca8342da2:~/myproj$ make # Or whatever
...
```

## Access the app from the host machine

If you look at the command we used to create the server, we used `-P` to *publish*
all the ports exposed by the machine. This proxies the port `8000` to a port on the
host machine.

You can run `docker ps` to see what port it has been mapped to. For example -

```
$ docker ps
CONTAINER ID        IMAGE        COMMAND                  CREATED             STATUS              PORTS                     NAMES
17aca8342da2        reflex       "/sbin/my_init -- bas"   3 seconds ago       Up 2 seconds        0.0.0.0:32769->8000/tcp   peaceful_turing
```

So the app can be accessed on `http://localhost:32769` on the *host* machine.

# Concourse Podman

This repository is a fork of [concourse/concourse-docker](https://github.com/concourse/concourse-docker) - it ditches packaging the image and only focuses on getting Concourse running with rootless** Podman + Podman Compose / Docker Compose 

Refer to `concourse/concourse-docker` for configuration & other documentation.

** you need root access to setup a kernel module - containers runs in rootless mode

## The Basics

### Inserting some kernel modules
You need to run:
```sh
$ modprobe ip_tables
$ modprobe iptable_filter
```
as root to get around this error when starting up worker:
```
containerd-garden-backend exited with error: setup host network failed: create chain or flush if exists failed: running [/usr/sbin/iptables -t filter -N CONCOURSE-OPERATOR --wait]: exit status 3: iptables v1.8.7 (legacy): can't initialize iptables table `filter': Table does not exist (do you need to insmod?)
```

I faced this with netavark stack - I haven't tested with CNI stack.

You can persist this by appending module names to `/etc/modules`. After this, all the steps can be executed as rootless user.

### Getting started

Clone this repo
```sh
$ git clone https://github.com/Sid-Sun/concourse-podman
$ cd concourse-podman
```

The `docker-compose.yml` in this repo will get you up and running with the
latest version of Concourse. To use it you'll first need to execute
`./keys/generate` - this will generate credentials used to authorize the
Concourse components with each-other:

```sh
$ ./keys/generate
wrote private key to /keys/session_signing_key
wrote private key to /keys/tsa_host_key
wrote ssh public key to /keys/tsa_host_key.pub
wrote private key to /keys/worker_key
wrote ssh public key to /keys/worker_key.pub
```

### Post container create notes:
The default configuration sets up a `test` user with `test` as their password
and grants them access to `main` team. To use this in production you'll
definitely want to change that - see [Auth &
Teams](https://concourse-ci.org/auth.html) for more information..

By default, `docker-compose.yml` sets `restart: always` so `podman-restart.service` can restart the containers on reboots (assuming it is enabled)

## Running with `docker-compose`

### Getting docker-compose
- You can get either docker-compose or [docker compose v2](https://github.com/docker/compose) (this repo has been tested with v2) - you can use either your distro's package manager or for v2:
  - Download the binary from docker compose git repo and place it in your `$PATH` with name `docker-compose` (yes - placing v2 compose as v1's name is fine)
### Setup docker-compose with podman
- Start the Podman socker for your user `systemctl start --user podman.socket` & enable it `systemctl enable --user podman.socket`
- set the `DOCKER_HOST` environment variable `export DOCKER_HOST=unix:///run/user/$UID/podman/podman.sock`

### Start the containers
Run `docker-compose up -d` to start Concourse in the background:

```sh
$ docker-compose up -d
Starting concourse-podman_db_1 ... done
Starting concourse-podman_web_1 ... done
Starting concourse-podman_worker_1 ... done
```
### Ensure containers are running:
```sh
$ docker-compose ps
```
or
```sh
$ podman ps -a
```

### If things seem to be going wrong, check the logs for any errors:

```sh
$ docker-compose logs -f
```
or
```sh
$ podman logs <container>
```

## Running with `podman-compose`

### Getting podman-compose
- Install [podman-compose](https://github.com/containers/podman-compose) if not already installed

### Start the containers
Run `podman-compose up -d` to start Concourse in the background:

```sh
$ podman-compose up -d
```

### Ensure containers are running:
```sh
$ podman-compose ps
['podman', '--version', '']
using podman version: 4.2.0
podman ps -a --filter label=io.podman.compose.project=concourse-podman
CONTAINER ID  IMAGE                                 COMMAND     CREATED         STATUS             PORTS                   NAMES
d3115c8d24fa  docker.io/library/postgres:latest     postgres    22 minutes ago  Up 22 minutes ago                          concourse-podman_db_1
97e8efa56d5e  docker.io/concourse/concourse:latest  web         22 minutes ago  Up 22 minutes ago  0.0.0.0:8080->8080/tcp  concourse-podman_web_1
1b57d6909298  docker.io/concourse/concourse:latest  worker      22 minutes ago  Up 22 minutes ago                          concourse-podman_worker_1
exit code: 0
```
or
```sh
$ podman ps -a
```

### If things seem to be going wrong, check the logs for any errors:

```sh
$ podman-compose logs -f
```
or
```sh
$ podman logs <container>
```

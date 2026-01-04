# Docker Free

This guide is for you if you want to use docker with Windows but cannot live with the [license restrictions](https://docs.docker.com/subscription/desktop-license/) of [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/).

If you are anything like me then you just want the ability to run some Linux containers on your Windows workstation,
for example, so that you can make build scripts that use [TestContainers](https://testcontainers), work.

With the recipe here, you will run the docker engine inside WSL2. But you can use that engine from Windows too as it will
be accessible on `localhost:2376`.

Result: You'll get a safe, well-performing docker environment without license restrictions. 
You can use it from Windows or WSL2/Linux applications alike. It will only be able to run Linux containers, not Windows
containers. However, the latter is really, really rare, so it shouldn't be something you would miss.


<br>

## Alternatives

- **Rancher Desktop**? Installs too many other things for my taste.
- **Podman**?  Many tools expect docker, not Podman.

<br>
<br>

## Prerequisites

This guide assumes that you already have WSL2 running with an Ubuntu distro, v22 or later.

<br>
<br>
<br>

## Installation


<br>
<br>

### Step 1 - Install docker engine inside WSL

1. Install docker engine inside WSL (Ubuntu) by following the [guide from Docker Inc.](https://docs.docker.com/engine/install/ubuntu). 
Use the "apt repository method" for installation. It is the only future proof solution. 
1. Post-installation:   `sudo usermod -aG docker $USER`
1. Logout of your Ubuntu shell and login in again. You should now be a member of the `docker` group.
   This can be verified with the `groups` command. This means you don't have to use sudo everytime you want to use the `docker` command in
   Ubuntu.

If you want, you can verify your docker engine installation at this point by using **Step 6a**. However, the docker daemon will not 
yet have been secured, nor is it yet available from Windows side. But it should still work.

<br>
<br>

### Step 2 - Generate certificates for docker engine

(In WSL/Ubuntu)

You will need to secure your docker daemon from unauthorized access. Docker (the company) says that in the future they will
no longer allow unprotected daemons to start. It works for now but we better make ourselves future proof. And safe.

The way to do this is to protect the docker daemon with mTLS (mutual TLS). The docker daemon will encrypt the traffic with
a server-side certificate so effectively it will be exposing `https`, not `http`. At the same time, the server will only
allow clients which provide an acceptable client-side certificate during the TLS handshake, to connect. In others words: 
both sides of the connection have a certificate/private-key combo.

How?

Execute the [docker-create-certs.sh](docker-create-certs.sh) script. It will generate custom keys and certificates
needed for docker engine _and_ put them in their right place.

It should output something like this:

```
Generating CA's root key
Generating CA certificate

Generating server-side key
Generating server-side certificate
Certificate files for docker engine now exist in /etc/docker/certs

Generating client-side key
Generating client-side certificate
Certificate files for docker CLI now exist in /home/john/.docker
Certificate files for docker CLI now exist in C:\Users\john\.docker

Done!

Set the following environment variables in Windows OS:
  DOCKER_HOST=tcp://localhost:2376
  DOCKER_TLS_VERIFY=1
  DOCKER_CERT_PATH=%USERPROFILE%\.docker
```

The generated certificates will be valid for the next 50 years. Should be enough 😄

<br>
<br>

### Step 3 - Make the docker daemon run on TCP (too)

(In WSL/Ubuntu)

In order for an application executing in Windows to reach the docker daemon in WSL, it must be exposed on a TCP port.
For this, we need to change the daemon's startup options. The easiest way to do this is
by overriding the Systemd service named `docker`.

Now, before we make any changes, it may be worth knowing what the existing command line to start the docker daemon is:

```bash
sudo systemctl cat docker.service | grep ExecStart -m 1
```

which will give you:

```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

We want to add some extra command line options, in particular what we want to **add
an additional `-H` option**. This option controls what sockets the daemon listens too, and it can be applied
multiple times on the command line. We also want to add some TLS stuff to make the daemon secure.

Create and apply an override on the Systemd service named `docker`:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo cat <<EOF > /etc/systemd/system/docker.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd \
    --host=fd:// \
    --host=tcp://:2376 \
    --containerd=/run/containerd/containerd.sock \
    --tlsverify=true \
    --tlscacert=/etc/docker/certs/ca.pem \
    --tlscert=/etc/docker/certs/cert.pem \
    --tlskey=/etc/docker/certs/key.pem
EOF
sudo systemctl daemon-reload --system
sudo systemctl restart docker.service
```


:information_source: If you use a value of `"tcp://:2376"` (i.e., leaving out the hostname part) for the `--host` option
it will bind to the *IPv4* loopback interface. This is most likely what you want. 
By contrast, if you use a value of `"tcp://0.0.0.0:2376"` it will only bind to the *IPv6* loopback interface.
Which is most likely not what you want.

:information_source: In Systemd, the `ExecStart` directive is additive. This is why it needs to be 
set to an empty string first.

<br>

Verify:

```bash
ps -efd | grep dockerd
```

<br>
<br>

### Step 4 - Install docker CLI in Windows (optional)

At this point, you already have a docker CLI, the `docker` command, in WSL/Ubuntu.
If you also need one in Windows, then here is how:


1. Grab Docker Inc's pre-build binary for Windows from [HERE](https://download.docker.com/win/static/stable/x86_64). 
Use the latest version. All you need is the file named `docker.exe`.
1. Put `docker.exe` somewhere on your disk. For example, create a directory named `C:\Program Files\Tools` and put it
into that one. Then make sure your Windows `PATH` variable includes that directory.
1. You should now be able to use the `docker` command.

(If you try it at this point, it may not work, because some environment variables still need to be set ... and
indeed the `docker.exe` uses those to figure out where to find the docker daemon. So continue to next step.)

Note: The docker CLI is really just a binary that helps you communicate with the docker daemon, using the docker API.
The TestContainers library does not depend on the docker CLI as it can communicate directly with the docker daemon 
simply by using the docker API (which is essentially a http REST api). For this reason, the docker CLI is _not_
a requirement for TestContainers.

<br>
<br>

### Step 5 - Define Windows environment variables

Set environment variables as follows:

```
DOCKER_HOST=tcp://localhost:2376
DOCKER_TLS_VERIFY=1
DOCKER_CERT_PATH=%USERPROFILE%\.docker
```

Example:

![image](https://github.com/user-attachments/assets/28b347c6-39eb-4600-870f-9424fb25da76)


<br>
<br>


### Step 6 - IntelliJ IDEA (optional)

IntelliJ IDEA has a nice UI for Docker. If you want to use it you'll need to tell the IDE that you are using the Docker Engine
inside WSL, meaning you are not using "Docker for Windows" which is what the IDE will assume by default.

Here is the setting:

![image](https://github.com/user-attachments/assets/cd265d49-e8d5-4c88-b84a-f587ccc18488)

Ref: [Jetbrains's documentation](https://www.jetbrains.com/help/idea/settings-docker.html)

<br>
<br>

### Step 7a - Verify your docker engine installation (from WSL/Ubuntu)

(In WSL/Ubuntu)

```
docker run hello-world
```

This should print a lot of stuff on the console, most importantly, "Hello from Docker!"


<br>
<br>

### Step 7b - Verify your docker engine installation (from Windows)

(In Windows - this step assumes you've done Step 4 previously)


```
docker run hello-world
```

This should print a lot of stuff on the console, most importantly, "Hello from Docker!"


<br>
<br>

### Step 7c - Verify your docker engine installation with TestContainers (from Windows)

(In Windows)

If you are a Java developer and use TestContainers, then you use the project in 
[testcontainers-verification](testcontainers-verification/) to verify that it works.


<br>
<br>

### Step 8 - Make WSL start automatically when Windows starts (optional)

Personally, I can live with always starting a WSL/Ubuntu session in order to have my docker running. 
After all, WSL2 does take up some resources, although it is surprisingly effective.

However, you may want that WSL/Ubuntu is _always_ started.
For this purpose, [Windows Terminal](https://learn.microsoft.com/en-us/windows/terminal/) can be used.
If you do not already have this tool installed, then please do. 

You can configure Windows Terminal to start automatically when Windows starts (or more precisely: when you log in, as it is not a background service)
and then you can make sure that Windows Terminal always starts out with an Ubuntu sesssion. 

Here is what it looks like in the _Settings_ for Windows Terminal:


![image](https://github.com/user-attachments/assets/af17aa86-1aac-4778-8f08-e0895e324c24)


Voila. Because the Terminal starts an Ubuntu shell it effectively needs to start WSL too.


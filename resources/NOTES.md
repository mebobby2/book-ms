# Docker
## Base Images
In systems that host many containers, the size of the base image is less important than how many different base images are used. Remember, each image is cached on the server and reused across all containers that use it. If all your containers are, for example, extending from the debian image, the same cached copy will be reused in all cases meaning that it will be downloaded only once.

## Layers
Docker containers are building blocks for applications. A docker image describes the instructions for running a container. Each of these instructions, contained in the Dockerfile, create a layer. The set of these layers is useful to speed up the build operations and the image transfer. This saves a lot of disk space when many images share the same layers.

```
FROM debian:stable

WORKDIR /var/www

RUN apt-get update
RUN apt-get -y --no-install-recommends install curl
RUN apt-get -y --no-install-recommends install ca-certificates

RUN curl https://raw.githubusercontent.com/gadiener/docker-images-size-benchmark/master/main.go -o main.go

RUN apt-get purge -y curl
RUN apt-get purge -y ca-certificates
RUN apt-get autoremove -y
RUN apt-get clean
```

Each statement, like RUN, creates a new layer. In the previous example, at the top of the layers from the Debian image, Docker creates nine extra layers.

Layers are like Git commits, each level stores the difference between the previous and current versions of the image. And like Git commits are useful if you share them with other branches or, in the Docker world, with other images.

Layers are like Git commits, each level stores the difference between the previous and current versions of the image. And like Git commits are useful if you share them with other branches or, in the Docker world, with other images.

**But layers aren’t free.**

Layers use space and as the levels increase, the final image size also increases. This is because the system keeps all the changes between the various statements.

It may be a good practice to combine the various instructions in a single line. But beware, it’s important to design the Dockerfile to use the layers cache system as much as possible.

```
FROM debian:stable

WORKDIR /var/www

RUN apt-get update && \
    apt-get -y --no-install-recommends install curl \
        ca-certificates && \
    curl https://raw.githubusercontent.com/gadiener/docker-images-size-benchmark/master/main.go -o main.go && \
    apt-get purge -y curl \
        ca-certificates && \
    apt-get autoremove -y && \
    apt-get clean
```

The example above creates just two layers on top of the Debian image instead of nine.

As you can see the last commands used are apt-get autoremove and apt-get clean. It’s very important to delete temporary files that do not serve the purpose of the image, such as the package manager’s cache. These files increase the size of the final image without giving any advantage.

## COPY
COPY copies files from the host file system to the container we are building.

ADD is very similar to COPY. In most cases, it is encouraged to use COPY unless you need additional features that ADD provides (most notably TAR extraction and URL support).

## CMD
CMD provides only the default executor that can be easily overwritten when a container is run.

## Leverage build cache
One important thing left to discuss is the order of instructions in the Dockerfile. On one hand, it needs to be in logical. We can not, for example, run an executable before installing it or change permissions of the run.sh file before we copy it. On the other hand, we need to take in account Docker caching.

When a docker build command is run, Docker will go instruction by instruction and check whether some other build process already created the image. Once an instruction that will build a new image is found, Docker will build not only that instruction but of all those that follow. That means that, in most cases, COPY and ADD instructions should be placed near the bottom of the Dockerfile.

For example if you place an ADD instruction towards the top above a few RUN instructions and anything in the added file or directory has changed, all subsequent layers below, their cache is automatically invalidated regardless.

Even within a group of COPY and ADD instructions, we should make sure to place higher those files that are less likely to change. In our example, we’re adding run.sh before the JAR file and front-end components since later are likely to change with every build. If you execute the docker build command the second time you’ll notice that Docker outputs ```---> Using cache``` in all steps. Later on, when we change the source code, Docker will continue outputting ```---> Using cache``` only until it gets to one of the last two COPY instructions (which one it will be, depends on whether we changed the JAR file or the front-end components).


# apt-get
## no-install-recommends
Every package in Ubuntu has a set required packages, a set of recommended packages and a set of suggested packages. The required packages are dependencies, so their installation is mandatory, but the installation of other two sets can be skipped. The recommended and suggested packages are not essential to the functioning of the package being installed. Disabling the installation of recommendations allows to save a lot of disk space.

Disable recommendations temporally (for single package installation), adding the --no-install-recommends option:

```sudo apt-get install package --no-install-recommends```

# Unix

## Domain Sockets
A unix domain socket is a bidirectional pipe similar to a TCP/IP socket. A server listens for and accepts connections from clients, and then can communicate with the client on the newly accepted connection. What is special about unix domain sockets is that instead of having an IP address and port number, they have a file name as their address. This allows other applications that know nothing about networking to be told to open the file and read or write and the data is sent to the server instead of to the disk.

E.g. ```/var/run/docker.sock``` is a socket.

The .sock extension is used by Unix domain sockets as a convention, but not as a requirement. Unix domain sockets are special files used by different processes to communicate with each other - much like TCP/IP sockets.

docker.sock is the UNIX socket that Docker daemon is listening to. It's the main entry point for Docker API. It also can be TCP socket but by default for security reasons Docker defaults to use UNIX socket. Docker cli client uses this socket to execute docker commands by default. You can override these settings as well.

There might be different reasons why you may need to mount Docker socket inside a container. Like launching new containers from within another container. Or for auto service discovery and Logging purposes. This increases attack surface so you should be careful if you mount docker socket inside a container there are trusted codes running inside that container otherwise you can simply compromise your host that is running docker daemon, since Docker by default launches all containers as root.

Docker socket has a docker group in most installation so users within that group can run docker commands against docker socket without root permission but actual docker containers still get root permission since docker daemon runs as root effectively (it needs root permission to access namespace and cgroups).

## What is the difference between authorized_keys and known_hosts file for SSH?
The known_hosts file lets the client authenticate the server, to check that it isn't connecting to an impersonator. The authorized_keys file lets the server authenticate the user.

### Server authentication
One of the first things that happens when the SSH connection is being established is that the server sends its public key to the client, and proves (thanks to public-key cryptography) to the client that it knows the associated private key. This authenticates the server: if this part of the protocol is successful, the client knows that the server is who it claims it is.

The client may check that the server is a known one, and not some rogue server trying to pass off as the right one. SSH provides only a simple mechanism to verify the server's legitimacy: it remembers servers you've already connected to, in the ~/.ssh/known_hosts file on the client machine (there's also a system-wide file /etc/ssh/known_hosts). The first time you connect to a server, you need to check by some other means that the public key presented by the server is really the public key of the server you wanted to connect to. If you have the public key of the server you're about to connect to, you can add it to ~/.ssh/known_hosts on the client manually.

By the way, known_hosts can contain any type of public key supported by the SSH implementation, not just DSA (also RSA and ECDSA).

Authenticating the server has to be done before you send any confidential data to it. In particular, if the user authentication involves a password, the password must not be sent to an unauthenticated server.

### User authentication
The server only lets a remote user log in if that user can prove that they have the right to access that account. Depending on the server's configuration and the user's choice, the user may present one of several forms of credentials (the list below is not exhaustive).

* The user may present the password for the account that he is trying to log into; the server then verifies that the password is correct.
* The user may present a public key and prove that he possesses the private key associated with that public key. This is exactly the same method that is used to authenticate the server, but now the user is trying to prove its identity and the server is verifying it. The login attempt is accepted if the user proves that he knows the private key and the public key is in the account's authorization list (~/.ssh/authorized_keys on the server).
* Another type of method involves delegating part of the work of authenticating the user to the client machine. This happens in controlled environments such as enterprises, when many machines share the same accounts. The server authenticates the client machine by the same mechanism that is used the other way round, then relies on the client to authenticate the user.

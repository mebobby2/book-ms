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

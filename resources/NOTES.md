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

## Networks vs Links
Links have been replaced by networks. Docker describes them as a legacy feature that you should avoid using. You can safely remove the link and the two containers will be able to refer to each other by their service name (or container_name).

With compose, links do have a side effect of creating an implied dependency. You should replace this with a more explicit depends_on section so that the app doesn't attempt to run without or before redis starts.

If we don't use Networks or Links, we can always use a proxy service (such as nginx, HAProxy, etc). However, for databases, it does not expose any ports to the outside world. A good practice is to expose only ports of services that are publicly accessible. Hence a proxy won't help us here. We need to use a network overlay here so we can access the database using the network layer (IP and NATs) instead of the application layer (service discovery). The network layer handles this automatically whereas if we use service discovery, we have to build it into the service (whether as custom code or a library).

With networks, there is no more need for proxy services to connect containers internally. That is not to say that proxy is not useful, but that we should use a proxy as a public interface towards our services and networking for connecting containers that form a logical group. Docker networking and proxy services have different advantages and should be used for different use cases. Proxy services provide load balancing and can control the access to our services. Docker networking is a convenient way to connect separate containers that form a single service and reside on the same network. A typical use case for Docker networking would be a service that requires a connection to a database. We can connect those two through networking. Furthermore, the service itself might need to be scaled and have multiple instances running. A proxy service with load balancer should fulfill that requirement. Finally, other services might need to access this service. Since we want to take advantage of load balancing, that access should also be through a proxy.

In other words, all communication between containers that compose a single service is done through networking while the communication between services is performed through the proxy.


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

## SIGHUP
The original meaning of SIGHUP was that the user had lost access to the program, and so interactive programs should die. Daemons — programs that don't interact directly with the user — have no need for this behavior and instead often reload their configuration files when they receive SIGHUP. But these are just conventions.

E.g. ```docker kill -s HUP nginx``` nginx allows configuration reloads with a HUP signal.

## Nohup
Nohup makes a program ignore the HUP signal, allowing it to run after current terminal is closed / user logged out. Nohup does not send program to background.

````&``` at the end of command is related to shell job control, allowing user to continue work in current shell session.

Usually nohup and & are combined to launch program which runs after logout of user and allows to continue work at current shell session.

```> /dev/null 2>&1``` is to redirect stdout and stderr to /dev/null to ignore the output.

# Architecture
* Services should be deployed on ports and, potentially, servers that are unknown to us in advance. Flexibility is the key to scalable architecture, fault tolerance, and many other concepts.

# Proxies
Apache is process based, making its performance when faced with a massive traffic less than desirable. At the same time, its resource usage skyrockets easily. If you need a server that will serve dynamic content, Apache is a great option, but should not be used as a proxy.


That leaves us with nginx and HAProxy. If you spend some more time investigating opinions, you’ll see that both camps have an enormous number of users defending one over the other. There are areas where nginx outperforms HAProxy and others where it underperforms. There are some features that HAProxy doesn’t have and other missing in nginx. But, the truth is that both are battle-tested, both are an excellent solution, both have a huge number of users, and both are successfully used in companies that have colossal traffic. If what you’re looking for is a proxy service with load balancing, you cannot go wrong with either of them.

I am slightly more inclined towards nginx due to its better (official) Docker container (for example, it allows configuration reloads with a HUP signal), option to log to stdout and the ability to include configuration files. Excluding Docker container, HAProxy made the conscious decision not to support those features due to possible performance issues they can create.

However, there is one critical nginx feature that HAProxy does not support. HAProxy can drop traffic during reloads. If microservices architecture, continuous deployment, and blue-green processes are adopted, configuration reloads are very common. We can have several or even hundreds of reloads each day. No matter the reload frequency, with HAProxy there is a possibility of downtime.

# Pipelines
* It is crucial not only to have the right process in place but also to have the scripts, configurations and the code properly located. Everything that is common to multiple projects should be centralized (as is the case with Ansible playbooks located in the vfarcic/ms-lifecycle137 repository). On the other hand, things that might be specific to a project should be stored in the repository that project resides in. Storing everything in one centralized place would introduce quite a lot of waiting time since a project team would need to request a change from the delivery team. The other extreme is just as wrong. If everything is stored in the project repositories, there would be quite a lot of duplication. Each project would need to come up with scripts to set up servers, deploy a service, and so on.

* Please keep in mind that the shell module should be avoided in most cases. The idea behind Ansible is to specify the desired behavior and not the action that should be performed. Once that “desire” is run, Ansible will try to “do the right thing”. If, for example, we specify that some package should be installed, Ansible will check whether such a package already exists and do the installation only if it doesn’t. The shell module that we used, in this case, will always run, no matter the state of the system. In this particular situation, that is OK, because Docker itself will make sure that only changed layers are built. It won’t build the whole container every time. Please keep this in mind when designing your roles.

* One of the main reasons companies are choosing some other tool (especially Bamboo and Team City) are their enterprise offerings. When an organization becomes big, it needs support and reliability that comes with it.

## CI/CD Platforms
### First Generation
Operations, maintenance, monitoring, and the creation of jobs is mostly done through their UIs.

### Second Generation
Instead of defining your jobs in a centralized location, those tools would inspect your code and act depending on the type of the project. If, for example, they find build.gradle file, they would assume that your project should be tested and built using Gradle161. As the result, they would run gradle check to test your code and, if tests passed, follow it by gradle assemble to produce the artifacts. Based themselves on auto-discovery.

### Third Generation
Different approaches belong to different contexts and types of tasks. Jenkins and similar tools benefit greatly from their UIs for monitoring and visual representations of statuses. The part it fails with is the creation and maintenance of jobs. That type of tasks would be much better done through code. With Jenkins, we had the power but needed to pay the price for it in the form of maintenance effort.

Combining the best of both generations, and some more. We can write a whole pipeline using a DSL. Example: Jenkins Workflow and Jenkinsfile. Newly introduced Jenkinsfile allows us to define the Workflow script inside the repository together with the code. That means that developers in charge of the project can be in control of the CI/CD pipeline as well.

Overall Jenkins management is centralized while individual CI/CD pipelines are placed where they belong (together with the code that should be moved through it).

Moreover, if we combine all that with the Multibranch Workflow job type, we can even fine tune the pipeline depending on the branch.

## Jenkins
With Jenkins Workflow and Groovy DSL removes the need for deployment defined in Ansible. We’ll keep using Ansible playbooks for provisioning and configuration since those are the areas it truly shines. On the other hand, Jenkins Workflow and Groovy DSL provide much more power, flexibility, and freedom when defining the deployment process. The main difference is that Groovy is a scripting language and, therefore, provides a better syntax for this type of tasks. At the same time, its integration with Jenkins allows us to utilize some powerful features. For example, we could define five nodes with a label tests. Later on, if we specify that some Workflow instructions should be run on a tests node, Jenkins would make sure that the least utilized of those five nodes is used (or there might be a different logic depending on the way we set it up).

At the same time, by using Jenkins Workflow, we’re avoiding complicated and not easy to understand XML definitions required by traditional Jenkins jobs and reducing the overall number of jobs.

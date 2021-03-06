# Book MS

## Prerequisites
* Git
* Vagrant (virtualbox by dependency)

## Setup
1. Speed up the creation of new VMs. It caches all packages that are being used so that the next time we need them, they are obtained from the local HD instead being downloaded. ```vagrant plugin install vagrant-cachier```
2. ```vagrant up dev```
3. ```sudo apt-get install jq```

## Development
1. ```vagrant up dev```
2. ```cd /vagrant```
3. Build the container for running tests (done once only): ```sudo docker build -f Dockerfile.test -t 10.100.198.200:5000/books-ms-tests .```
4. Runs the pre-deployment tests and compile the Scala code: ```sudo docker-compose -f docker-compose-dev.yml run --rm tests```
5. Build the microservice container: ```sudo docker build -t 10.100.198.200:5000/books-ms .```
6. Run the db: ```sudo docker run -d --name books-db mongo```
7. Run microservice: ```sudo docker run -d --name books-ms -p 8080:8080 --link books-db:db 10.100.198.200:5000/books-ms```
* Steps 6 & 7 can be ran using docker-compose: ```docker-compose -f docker-compose-dev.yml up -d app```
* Push container to registry: ```sudo docker push 10.100.198.200:5000/books-ms```

## Tests
1. ```vagrant up dev```
2. ```vargrant ssh dev```
3. ```cd /vagrant```
4. Build the container for running tests (done once only): ```sudo docker build -f Dockerfile.test -t 10.100.198.200:5000/books-ms-tests .```
5. Run front-end tests: ```sudo docker-compose -f docker-compose-dev.yml run feTestsLocal```
6. Run all tests and package JAR: ```sudo docker-compose -f docker-compose-dev.yml run testsLocal```

## Start Local Docker Registry
1. cd into the ms-lifecycle project
2. ```vagrant up cd --provision```
3. Registry is hosted at 10.100.198.200:5000

## Set up Prod Machine + Deploy Microservice
1. ```vagrant up prod --provision```
2. From cd vm: ```ansible-playbook /vagrant/ansible/prod2.yml -i /vagrant/ansible/hosts/prod```
3. ```wget https://raw.githubusercontent.com/vfarcic/books-ms/master/docker-compose.yml```
4. ```export DOCKER_HOST=tcp://prod:2375```
5. ```docker-compose up -d app```
6. Visit ```http://10.100.198.201:8500/ui``` to see Consul UI

## Accessing Microservice
* ```curl -H 'Content-Type: application/json' -X PUT -d "{\"_id\": 1, \"title\": \"My First Book\", \"author\": \"John Doe\", \"description\": \"Not a very good book\"}" http://prod:$PORT/api/v1/books | jq '.'```
* ```curl http://prod:$PORT/api/v1/books | jq '.'```

## Setup Service Discovery
1. ```vagrant up cd serv-disc-01 serv-disc-02 serv-disc-03```
2. ```vagrant ssh serv-disc-01```
3. Install by running these commands on serv-disc-0:
```curl -L https://github.com/coreos/etcd/releases/download/v2.1.2/etcd-v2.1.2-linux-amd64.tar.gz -o etcd-v2.1.2-linux-amd64.tar.gz
tar xzf etcd-v2.1.2-linux-amd64.tar.gz
sudo mv etcd-v2.1.2-linux-amd64/etcd* /usr/local/bin
rm -rf etcd-v2.1.2-linux-amd64*
etcd >/tmp/etcd.log 2>&1 &
```
4. Or, run this playbook from the cd VM: ```ansible-playbook /vagrant/ansible/etcd.yml -i /vagrant/ansible/hosts/serv-disc```
5. Then run this playbook from the cd VM to set up Registrator: ```ansible-playbook /vagrant/ansible/registrator-etcd.yml -i /vagrant/ansible/hosts/serv-disc```
6. Then run this playbook from the cd VM to set up confd: ```ansible-playbook /vagrant/ansible/confd.yml -i /vagrant/ansible/hosts/serv-disc```

## Set up Jenkins

### Server
1. from cd vm, provision the jenkins server: ```ansible-playbook /vagrant/ansible/jenkins.yml -c local```
* Jenkins UI: ```http://10.100.198.200:8080```

### Node: swarm-master
1. From cd vm, provision the jenkin nodes: ```ansible-playbook /vagrant/ansible/jenkins-node-swarm.yml -i /vagrant/ansible/hosts/prod```
2. From swarm-master vm, ```wget http://10.100.198.200:8080/jnlpJars/agent.jar```
3. From swarm-master, run this command as the *ubuntu* user: ```java -jar agent.jar -jnlpUrl http://10.100.198.200:8080/computer/swarm-master/slave-agent.jnlp -workDir "/data/jenkins_slaves/swarm-master" > /dev/null 2>&1 &```. (This is assuming the agent's Launch Method is set to 'Launch agent by connecting it to the master')
* Fix the port for inbound agents: Manage Jenkins -> Configure Global Security -> TCP port for inbound agents. Set it to Fixed and port 50000

### Node: prod
1. From cd vm: ```ansible-playbook /vagrant/ansible/jenkins-node.yml -i /vagrant/ansible/hosts/prod```
2. From prod vm, ```wget http://10.100.198.200:8080/jnlpJars/agent.jar```
3. Start the Jenkin nodes: ```java -jar agent.jar -jnlpUrl http://10.100.198.200:8080/computer/prod/slave-agent.jnlp -workDir "/data/jenkins_slaves/cd" > /dev/null 2>&1 &```. (This is assuming the agent's Launch Method is set to 'Launch agent by connecting it to the master')

### Update Jenkin
The Jenkins image was built by the author in Feb 2018. The Dockerfile specifies the latest version of Jenkins, but when it was build in 2018, the latest version then, is pretty old as of 2020... We will rebuild the image and push it to our private docker registry instead.
1. from ```/vagrant``` directory inside the ```cd``` vm
2. ```sudo docker build -f Dockerfile.jenkins -t 10.100.198.200:5000/jenkins .```
3. ```docker push 10.100.198.200:5000/jenkins```

### First Time Password
Jenkins will ask for the admin password on first load
1. from cd vm: ```sudo docker exec -it container_id bash```
2. ```cat /root/.jenkins/secrets/initialAdminPassword```

## Docker Swarm
1. ```vagrant up cd swarm-master swarm-node-1 swarm-node-2```
2. ```vagrant ssh cd```
3. Provision servers: ```ansible-playbook /vagrant/ansible/swarm.yml -i /vagrant/ansible/hosts/prod```
4. Create network overlay: ```docker network create --driver overlay books-ms-nw```
5. Deploy db: ```docker service create --name books-ms-db --network books-ms-nw --publish 27017:27017 --reserve-cpu=0.5 --reserve-memory=1GB mongo```
6. Deploy app: ```docker service create --name books-ms-app --network books-ms-nw --publish 8080:8080 --env SERVICE_NAME=books-ms --env DB_HOST=books-ms-db 10.100.198.200:5000/books-ms```
7. Scale service: ``` ```

### Why can't we use docker-compose?
Docker-compose requires version 1.25 of docker API to deploy to a swarm, which uses e.g. ```docker stack deploy --compose-file docker-compose.yml``` commands. We are using 1.24, and can't upgrade because we are on an older version of Ubuntu. Hence, we have to switch to using manual deployments using ```docker service``` commands.

## Test Microservice
1. ```curl http://swarm-master/api/v1/books  | jq '.'```
2. ```curl -H 'Content-Type: application/json' -X PUT -d "{\"_id\": 1, \"title\": \"My First Book\", \"author\": \"John Doe\", \"description\": \"Not a very good book\"}" http://swarm-master/api/v1/books | jq '.'```

## Helpful Commands
* ```ll target/scala-2.10``` - list files in a directory.
* ```sudo docker exec -it books-ms bash``` - start a bash session inside the container
* ```docker logs books-ms``` - view the logs of the container
* ```docker rm -f books-ms books-db``` - remove running containers
* ```docker-compose logs``` - check logs of all running containers
* ```docker-compose stop``` - stop all containers
* ```docker-compose rm -f``` - remove all containers (the stop command doesn't remove them)

## Upto
Page 327

Now that we know how easy it is to add a check in Consul,

Not following the book anymore because
1. I've lost my old laptop where this was originally set up
2. Too much effort to set up on new laptop
3. Most of the tech (consul, ELK monitoring, splunk) are quite old and not worth knowing

So, I'm just reading it the rest of the book for completeness.

Finished the book!

## Battle Notes

Before that: Automate workflow-util.groovy to get the Jenkensfile https://github.com/vfarcic/books-ms/blob/swarm/Jenkinsfile  working. Stuck in the problem where ```docker service create --name ${serviceName}-green --network ${serviceName}-nw --publish 8080:8080 --env SERVICE_NAME=${serviceName}-green --env DB_HOST=${serviceName}-db 10.100.198.200:5000/${serviceName}``` is failing because port '8080' is already use by {serviceName}-blue for the second deployment (The Green deployment). Look into ```docker service update``` using the arguments --publish-add to try to do something.

Steps for testing:
1. vagrant up cd swarm-master swarm-node-1 swarm-node-2 --provision
2. cd: ansible-playbook /vagrant/ansible/jenkins.yml -c local
3. cd: ansible-playbook /vagrant/ansible/swarm.yml -i /vagrant/ansible/hosts/prod
3.5. Provision the swarm master as a jenkins node and download agent.jar first if swarm-master is a brand new node
4. swarm-master: java -jar agent.jar -jnlpUrl http://10.100.198.200:8080/computer/swarm-master/slave-agent.jnlp -workDir "/data/jenkins_slaves/swarm-master" > /dev/null 2>&1 &
5. http://10.100.198.200:8080, then build books-ms-swarm

6. To test: curl http://swarm-master/api/v1/books  | jq '.'

Sidetracks:

Before that: Success! But figure out why books-ms can't connect to mongodb. Can connect to mongodb if set up the service manually, but doesnt seem to work when automated through the jenkins scipts. Looks like the books-ms-db container is not added into the virtual network. It is added when the service is ran from swarm-master, but not when its ran from cd vm.

I think it was because docker on swarm-master and cd were different versions... cd = old docker-engine, swarm-master = new docker-ce

Anyways, turns out apt.dockerproject.org repo is shutdown so I need to use download.docker.com. However, the new repo does not have docker-engine, it only has docker-ce. However, docker-ce reqires xenial and not trusty. I've upgraded to xenial but in the process I have deleted the book-ms and jenkins image from my private repo. book-ms was build and pushed but jenkins is failing to build. This is because ubuntu cosmic has reached End of Live and so I need to switch from using archive.ubuntu.com repot to old-releases.ubuntu.com following this thread: https://superuser.com/questions/1527250/apt-update-error-with-ubuntu-18-10-cosmic-version

Yay, building ubuntu with old packages seem to work. However, getting a 'No space left on device' error while building.
'docker system prune' does not help.
/var/lib/docker is where docker store image files. 'df -h /var/lib/docker' shows it has 10gb of total space. The disk mount is /dev/sda1. Following steps 1 & 2 on https://tvi.al/resize-sda1-disk-of-your-vagrant-virtualbox-vm/ solved this issue. The jenkins image is built and pushed into our local registry.

Works! blue/green deployments work. However, running into a problem when executing commands from cd vm, the mongo db and books-ms containers are not being added to the books-ms-nw network overlay. So, same problem as outlined a few paragraphs above.

Figured out why books-ms and mongo db are not added to the network overlay. To do with different docker versions on the VMs?

So, I upgraded the docker engine on the swarm nodes. However, I forgot to update the systemd unit fields for docker, so it meant I could start docker manually with dockerd, but could not do so via 'systemctl start docker.service'. After updating the systemd unit files to use the new start command for the newer version of docker, everything worked! Running the jenkins pipeline to deploy the book-ms and mongo db worked! And mongo and book-ms were added to the same network overlay and were able to connect!

THere is small issue - using DOCKER_HOST on the cd vm pointing to swarm-master gets a unable to connect error. However, if I:
1. manually ssh into swarm-master and do 'systemctl stop docker.service' and 'systemctl start docker.service' it Works
2. run the jenkins pipeline (which restarts docker) it works.

Can't be bothered fixing this so in the meantime, just use this workaround...

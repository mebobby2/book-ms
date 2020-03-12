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
* Push container to registry: ```docker push 10.100.198.200:5000/books-ms```

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
1. from ```ms-lifecycle``` directory inside the ```cd``` vm
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
7. Scale service: ```docker service scale books-ms-app=3```

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
Page 299

The two playbooks deployed the familiar Jenkins instance with two nodes

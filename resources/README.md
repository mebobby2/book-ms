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

## Helpful Commands
* ```ll target/scala-2.10``` - list files in a directory. ```ll``` is an alias for ```ls -l```
* ```sudo docker exec -it books-ms bash``` - start a bash session inside the container
* ```docker logs books-ms``` - view the logs of the container
* ```docker rm -f books-ms books-db``` - remove running containers
* ```docker-compose logs``` - check logs of all running containers
* ```docker-compose stop``` - stop all containers
* ```docker-compose rm -f``` - remove all containers (the stop command doesn't remove them)

## Upto
Page 84

The Checklist

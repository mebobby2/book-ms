# Book MS

## Prerequisites
* Git
* Vagrant (virtualbox by dependency)

## Setup
1. Speed up the creation of new VMs. It caches all packages that are being used so that the next time we need them, they are obtained from the local HD instead being downloaded. ```vagrant plugin install vagrant-cachier```

## Development
1. ```vagrant up dev```
2. ```cd /vagrant```
3. Build the container for running tests (done once only): ```sudo docker build -f Dockerfile.test -t 10.100.198.200:5000/books-ms-tests .```
4. Runs the pre-deployment tests and compile the Scala code: ```docker-compose -f docker-compose-dev.yml run --rm tests```

## Tests
1. ```vagrant up dev```
2. ```vargrant ssh dev```
3. ```cd /vagrant```
4. Build the container for running tests (done once only): ```sudo docker build -f Dockerfile.test -t 10.100.198.200:5000/books-ms-tests .```
5. Run front-end tests: ```sudo docker-compose -f docker-compose-dev.yml run feTestsLocal```
6. Run all tests and package JAR: ```sudo docker-compose -f docker-compose-dev.yml run testsLocal```

## Upto
Page 73

The second command run all the pre-deployment tests and compiled the Scala cod

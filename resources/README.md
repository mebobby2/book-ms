# Book MS

## Prerequisites
* Git
* Vagrant (virtualbox by dependency)

## Setup
1. Speed up the creation of new VMs. It caches all packages that are being used so that the next time we need them, they are obtained from the local HD instead being downloaded. ```vagrant plugin install vagrant-cachier```

## Development
1. ```vagrant up dev```

## Tests
1. ```vagrant up dev```
2. ```vargrant ssh dev```
3. ```cd /vagrant```
4. Run front-end tests: ```sudo docker-compose -f docker-compose-dev.yml run feTestsLocal```
5. Run all tests and package JAR: ```sudo docker-compose -f docker-compose-dev.yml run testsLocal```


## Upto
Page 70

Implementation of the Deployment

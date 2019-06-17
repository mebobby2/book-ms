# DevOps 2.0

# Notes
## The DevOps Ideal
CI/CD means more than Jenkins and writing scripts. It relates to almost every aspect of software development. CI/CD cannot be done without making architectural decisions. Similar can be said for tests, configurations, environments, fail-over, and so on. To create a successful implementation of CI/CD, we need to make a lot of changes that, on the first look, do not seem to be directly related. We need to apply some patterns and practices from the very beginning. We have to think about architecture, testing, coupling, packaging, fault tolerance, and many other things. CI/CD requires us to influence almost every aspect of software development.

To be truly proficient with CI/CD, we need to be much more than experts in operations. DevOps movement was a significant improvement that combined traditional operations with advantages that development could bring. But it's not enough. We need to know and influence architecture, testing, development, operations and even customer negotiations if we want to gain all the benefits that CI/CD can bring. We need to go back to the beginning and not only make sure that operations are automated but that the whole system is designed in the way that it can be automated, fast, scalable, fault-tolerant, with zero-downtime, easy to monitor, and so on. We cannot accomplish this by simply automating manual procedures and employing a single do-it-all tool. We need to go much deeper than that and start refactoring the whole system both on the technological as well as the procedural level.

### Continuous Integration, Delivery, and Deployment
eXtreme Programming (XP) bought forth the thinking of *continuous integration* (CI). You should not wait until the last moment to integrate! In the waterfall era, such a thing was not so obvious as today. We implemented a continuous integration pipeline and started checking out every commit, running static analysis, unit and functional tests, packaging, deploying and running integration tests.

Later on, *continuous delivery* (CD) started to take ground, and we would have confidence that every commit that passed the whole pipeline could be deployed to production. We could do even better and not only attest that each build is production ready, but apply *continuous deployment* and deploy every build without waiting for (manual) confirmation from anyone. And the best part of all that was that everything was fully automated.

### Architecture


# Upto
Page 14

Architecture

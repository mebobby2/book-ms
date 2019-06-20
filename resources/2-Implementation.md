# The Implementation Breakthrough: Continuous Deployment, Microservices, and Containers
When *continuous deployment*, *microservices*, and *containers* are combined, new doors open waiting for us to step through. Recent developments in the area of containers and the concept of immutable deployments enable us to overcome many of the problems microservices had before. They, on the other hand, allow us to gain flexibility and speed without which CD is not possible or cost effective.

## Continuous Integration
To understand continuous deployment we should first define its predecessors; continuous integration and continuous delivery.

Integration phase of a project development tended to be one of the most painful stages in software development life-cycle. We would spend weeks, months or even years working in separate teams dedicated to separate applications and services. While it wasn’t hard to periodically verify each of those applications and services in isolation, we all dreaded the moment when team leads would decide that the time has come to integrate them into a unique delivery.

Extreme Programming (XP) and other agile methodologies become familiar, automated testing become frequent, and continuous integration started to take ground. Today we know that the way we developed software back then was wrong.

Continuous integration (CI) usually refers to integrating, building, and testing code within the development environment. It requires developers to integrate code into a shared repository often, in most cases, be done at least a couple of times a day. Getting code to the shared repository is not enough and we need to have a pipeline that, as a minimum, checks out the code and runs all the related tests. The result of the execution of the pipeline can be either red or green.

The continuous integration pipeline should run on every commit or push. Unlike continuous delivery, continuous integration does not have a clearly defined goal of that pipeline. Saying that one application integrates with others does not tell us a lot about its production readiness. We do not know how much more work is required to get to the stage when the code can be delivered to production. All we are truly striving for is the knowledge that a commit did not break any of the existing tests.

To gain maximum benefits, we should write tests in test-driven development (TDD) fashion. Tests are not the only CI prerequisite. One of the most important rules is that when the pipeline fails, fixing the problem has higher priority than any other task. If this action is postponed, next executions of the pipeline will fail as well. People will start ignoring the failure notifications and, slowly, CI process will begin losing its purpose.

The most common flow is the following.
1. Pushing to the code repository
2. Static analysis
3. Pre-deployment testing
4. Packaging and deployment to the test environment
5. Post-deployment testing

### Pushing to the code repository
Developers work on features in separate branches. Once they feel comfortable that their work is stable, the branch they’ve been working on is merged with the mainline (or trunk). More advanced teams may skip feature branches altogether and commit directly to the mainline. The crucial point is that the mainline branch (or trunk) needs to receive commits often (either through merges or direct pushes).

As a minimum, failure should result in some form of a notification sent to the developer that pushed the commit that resulted in a failed pipeline. It should be his responsibility to fix the problem (after all, he knows best how to fix a problem created by him only minutes ago). Try to keep a number of people who receive the failure notification to a minimum. The whole process from detecting a problem until it is fixed should be as fast as possible. The more people are involved, the more administrative work tends to happen and the more time is spent until the fix is committed.

### Static analysis
While benefits of using static analysis are questionable, the effort required to implement it is so small that there is no real reason not to use it.

Static analysis is often the first step in the pipeline for the simple reason that its execution tends to be very fast and in most cases faster than any other step we have in the pipeline.

### Pre-deployment testing
Unlike (optional) static analysis, pre-deployment tests should be mandatory. As a rule of thumb, all types of tests that do not require code to be deployed to a server should be run in this phase.

### Packaging and Deployment to the Test Environment
Once deployment package is created, we can proceed to deploy it to a test environment. Depending on the capacity of the servers you might need to deploy to multiple boxes with, for example, one being dedicated to performance testing and the other for all the rest of tests that require deployment.

### Post-deployment testing
Once deployed to a test environment, we can execute the rest of the tests those that could not be run without deploying the application or a service as well as those that prove that the integration was successful. As a general rule, they include functional, integration and performance tests.

Later on, one of the builds of the pipeline will be elected to be deployed to production. Means and details of additional checks and deployment to production are not part of continuous integration.

## Continuous Delivery and Deployment
The continuous delivery pipeline is in most cases the same as the one we would use for CI. The major difference is in the confidence we have in the process and lack of actions to be taken after the execution of the pipeline. While CI assumes that there are (mostly manual) validations to be performed afterward, successful implementation of the CD pipeline results in packages or artifacts being ready to be deployed to production. In other words, every successful run of the pipeline can be deployed to production, no questions asked. Whether it will be deployed or not depends more on political than technical decisions.

Continuous deployment pipeline goes a step further and automatically deploys to production every build that passed all verifications. There is no human intervention.

We need to pay particular attention to databases (especially when they are relational) and ensure that changes we are making from one release to another are backward compatible and can work with both releases (at least for some time).

While continuous integration welcomes but does not necessarily requires deployed software to be tested in production, continuous delivery and deployment have production (mostly integration) testing as an absolute necessity and, in the case of continuous deployment, part of the fully automated pipeline.

Another very useful technique in the context of continuous deployment is feature toggles. Since every build is deployed to production, we can use them to disable some features temporarily.

Things that take less to run are run first in the pipeline. The same rule should be followed within each phase. If, for example, you have different types of tests within the pre-deployment phase, run those that are faster first. The reason for this quest for speed is time until we get feedback.

## Microservices
Microservices provide us with more freedom to make better decisions, faster development and easier scaling of our services.

Running the whole pipeline for a huge monolithic application is often slow. Same applies to testing, packaging, and deployment. On the other hand, microservices are much faster for the simple reason that they are much smaller. Less code to test, less code to package and less code to deploy.

## Containers
Before containers become common, microservices were painful to deploy. Deploying one microservice is simple but when their number multiplies, things are starting to get complicated. They might use different versions of dependencies, different frameworks, various application servers, and so on. After all, one of the reasons behind microservices is the ability to choose the best tool for the job. One might better off if it’s written in GoLang while the other would be a better fit for NodeJS. Installing and maintaining all that might quickly turn servers into garbage cans.

What was the solution most were applying before? Standardize as much as possible. Everyone should use only JDK7 for the back-end. All front-end should be done with JSP. The common code should be placed in shared libraries. In other words, people tried to solve problems related to microservices deployment applying the same logic they learned during years of development, maintenance, and deployment of monolithic applications. Kill the innovation for the sake of standardization.

Docker made it possible to work with containers without suffering in the process. They are a solution to make our software run reliably and on (almost) any environment.

Traditional deployments would put an artifact into an existing node expecting that everything else is in place; the application server, configuration files, dependencies, and so on. Containers, on the other hand, contain everything our software needs. The result is a set of images stacked into a container that contains everything from binaries, application server and configurations all the way down until runtime dependencies and OS packages.

Containers share the operating system of the physical server and, where appropriate, binaries and libraries. This gain in resource utilization is critical considering that we might have tens or hundreds of them on a single physical server.

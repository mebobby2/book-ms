# System Architecture

## Monolithic
Monolithic applications are developed and deployed as a single unit. We start dividing our applications into layers: presentation layer, business layer, data access layer, and so on. This separation is more logical than physical.

With time, the number of features our application was required to support was increasing and with that comes increased complexity. We might decide to add a layer with a rules engine, API layer, and so on. This results in situations where we might need to develop a simple feature that under different circumstances would require only a few lines of code but, due to the architecture we have, those few lines turn up to be hundreds or even thousands because all layers need to be passed through.

We still needed to test and deploy everything every time there was a change or a release. Testing, especially regression, tends to be a nightmare that in some cases last for months.

Scaling monoliths often mean scaling the entire application thus producing very unbalanced utilization of resources. In that case, we often end up with a monolith replicated across multiple nodes with a load balancer on top. This setup is sub-optimum at best.

## Services Split Horizontally
Service-oriented architecture (SOA) was created as a way to solve problems created by, often tightly coupled, monolithic applications. The approach is based on four main concepts we should implement.

1. Boundaries are explicit
2. Services are autonomous
3. Services share schema and contract but not class
4. Services compatibility is based on policy

However, we continued having the same layers as we had before, but this time physically separated from each other. There is an apparent benefit from this approach in that we can, at least, develop and deploy each layer independently from others. Another improvement is scaling. With the physical separation between what used to be layers, we are allowed to scale better.

In between services we would put ESB that would be in charge of transformation and redirection of requests from one service to another. ESB and similar products are beasts of their own and we often end up with another monolithic application that is as big or even bigger than the one we tried to split. What we needed was to break services by bounded contexts and separate them physically with each running in their own process and with clearly defined communication between them. Thus, microservices were born.

## Microservices
Microservices are an approach to architecture and development of a single application composed of small services.

In a way, microservices use the concepts defined by SOA. Then why are the called differently? SOA implementations went astray. That is especially true with the emergence of ESB products that themselves become big and complex enterprise applications. In many cases, after adopting one of the ESB products, the business went as usual with one more layer sitting on top of what we had before. Microservices movement is, in a way, reaction to misinterpretation of SOA and the intention to go back to where it all started. The main difference between SOA and microservices is that the later should be self-sufficient and deployable independently of each other while SOA tends to be implemented as a monolith.

We often employ some kind of a lightweight proxy server that is in charge of the orchestration of all requests no matter whether they come from outside or from one microservice to another.

### Disadvantages
#### Operational and Deployment Complexity
The primary argument against microservices is increased operational and deployment complexity. This argument is correct, but thanks to relatively new tools it can be mitigated.

#### Remote Process Calls
Another argument for monolithic applications is reduced performance produced by microservices’ remote process calls. How much that loss of performance affects a system depends on case to case basis. The important factor is how we split our system. By creating microservices organized around bounded contexts or functionality like users, shopping cart, products, and so on, it reduces the number of remote process calls but still keep services organization within healthy boundaries. Also, it’s important to note that if calls from one microservice to another are going through a fast internal LAN, the negative impact is relatively small.

### Advantages
#### Scaling
With microservices, we duplicate only those that need scaling. Not only that we can scale what needs to be scaled but we can distribute things better. We can, for example, put a service that has heavy utilization of CPU together with another one that uses a lot of RAM while moving the other CPU demanding service to a different hardware.

#### Innovation
Monoliths are innovation killers. Due to their nature, changing things takes time, and experimentation is perilous since it potentially affects everything.

#### Commitment Term
With microservices that need for a long-term commitment is much smaller. Change the programming language in one microservice and if it turns out to be a good choice, apply it to others. If the experiment failed or is not the optimum, there’s only one small part of the system that needs to be redone.

## Deployment Strategies
Continuous delivery and deployment strategies require us to rethink all aspects of the application lifecycle. That is nowhere more noticeable than at the very beginning when we are faced with architectural choices.

### Mutable Monster Server
Today, the most common way to build and deploy applications is as a mutable monster server. We create a web server that has the whole application and update it every time there is a new release. Changes can be in configuration (properties file, XMLs, DB tables, and so on), code artifacts (JARs, WARs, DLLs, static files, and so on) and database schemas and data. Since we are changing it on every release, it is mutable.

With mutable servers, we cannot know for sure that development, test, and production environments are the same.

Time to restart such a server when it receives a new release can be considerable. During that time server is usually not operational. Downtime that the new release provokes is a loss of money and trust.

Testing is also a problem. No matter how much we tested the release on development and test environments, the first time it will be tried in production is when we deploy it and make it available not only to our testers but also to all of our users.

Moreover, fast rollback of such a server is close to impossible. Since it is mutable, there is no “photo” of the previous version unless we create a snapshot of a whole virtual machine that brings up a whole new set of problems.

### Immutable Server and Reverse Proxy
If we change our architecture to immutable deployments, we gain immediate benefits. Whenever we deploy an image or a container to the production server, we know that it is precisely the same as the one we built and tested.

A reverse proxy can be used to accomplish zero-downtime.

### Immutable Microservices
Reverse proxy gives us zero-downtime and, having two releases up and running allows us to rollback easily. However, since we’re still dealing with one big application, deployment and tests might take a long time to run. That in itself might prevent us from being fast and thus from deploying as often as needed. Microservices to the rescue!

Due to the size of microservices, we often do not need a separate server to deploy the new release in parallel with the old one.

Now we can truly deploy often automatically, be fast with zero-downtime and rollback in case something goes wrong.

## Microservices Best Practices
### Proxy Microservices or API Gateway
Requests often take more time to be invoked than to receive response data. Proxy microservices might help in that case. Their goal is to invoke different microservices and return an aggregated service. They should not contain any logic but only group several responses together and respond with aggregated data to the consumer.

### Reverse Proxy
Never expose microservice API directly. If there isn’t some orchestration, the dependency between the consumer and the microservices becomes so big that it might remove freedom that microservices are supposed to give us. Lightweight servers like nginx, Apache Tomcat, and HAProxy are excellent at performing reverse proxy tasks and can easily be employed with very little overhead.

### Minimalist Approach
Microservices should contain only packages, libraries, and frameworks that they truly need. The smaller they are, the better. That is quite in contrast to the approach used with monolithic applications. While previously we might have used JEE servers like JBoss that packed all the tools that we might or might not need, microservices work best with much more minimalistic solutions. Having hundreds of microservices with each of them having a full JBoss server becomes overkill. Apache Tomcat, for example, is a much better option. I tend to go for even smaller solutions with, for instance, Spray as a very lightweight RESTful API server. Don’t pack what you don’t need.

The same approach should be applied to OS level as well. If we’re deploying microservices as Docker containers, CoreOS might be a better solution than, for example, Red Hat or Ubuntu. It’s free from things we do not need allowing us to obtain better utilization of resources.

### Configuration Management
Deploying many microservices without tools like Puppet, Chef or Ansible (just to name few) quickly becomes a nightmare.

### Cross-Functional Teams
Microservices are done best when the team working on one is multifunctional. A single team should be responsible for it from the start (design) until the finish (deployment and maintenance). In many cases, one team might be in charge of multiple microservices, but multiple teams should not be in charge of one.

### API Versioning
If some change breaks the API format, it should be released as a separate version. In the case of public APIs as well as those used by other internal services, we cannot be sure who is using them and, therefore, must maintain backward compatibility or, at least, give consumers enough time to adapt.

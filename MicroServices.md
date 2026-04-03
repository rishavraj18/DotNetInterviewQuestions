# Microservices:

The microservices architecture style is an approach to develop a single application as a suite of small services, each running in its own process and communication with lightweight mechanisms.

* Set of practices
* Increases developement and deployment speed
* Improves scalability
* Loosely coupled
* Higly cohesive
* Technology Agnostic
* Principles and architecture pattern

## Micro:

* big or small : no universal measure
* does "one" thing
* scope of functionalities
* bounded context
* identify sub-domains

## Service:

* independently deployable component
* interoperability
* message based communication
* SOA (service oriented architecture)
* 

## SDLC:

AGILE or Waterfall

* Requirements
* Plan and design
* Develop
* Test
* Release
* Monitor

## Monolith:

### Designing a Monolith

![DesignMonolith](/img/DesignMonolith.JPG "DesignMonolith")

### Deploying a monolith:

* Single monolith
* Single database
* User Interface
* Expose APIs
* Multiple instances

### Monolith Pros and Cons:

![Mono-Pros-Cons](/img/Mono-Pros-Cons.JPG "Mono-Pros-Cons")


## Design MicroServices:

Microservices are created using most popular domain driven design. Sharing database is anti-pattern in the microservices.

![DesignMicroServices](/img/DesignMicroServices.JPG "DesignMicroServices")

## MicroServices organization:

![Microservices-Team](/img/Microservices-Team.JPG "Microservices-Team")

![Microservices-Codebase](/img/Microservices-Codebase.JPG "Microservices-Codebase")

## MicroServices data store:

* In microservices, need to store data independently to make it loosely coupled.
* Data synchronization has no distributed transactions
* Immediate consistency occurs through eventual consistency
* Capture data change : Event sourcing pattern
* Tools for publishing events : Kafka, RabbitMQ
* Tools to capture data change : Debezium


![Microservices-DataStore](/img/Microservices-DataStore.JPG "Microservices-DataStore")

## MicroServices user interface:

* Unique UI to display data from multiple microservices.
* We need to make user feel they are interacting with single application.
* UI composition design pattern
  - Server side : composing html fragemnet developed by multiple microservices team
  - Client side : browser deals with single UI composing UI fragements

![Microservices-UserInterface](/img/Microservices-UserInterface.JPG "Microservices-UserInterface")

## Services:

* Microservices are developed and deployed independently
* However they interact with each other frequently

### Remote procedure invocation:

* Remote procedure invocation aka Remote procedure call
* It works on Request/Reply principle
* Synchronous / ASynchronous
* REST / SOAP / gRPC

### Messaging:

* Exchange messages/event via a broker/channel.
* When a Microservices(M1) wants to interact with other(M2), it publishes a message to the broker.
* Other microservices(M2) subscribe to that broker and recieves message. If required, it(M2) changes their state.
* ASynchronous message plays significant role to keep things loosely coupled and improves availability.
* Message broker : Kafka / RabbitMQ

### Protocol Format Exchange:

#### 1) Text message : 

* XML / JSON / YAML
* Human readable
* Easy to implement

#### 2) Binary : 

* gRPC
* More compact

#### 3) APIs and Contracts : 

* API exposes a contarct for each type of request : GET / POST / PATCH / DEL
* SOAP, REST, gRPC
* WSDL, Swagger, IDL
* Different devices, different needs, different APIs, different contracts

## Distributed Services:

![ServiceRegistry](/img/ServiceRegistry.JPG "ServiceRegistry")

* When a Microservices(M1, M2, M3, M4) wants to interact with each other, it must be reliable. The microservice based application runs on environment, their network location changes dynamically.
* Service registry : microservices do self registration of their network location on startup. Also de-register on shutdown.
* Client needs to first discover the location of service instance by querying the registry.
* It invoke the microservices
* e.g. Eureka, Zookeeper, Consul

### 1) CORS:

* Cross-origin resource sharing
* Same-origin policy
* Origin : Same protocol, server and port
* Browser restrict cross-orign request
* Additional HTTP headers to access selective resource from different server : Access-Control-Allow-Origin

### 2) Circuit Breaker:

* Service avaialability is crucial in Microservice application. Request can failed due to : Network failure, Heavy load.
* Domino effect : If failure occurs in one microservice, it cascades into another microservices. Later it can leads to failure of entire application.
* To avoid domino effect, circuit breaker was introduced. Circuit breaker is a way to invoke remote service via a Proxy in order to deviate the call if needed.
* For e.g. if the number of consecutive failures crosses a certain threshold, the circuit breaker will stop attempting to invoke remote service and will deviate the calls.
* After the timeout expires, the circuit breaker will start allowing a limited no. of request to pass through. If those request succeed, the circuit breaker resumes normal operation.The circuit breaker slowly attempts to reintroduce traffic.
* e.g. Hystrix, JRugged

### 3) Gateway:

* Each microservices has its own set of graphical components, but at the EOD they must be aggregated into a single application.
* Gateway is the single entry point for all the clients. This allows each client to have a unified interface to all microservices.
* Gateway can handle request in one of two ways: 
  - Some requests are simply routed to the appropriate service.
  - Others can handle cross-cutting concerns like authentication, authorization or determining the location of services via the registry
* A gateway can also be the ideal place to insert API transalation.
* e.g. Zuul, Netty, Finagle

### 4) Security:

* Authentication : username & password & Authorization : rules to allow or not allow.
* SSO(Single sign-on) : authentication protocol - Kerberos, OpenID connect, OAuth 2, SAML
* IAM : OKTA, KeyCloak, Shiro
* Access Token : Once authenticated, an access token securely stores information about a user and is then exchanged between services. Each service needs to make sure the token is valid and takes the user information out of it to verify that a user is authorized e.g. JWT (JSON web token).
* Cookie : You can see the benefit of the gateway as it centralizes user interface calls and access token control.

### 5) Scalability:

* Microservices can be scaled independently depend on their needs.
* Scalability : Vertical and Horizontal.
* Vertical scaling : Adding more power, disk size, RAM to existing machine.
* Horizontal scaling : Adding more machine to replicate services in different machine. Service replication / Clustering.
* Services may scale up and down based on certain metrics.

#### Horizontal scaling:

![ClientLoadBalancing](/img/ClientLoadBalancing.JPG "ClientLoadBalancing")

* Client load balancing
* Several instances
* All instances are registered in service registry
* Client load balancer will pick and choose from registered instances of microservice to route the request. Load balancer decide based on criteria : Round-robin, weight capacity.
* Ribbon, Meraki

### 6) Availability: 

* Probability of system to be operational 
* Available system reports availability in terms of unit / hours / downtime per year.
* SPOF (Single point of failure) is part of the system, if it fails it stops entire system from working.
* SPOF e.g. - 1 Gateway / 1 Broker / 1 Registry / 1 IAM
* To fix SPOF, multiply instances

### 6) Monitoring: 

* It allows us to take pro-active action
* e.g. a microservice is not responding or consuming unexpected resources
* Many moving parts / machine needs to be inspected on failure. Centralized monitoring helps to find the issue quicker and reduce resolution time.
* Monitoring dashboard allows to visualize all source of information. 
* e.g. Kibana, Grafana, Splunk

#### a) Health check:

* Service is running but incapable of handling requests can be checked by healthcheck. 
* e.g. Healthcheck API on each microservice.
* HTTP endpoint returns the health of each microservices and can be pinged by centralized monitoring.
* Healthcheck API : database status, host status, disk space and avaialable memory etc.

#### b) Log Aggregation:

* Understand the behaviour to troubleshoot problem.
* Each microservices write it logs for information, debug msg, warning, errors
* It is NOT feasible to read each log files of each microservice to understand the error.
* Log aggregator stores each service log centralized and from monitoring dashboard admin can analyze the logs to check the error details easily.
* e.g. LogStash, Splunk, PaperTrail

#### c) Exception Tracking:

* Error : Throw an exception and record in centarlized excpetion tracking system
* Investigation and resolved. Also reduce the error wrt time.

#### d) Metrics:

* System slowing down
* Performance issues
* Gather statistics
* Aggregate metrics in centarlized metrics service which provide reporting and alerting
* e.g. DropWizard, Actuator, Prometheus

#### e) Auditing:

* Behaviour of user : Login, logout, visited pages, browsed products
* Record user activity : which part is heeavly used and optimize and scale accordingly

#### f) Rate limiting: 

* 3rd party microservices invokes/access our API, so need to control API usage by setting rate limiting.
* Rate limiting is not new and not entirely related to microservices. It is extensibly used to defend DoS attacks.
* It applies policy to limit traffic coming from specific sources / customer / API / ip address.
* We can limit HTTP request in a given period of time.
* It helps to monetize our APIs as well. for e.g. we can limit our api call to specific condition and if it exceeds, we can charge accordingly.

#### g) Alerting :

* Tons of information
* How to be proactive?
* Fix error when occurs
* Configure threshold
* Trigger alerts

#### h) Distributed Tracing :

* Request span services
* Logging
* Trace entire request from the UI to the database through microservices.
* Chain of calls : provides time of invokation, response time, latency, insights of database operation etc.
* With distributed tracing, A Correlation ID is a unique identifier (often a GUID) assigned to an initial client request and passed through all downstream components, microservices, and logs. It acts as a "glue" that enables tracing, debugging, and performance monitoring across distributed systems, allowing developers to consolidate logs and understand end-to-end request behavior.
* e.g. Dapper, HTrace, Zipkin

### 6) Deployment: 

* Each of our microservices must be independently deployable and scalable.
* It can be deployed either or physical or virtual servers / on-premise or cloud.
* We can deploy each single microservices instance on its own host. This way, each microservice instance is isolated from one another. No possibility of conflicting resources.
* We can also run several instances of different services on a single host. It is a more efficient resource utilization, but you need to make sure the microservices do not end up conflicting.

#### a) Containers:

* Our architecture starts to get a bit crowded. We could get lost in packaging a microservice with its right dependencies such as Operating System and File System.
* One way to simplify the deployment is to package each microservice as a container image, and deploy it as a container.
* It is the container's role to encapsulate the details of the technology used to build a service. 
* Container can also impose the limits on the CPU and memory usage.
* By providing an image that contains all the microservices dependencies, it becomes easy to move it from development to testing and finally to production.
* It also becomes straightforward to scale up and down services by changing the number of container instances.
* Container management system e.g. Docker, Rocket

#### b) Orchestrator:

* Containers are a way of packaging our microservices, but we need to orchestrate all these conatiners by running multiple containers across multiple machines.
* We also need to start the right containers at the right time, figure out how they can talk to each other, handle storage consideration, and deal with failed containers or hardware.
* Manually it's a nightmare. 
* Tools : Kubernetes, Mesos, Docker Swarm, Marathon

#### c) Continuous delivery:

* Even if every piece of architecture is a container that can be orchestrated, we want this entire deployment to be automated.
* Deployment our application should be as cost effective and quick as possible, but most important reliable.
* Continuous delivery is meant to get all the code from our version control, build each microservice, pass all the unit test, acceptance test, performance test, release the code, package it into a container and trigger the orchestrator so it can deploy all the pieces of our architecture.
* The more frequently continuous delivery run, the more robust a deployment into a production can be.
* CD Tools e.g. Jenkins, Asgard, or Aminator.

#### d) Environments:

* Containers, Orchestrator, and Continuous delivery are not just for production, they have to be used in all our environments depending on the complexity or the architecture.
* Environments e.g. Dev, Test, QA/Staging/UAT, Production

#### e) External Configuration:

* Different levels of logging in dev and prod envrionment.
* All database location and credentials will also be different between environments.

### NOTES: 

#### I) IT REACHES TO A POINT WHEN WE NEED TO INTEGRATE ALL THE MICROSERVICES WITH A GATEWAY, MESSAGING BROKER, SERVICE REGISTRY AND SO ON.. AND TEST THE ENTIRE APPLICATION.
#### II) WE ALSO NEED TO TAKE MICROSERVICES VERSIONING INTO ACCOUNT. E.G. TEST V 1.2 IN STAGING ENV HOWEVER WE HAVE V 1.0 IN PRODUCTION. 







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

### CORS:

* Cross-origin resource sharing
* Same-origin policy
* Origin : Same protocol, server and port
* Browser restrict cross-orign request
* Additional HTTP headers to access selective resource from different server : Access-Control-Allow-Origin

### Circuit Breaker:

* Service avaialability is crucial in Microservice application. Request can failed due to : Network failure, Heavy load.
* Domino effect : If failure occurs in one microservice, it cascades into another microservices. Later it can leads to failure of entire application.
* To avoid domino effect, circuit breaker was introduced. Circuit breaker is a way to invoke remote service via a Proxy in order to deviate the call if needed.
* For e.g. if the number of consecutive failures crosses a certain threshold, the circuit breaker will stop attempting to invoke remote service and will deviate the calls.
* After the timeout expires, the circuit breaker will start allowing a limited no. of request to pass through. If those request succeed, the circuit breaker resumes normal operation.The circuit breaker slowly attempts to reintroduce traffic.
* e.g. Hystrix, JRugged

### Gateway:

* Each microservices has its own set of graphical components, but at the EOD they must be aggregated into a single application.
* Gateway is the single entry point for all the clients. This allows each client to have a unified interface to all microservices.
* Gateway can handle request in one of two ways: 
  - Some requests are simply routed to the appropriate service.
  - Others can handle cross-cutting concerns like authentication, authorization or determining the location of services via the registry
* A gateway can also be the ideal place to insert API transalation.
* e.g. Zuul, Netty, Finagle

### Security:

* Authentication : username & password & Authorization : rules to allow or not allow.
* SSO(Single sign-on) : authentication protocol - Kerberos, OpenID connect, OAuth 2, SAML
* IAM : OKTA, KeyCloak, Shiro
* Access Token : Once authenticated, an access token securely stores information about a user and is then exchanged between services. Each service needs to make sure the token is valid and takes the user information out of it to verify that a user is authorized e.g. JWT (JSON web token).
* Cookie : You can see the benefit of the gateway as it centralizes user interface calls and access token control.





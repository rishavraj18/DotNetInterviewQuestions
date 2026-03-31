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






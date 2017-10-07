<h1><p style="text-align: center;">Backend Programming Guide</p></h1>

Table of content:
[TOC levels=2-4]

## Overview
This document introduce technical detail about backend programming. It introduce how to implement each
type of technical components and coding standard. Developers who are new to this project should go through
this document first before doing any backend service development tasks.


## Components For Tasks 

When developer want to create a new backend service for new functions, below types of technical components 
will be touched:

* Controllers - Controllers are used to expose HTTP REST endpoints that can be accessed by other services or frontend
program. It implement the C layer of MVC pattern. Usually, it call Services to conduct transactions or retrieve data.
It may transform the data that is retrieved from Services into a format that can be understood by the caller 
(such as an browser based frontend application). Usually, Controllers accept request or generate response in JSON
 format(except for the scenarios that need to handle binary data. Such as file upload, generate PDF).
* Services - Services are used to implement business logic. It call other Services, Repositories or Daos to manipulate 
data in DB. Transaction control should be done in this layer. 
* Repositories - Spring data is used in this project. Compare with traditional Daos, Spring data can save effort if
developer just need some simple DB access method. For example, Spring data can provide repositories with default 
CRUD method for one entity. Developer do not need to code any implementation. Spring data
also can generate implementation for simple query method for one entity by parsing method names. Developer just need 
to define that method in repositories interface. Spring data repositories only can be used to do DB access for one
entity only.
* Daos - In some scenarios, complicated DB access is needed for some reasons. For example, some query functions may 
have performance issue if "join table" is not used. In these scenarios, Daos will be used. Daos are used to handle the 
scenario that need cross-table or cross-entity DB access.
* Entities - JPA(Implemented with Hibernate) is used for DB access in this project. All DB access should be done through 
JPA entity except for some special scenarios. For example, an legacy store procedure must be reused becauuse it is too 
expensive to rebuild it in java. In this case, developer should call JPA EntityManager to access that store procedure. 
Developer should never by pass JPA(anyway, multiple technologies for one purpose is always not good for application 
maintenance) until something really cannot be done with JPA or it will cause other issues.
Please note that, Services, Repositories, Daos and Entities are belong to the M layer of MVC pattern.
* Service Forms - In some cases, there are many input fields for one transaction. For example, 10 fields need to be 
input to create an order. It is not a good practise to define those 10 fields as parameters for methods in Controllers
and Services. An Service Form should be defined for these 10 fields and use this Service Form as parameters for methods
in Controllers or Services. Basically, Service Forms are values objects for passing values only. It should not include 
any logic. Service Forms are used to contain JSON data that is parsed from body in a PUT/POST HTTP requests as well
 (please see 
[@RequestBody usage](https://docs.spring.io/spring/docs/4.3.10.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#mvc-ann-requestbody))  
. In this case, the fields definition in Service Forms should match the format of JSON from frontend program. So, 
Service Forms are used to defined the interface between frontend and backend programs.
* Views - It is similar to Service Forms, Views are values objects and they are used to defined the interface
between frontend and backend. Different from Service Forms that are used as input for Controllers or Services
, Views are used as output. For example, there is a complicated query function need to retrieve 10 fields from 3 tables.
An View class that include 10 fields. Services should call Repositories/Daos to retrieve data to fill that View object
and return it to Controllers. Controllers will return this View object with annotation 
[@ResponseBody](https://docs.spring.io/spring/docs/4.3.10.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#mvc-ann-responsebody)   
to generate JSON to response frontend program. Basically, Views are used to show data in screens and Service Forms are 
used to submit data. Sometimes, the no of fields shown in screens are more than the no of fields that will be 
submitted in the same screen. For example, in an user editing screen, the role for a user can be choosed. 
The screen need to show the role code and role description. However, when submit button is clicked, only role code 
need to be submitted. In this case, the View class may extend the Service Form class.
* Feign Clients - In micro-service architecture, backend services are deployed into different processes so that cross 
processes communication is needed. Besides the cross processes communication happen in production env, the API testing
(subcutaneous test) also need cross processes communication. The reason is the test case is deployed in an separated process. 
 They communicate with the APIs that are being tested by sending HTTP request(simulate the behaviour of frontend 
 program). In this project, Feign Clients are used for this kind of communication. With Feign Client, developer do not
 need to construct the HTTP request manually which may cause typo mistake easily and is not refactoring friendly.
Developer just need to make sure the methods definitions is exactly the same as the ones in 
Controllers in target process. Then, the Feign Clients can construct proper HTTP request to call the target REST API. 
Besides used as an remote communication channel, Feign Clients also provide load balancing features with 
[Ribbon](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-ribbon).
For simple application, the IP/ports for remote processes can be written in property file. When more and more processes
are involved, it is difficult to use property file to maintain the IP/ports list. To handle this case, Feign Clients 
also can be integrated with 
[Eureka](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-eureka-server) 
for service registration which is widely used in micro-service architecture. 


When an HTTP request is sent to a backend service, this request is handled as below:
![components sequence diagram](components_seq.svg)

**Note**: Because Interceptors are not touched for tasks usually, they are not mentioned in the list above. However,
to illustrate the overall process, it is mentioned in the diagram above.

Below diagram illustrate the relationship between the components mentioned above:
![components class diagram](components_class.svg)

 


 
  
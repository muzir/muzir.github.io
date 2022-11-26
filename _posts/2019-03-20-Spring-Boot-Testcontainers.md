---
layout: post
title:  Introduction to Testcontainers
fullview: false
---

Spring Boot makes it easy to setup and start a microservice project. But in microservice’s world, it is hard to test changes in all architecture.
Because it is hard to setup same environment in local. So each service should responsible their own changes and guarantee that
it won't break any RPC call or business logic  with new changes. Integration tests help in that manner to release the changes in more confident.
So it is important to create an environment which is as much as close to production.

### Introduction

Testcontainers is a java library which rely on junit and docker containers. Any external dependency may included in testcontainer if it has a
docker image.  

Before testcontainers embedded or in memory solutions are exist for databases, message queues, etc. Testcontainers are different from in memory solutions, 
because it provides the ability to run the tests as much as close to production environment. On the other hand nowadays the services are running in containers.
So it is a great ability to run the tests in docker containers too.
  
Assume a web application are using Postgres database. Developer want to test new feature which save some data to some tables
and the query it from some others. Before testcontainers, to test the changes with integration tests developer should configure the postgres in her local machine. 
Ideal case it should be configures in continues integration(CI) environment too. Another approach may use the in memery solutions but in that case
tests will be running different environment that production. Another problem is keeping the local database performance and isolation. Also it takes a lot of time to setup and maintenance. 
Testcontainers are solve all these problems.

### Configuration  

In demo project, integration tests are using Spring tests and testcontainers. To see all dependency, you may take a look to
[build.gradle](https://github.com/muzir/softwareLabs/blob/master/spring-boot-containers/build.gradle){:target="_blank"} file.

```gradle

testImplementation 'org.springframework.boot:spring-boot-starter-test'
testImplementation 'org.testcontainers:postgresql:1.10.6'

```

Created a configuration class for integration tests dependency beans. ```PostgreSQLContainer``` and ```DataSource``` beans are created in
IntegrationTestConfiguration class. Also need to set the ```DataSource``` bean as ```@Primary```, because application context initialize  
a default data source and in ```IntegrationTestConfiguration``` created another one. So it should know which one is injected primarily.

<script src="https://gist.github.com/muzir/1e9015be2be4cf90b7722ef16e7ddbb5.js"></script>

Another important file is ```embedded-postgres-init.sql```. It is a using to initialize the database. In our example 
there are two statements as one of them create hibernate_sequence, which is a requirement for ```AbstractPersistable```, and
another one is just creating the product table. You may also want to run some insert queries before run your tests, so in that
case need to put all statements in that file which is required by test scenarios.

<script src="https://gist.github.com/muzir/0bdb7aae4d487a7dbb78315d8baf0d1f.js"></script>

In the last step for the configuration, defined a base integration tests which is an abstract class and will be extended by 
other integration classes. It is a good practice to run your integration tests with a different profile, so below active profile
is integration, which helps to isolate your integration tests application context from other profiles like development, test or
production.

<script src="https://gist.github.com/muzir/4ea085090b15c6a7e419fe4a801dadd4.js"></script>

### How to write a test

After configure the configuration and base classes then below is a simple example, an integration junit test which is 
using testcontainers. In below scenario; just save a product via ```productRepository.save(product)``` and then query it from database 
with ```productRepository.findByName(productName)```, then check the product names.

<script src="https://gist.github.com/muzir/2def1aca0aa172e919f455ac8d841699.js"></script>


### Notes

In this example, business logic keep it as simple, because it just focus how testcontainers is configured for integration 
tests. After configuration you may want to create more complex test scenarios with some other external storage or components too.

Also before run your tests check the docker first. Docker should be running at your machine otherwise you may see the error below; 

```Caused by: java.lang.IllegalStateException: Could not find a valid Docker environment. Please see logs and check configuration```

### Result


You can find the all project [on Github](https://github.com/muzir/softwareLabs/tree/master/spring-boot-containers){:target="_blank"}


### References

[https://www.testcontainers.org/](https://www.testcontainers.org/){:target="_blank"}


Happy coding :) 


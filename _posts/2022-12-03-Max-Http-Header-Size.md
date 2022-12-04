---
layout: post
title: Http Header Size - Difference Between Tomcat and Jetty Application Servers  
fullview: false
---

 Few days ago I have an interesting case to figure out in my daily job. Some of the requests have failed because of [HTTP 400](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/400) which cause some frontend components 
failures for our customers. I've started to investigate however it wasn't a common failure, it was happening only for few customers. And the total failures are really low
when comparing the with the all requests to that endpoint. Another strange thing is some of these requests failure in different steps in our system. Some reach another service and 
 fail as [HTTP 413](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/413). Also HTTP 400 is a such a common response code doesn't give much idea, it was a bad request however why it is 
bad request server logs don't provide any details about it. 

### Deeper Investigation

I started to investigate more with all call chains and start to compare the successful requests and failed ones. Thanks for our great observability tool I realised a pattern
that all failed requests are greater than 7,5 KBs. So something should be wrong in somewhere in the call chain regarding with request size. After few discussion with platform team
we realised it is at the one of the common service which we use as a gateway. At the same time I was checking the Internet to found out any similar cases and potential solution. 
I come up these two useful resources([stackoverflow](https://stackoverflow.com/questions/686217/maximum-on-http-header-values), [baeldung](https://www.baeldung.com/spring-boot-max-http-header-size) which are similar to my problem.

### Mystery Resolving

These two links give clear idea regarding with that problem and how to solve it. Now I decided to proof it myself based what I understand from these resources. Stackoverflow 
answers assert that servers and application servers have different default limits for request's payload and header sizes. And baeldung provide how to chnage that attribute 
to tune your Spring Boot application services. 

### Testing Max Header Size 
 I create below scenarious in one of my Spring Boot 2 service. Here is the application.yml;

<script src="https://gist.github.com/muzir/27d01811907b5ea17a301bf718fc909d.js"></script>




### How to write a test with ```TestRestTemplate```

### Notes

Because  ```@SpringBootTest``` needs all application contexts, these tests spend more time than unit tests. It can cause some 
delay to your build pipeline, they should be configure wisely. Also it is a good practice to configure it with a different profile.
 

### Result

So Spring starts the server and make the tests as much as close to production environment.

You can find the all project [on Github](https://github.com/muzir/softwareLabs/tree/master/spring-boot-integration-test)


### References

[https://spring.io/guides/gs/testing-web/](https://spring.io/guides/gs/testing-web/)

Happy coding :) 


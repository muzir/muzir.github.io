---
layout: post
title: Http Header Size - Difference Between Tomcat and Jetty Application Servers  
fullview: false
---

Few days ago I had an interesting case to figure out in my daily job. Some of the requests have failed because of [HTTP 400](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/400) which cause some frontend components
failures for our customers. I've started to investigate however it wasn't a common failure, it was happening only for a few customers. And the total failures are really low
when comparing them with all requests to that endpoint. Another strange thing is some of these requests fail in different steps in our system. Some reach another service and
fail as [HTTP 413](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/413). Also HTTP 400 is a such a common response code doesn't give much idea, it was a bad request however why it is
Bad request server logs don't provide any details about it.

### Deeper Investigation

I started to investigate more with all call chains and started to compare the successful requests and failed ones. Thanks for our great observability tool I realized a pattern
that all failed requests are greater than 7,5 KBs. So something should be wrong somewhere in the call chain regarding request size. After few discussion with platform team
we realized it is one of the common services which we use as a gateway. At the same time I was checking the Internet to find out any similar cases and potential solutions.
I come up these two useful resources([stackoverflow](https://stackoverflow.com/questions/686217/maximum-on-http-header-values), [baeldung](https://www.baeldung.com/spring-boot-max-http-header-size) which are similar to my problem.

### Mystery Resolving

These two links give clear ideas regarding that problem and how to solve it. Now I decided to prove it myself based on what I understand from these resources. Stackoverflow
answers assert that servers and application servers have different default limits for request's payload and header sizes. And baeldung provide how to change that attribute
to tune your Spring Boot application services.

### Testing Max Header Size 

I created the scenario below in one of my Spring Boot 2 service. Here is the application.yml;

<script src="https://gist.github.com/muzir/27d01811907b5ea17a301bf718fc909d.js"></script>

#### Testing with Tomcat

Then I created the integration port test with the default Spring Boot 2 embedded Tomcat application server.

<script src="https://gist.github.com/muzir/639eac63ce6bc1852fa470e3778b0e21.js"></script>

As you see in the assert statement, it checks the response code as `HttpStatus.BAD_REQUEST` which is HTTP 400 response. So in Tomcat when
header size is greater than `max-http-header-size` it returns 400. In that scenario server  `max-http-header-size` set it as `10 KB`
and in the test I sent a request with `AUTHORIZATION` header `11 KB`. Let's see the response on Postman too.

I'm running the service with default profile `max-http-header-size` set `10 KB` and send the request with `11 KB` `AUTHORIZATION` header.

![postman_http_400_tomcat.png](/img/posts/max_http_header_size/postman_http_400_tomcat.png)

#### Testing with Jetty

Then I made a small change, changed the application server [from Tomcat to Jetty with that PR](https://github.com/muzir/softwareLabs/pull/324/files).
As you see in that PR there is also a change in the response code check, now it is `HttpStatus.REQUEST_HEADER_FIELDS_TOO_LARGE` which is [HTTP 431](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/431).
Let's run it with Postman too.

![postman_http_431_jetty.png](/img/posts/max_http_header_size/postman_http_431_jetty.png)

### Notes

Last tip, if you need to write integration port test against a server which has authentication you don't have to pass/disable
that authentication to test this kind of server properties(`max-http-header-size`), you can create a controller under your `test/java` folder
then you can use that endpoint for your tests.


### Result

Servers and application servers have their own `max-http-header-size` limit and the best option to write integration tests against them to
understand all these differences.

You can find the all project [on Github](https://github.com/muzir/softwareLabs/tree/master/spring-boot-integration-test)


### References

[Stackoverflow maximum-on-http-header-values](https://stackoverflow.com/questions/686217/maximum-on-http-header-values)

[Baeldung spring-boot-max-http-header-size](https://www.baeldung.com/spring-boot-max-http-header-size)

Happy coding :) 



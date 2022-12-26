---
layout: post
title: Spring Boot 3.0 Migration
background: '/img/posts/background_spring_boot_3.jpeg'
fullview: false
---

Spring has released [Spring Boot 3 around a month ago](https://spring.io/blog/2022/11/24/spring-boot-3-0-goes-ga). Because it is a major release, there are huge/breaking changes like required Java and Gradle versions upgraded.
In this post I list these requirements and share my experience when migrating a few of my projects from Spring Boot 2 to Spring Boot 3.

### What has changed? 

Spring has created [a migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#upgrade-to-spring-boot-3) which I refer to in this post a few times.
Besides that I share my own experience when [upgrading few of my own Spring projects](https://github.com/muzir/softwareLabs). Let's list the new requirements for Spring Boot 3;

- A Java 17 baseline.
- Support for Jakarta EE 10 with an EE 9 baseline, which means `javax.*` library imports have changed with `jakarta.*`.
- For Gradle users, Spring Boot’s Gradle plugin requires Gradle 7.x (7.5 or later).


### Java Upgrade

Spring Boot 3 requires Java 17, so locally you need to install Java 17 and set it as a default in your IDE. I use Jetbrains IDEA, so need to change;
Under IntelliJ => Preferences => Build, execution, deployment => Gradle, I had to set the Gradle JVM to: Project SDK (17). I use CircleCI as a continuous integration 
environment. I also need to update the base image which supports Java 11. Now I have [a new image](https://discuss.circleci.com/t/ubuntu-linux-vm-machine-images-2022-april-q2-update/43749) which supports Java 17.


### Gradle Upgrade 

I use gradle wrapper in my projects. So I increase the gradle version to `7.6` in below configuration;

```
wrapper{
    gradleVersion = '7.6'
}
```

Then I run the `gradle wrapper` command which downloads Gradle 7.6 for my project. 

### Spring Upgrade

To upgrade the project from Spring Boot 2 to Spring Boot 3 first I upgraded to the latest Spring Boot 2 version which is `2.7.7` when I upgraded my project. 
After I see that it works with the latest Spring Boot 2 version then I upgraded to Spring Boot 3 with below updates;

```
plugins {
    id 'org.springframework.boot' version '3.0.1'
}
```

After that I run `gradle build` to build and run the integration test in [my project](https://github.com/muzir/softwareLabs/tree/master/spring-boot-containers). My project
didn't compile because of `Jakarta EE` changes. I list the some of the imports that I have changed;

```
- import javax.validation.Valid; -> import jakarta.validation.Valid;
- import javax.validation.constraints.NotNull; -> import jakarta.validation.constraints.NotNull;
- import javax.annotation.PostConstruct; -> import jakarta.annotation.PostConstruct;
- mport javax.annotation.PreDestroy; -> import jakarta.annotation.PreDestroy;
```

After fixing the compile time issues still application context couldn't be up because of `application.yml` changes. Based on the [migration document](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#configuration-properties-migration)

> With Spring Boot 3.0, a few configuration properties were renamed/removed and developers need to update their application.properties/application.yml accordingly. 
> To help you with that, Spring Boot provides a spring-boot-properties-migrator module. Once added as a dependency to your project, 
> this will not only analyze your application’s environment and print diagnostics at startup, but also temporarily migrate properties at runtime for you.

So I had added the migration library to `build.gradle` as follows;

```gradle
runtimeOnly("org.springframework.boot:spring-boot-properties-migrator")
```

At that time I was writing that post there is a typo in that command in the [original migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#configuration-properties-migration)
Gradle doesn't have a ``runtime`` as a dependency configuration, it should be `runtimeOnly`. After adding that library it shows the issues on the application logs.
In my case I need to replace `spring.profiles.active` to ``spring.config.activate.on-profile``.  

### Hibernate Upgrade

The last note is from Hibernate, however I don't give much insight about it because I decided to remove my Spring Data JPA usage when I'm upgrading my projects.
In general I'm not a fan of Hibernate and it seems over complicated to configure the Id generator in the new version. I found [that post](https://thorben-janssen.com/sequence-naming-strategies-in-hibernate-6/) which gives all the details
regarding the `Hibernate 6` upgrade which needs to be done with Spring Boot 3.

### Notes

[Here is the PR](https://github.com/muzir/softwareLabs/pull/333) I have done for all the changes explained in this post.

### Result

Spring released a major upgrade which makes the platform more stabilized with [Jakarta EE changes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#jakarta-ee).   

### References

[Spring Boot 3 project announcement](https://spring.io/blog/2022/11/24/spring-boot-3-0-goes-ga)

[Spring Boot 3 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)

[Gradle Spring Boot Plugin Page](https://docs.spring.io/spring-boot/docs/3.0.0-SNAPSHOT/gradle-plugin/reference/htmlsingle/)

[Sequence Naming Strategies In Hibernate 6](https://thorben-janssen.com/sequence-naming-strategies-in-hibernate-6/)

[Background picture reference](https://www.jrebel.com/blog/what-expect-spring-boot-3)

Happy coding :) 



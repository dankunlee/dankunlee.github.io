---
title: "Building a simple web forum 1: Intro/Setup"
date: 2020-09-11T15:34:30-04:00
categories:
  - Tutorial
tags:
  - Java
  - Spring Boot
  - JPA
  - MySQL
classes: wide
toc: true
---

# Intro

This post is to write how I built a simple web forum back-end API using Spring Boot for fun.

As everything here is hosted on my local machine, you may experience some troubles following the tutorial. 

# Getting started with Spring Initializer 

Spring Initializer is useful for creating a Boot project.   
(Read this <a href="https://spring.io/guides/gs/spring-boot/#scratch">link</a> as this is a good resource to get started.)

There are two main build automation systems for Java projects, Maven and Gradle.

I chose to go with Gradle as it seemed easier to read.

![image](/assets/images/tutorial1/springBootInitializer.png)  

When you create a project, the structure of the code will be like below:  
![image](/assets/images/tutorial1/initialFileStructure.png)  

_build.gradle_ file contains all the dependencies that I included from the initializer.  
   
Because we haven't configured for JPA yet, let's comment it out as it will only run into errors. 

```
plugins {
	id 'org.springframework.boot' version '2.3.3.RELEASE'
	id 'io.spring.dependency-management' version '1.0.10.RELEASE'
	id 'java'
}

group = 'com.dankunlee'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
	mavenCentral()
}

dependencies {
  //	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	runtimeOnly 'mysql:mysql-connector-java'
	testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}
}

test {
	useJUnitPlatform()
}
```

## Running the application

The main function of _ForumApplication_ will look like this. 

```java
package com.dankunlee.forum;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ForumApplication {
	public static void main(String[] args) {
		SpringApplication.run(ForumApplication.class, args);
	}
}
```

When you run the application and go to "localhost:8080", you will first see "_This application has no explicit mapping for /error_". 

This is because there is no mapping to "localhost:8080/".

Create a package called "controller" and let's write a sample controller. 

I'll write what a controller does in a bit. 

```java
package com.dankunlee.forum.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class sampleController {
    @GetMapping("/")
    public String helloWorld() {
        return "hello world!";
    }
}
```
This will be our first endpoint of the website. 

Now if you goto "localhost:8080", you will see a hello world printed. 

![image](/assets/images/tutorial1/helloWorld.png)  

# Connect JPA to DB

JPA(Java Persistance API) is a Jakarta EE application programming interface specification that describes the management of relational data in Java applications.

We'll be using Spring-data-jpa (Spring Boot's object relation mapping library) to interact with database system indirectly.  

You can also use persistance frameworks that maps SQL commands directly to databases such as _mybatis_, but I prefer to use ORM as it allows more object-oriented model for playing with databases.

Now let's uncomment the jpa dependency from _build.gradle_. 

```
dependencies {
  	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	runtimeOnly 'mysql:mysql-connector-java'
	testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}
}
```

## Configuration: application.yml

We need to configure general settings for this application (ex. to which DB are we connecting). 

From "src -> resources -> application.properties", we can configure these. 

Because I prefer YAML format, I converted the file to _yml_ type and configured the settings. 

Keep in mind that I'm connecting to my local MySQL database system. 

```
server:
  port: 8080

spring:
  # Database (MySQL) Configs
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/pear?serverTimezone=GMT
    username: root
    password: password

  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: update # create
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
      use-new-id-generator-mappings: false
    show-sql: true
    properties:
      hibernate.enable_lazy_load_no_trans: true
      hibernate.format_sql: true
```

Spring framework will read configurations from _application.yml_ and automatically apply those. 

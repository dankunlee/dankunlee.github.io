---
title: "Building a simple web forum 1"
date: 2020-09-11T15:34:30-04:00
categories:
  - Tutorial
tags:
  - Java
  - Spring Boot
  - JPA
classes: wide
toc: true
---

**Intro**

This post is to write how I built a simple web forum back-end API using Spring Boot for fun.

The main build automation systems for Java projects are Maven and Gradle.  
I chose to go with Gradle as it seemed easier to read.

**Getting started with Spring Initializer**

I created a Spring Boot project using Spring Initializer as shown below. 
//image: springBootInitializer.png

This <a href="https://spring.io/guides/gs/spring-boot/#scratch">link</a> is a good resource I referred to a lot. 

The structure of the code starts as shown. 
//image: initalfilestructure

build.gradle file contains all the dependencies that I included. 
Explain dependencies


```java
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

Because we haven't configured jpa yet, let's comment it out as it will only run into an error. 

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

When you run the application and go to localhost:8080, you will first see 
This application has no explicit mapping for /error. 
This is because a mapping to "localhost:8080/" does not exist. 

Create a package called "controller" and let's write a sample controller. 
What is a controller
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

Now if you goto localhost:8080, you will see a hello world printed. 
//image: helloworld


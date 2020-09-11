---
title: "Building a simple web forum 1"
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

This <a href="https://spring.io/guides/gs/spring-boot/#scratch">link</a> is a good resource to get started. 

There are two main build automation systems for Java projects, Maven and Gradle.

I chose to go with Gradle as it seemed easier to read.

(/assets/images/tutorial1/springBootInitializer.png)

When you create a project, the structure of the code will be like below:
//image: initalfilestructure

_build.gradle_ file contains all the dependencies that I included from the initializer. 
Explain dependencies
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

Now if you goto "localhost:8080", you will see a hello world printed. 

//image: helloworld

# Java Persistance API

JPA is a Jakarta EE application programming interface specification that describes the management of relational data in Java applications.

We'll be using Spring-data-jpa (Spring Boot's object relation mapping library) to interact with database system indirectly.  

You can also use persistance frameworks that maps SQL commands directly to databases such as _mybatis_, 

but I prefer to use ORM as it allows more object-oriented model for playing with databases.

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

## Connect JPA to DB
We need to configure general settings for this application (ex. how we are going to connect to a DB). 

From "src -> resources -> application.properties", we can configure these. 

Because I prefer YAML format, I converted the file to _yml_ type and made some changes. 

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

  servlet:
    multipart:
      enabled: true
      file-size-threshold: 2KB
      max-file-size: 200MB
      max-request-size: 200MB
```

Spring framework will read configurations from _application.yml_ and automatically apply those. 

## How does Spring JPA work?

Basic Structure: Controller -> Repository -> Entity -> DB

## Entity

Now let's make an _entity_ package which will contain all the necessary DB entities. 

Below is a _post_ entity for storing information about posts. 

```java
package com.dankunlee.forumapp.entity;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.validation.constraints.NotNull;
import java.util.Objects;

@Entity
public class Post extends Auditing{
    @Id
    @GeneratedValue
    @Column(name = "post_id")
    private long id;

    @NotNull // for validating the input
    @Column(nullable = false) // for setting the key NOT NULL
    private String title;

    private String content;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, title, content);
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || obj.getClass() != this.getClass()) return false;
        Post post = (Post) obj;
        return Objects.equals(post.id, id) &&
                Objects.equals(post.title, title) &&
                Objects.equals(post.content, content);
    }
}
```

## Repository

The next step is to create _repository_ package and interface for . 

Think of repository objects as DAO (Data Access Object)  
--> runs queries for creating, reading, updating and deleting entries from DBs.

```java
package com.dankunlee.forumapp.repository;

import com.dankunlee.forumapp.entity.Post;
import org.springframework.data.jpa.repository.JpaRepository;

public interface PostRepository extends JpaRepository<Post, Long> {

}
```

By extending _JpaRepository_ interface, you can now use this interface for retrieving data. 

For example, _findAll()_ method will retrieve all "post"s from the DB and _findById()_ will retrieve a post with the specified id.

See _Controller_ for more details.   

## Controller

To use above repository, you need a controller that returns the outputs to endpoints.

Create _controller_ package and postController. 

```java
package com.dankunlee.forumapp.controller;

import com.dankunlee.forumapp.entity.Post;
import com.dankunlee.forumapp.repository.PostRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.List;
import java.util.Optional;

@RestController
public class PostController {
    @Autowired
    private PostRepository postRepository;

    @GetMapping("/")
    public String hello() {
        return "API";
    }

    @PutMapping("/api/post")
    public Post newPost(@Valid @RequestBody Post newPost) {
        // creates a new post in the db
        Post createdPost = postRepository.save(newPost);
        return createdPost;
    }

    @GetMapping("/api/post")
    public List<Post> getAllPost() {
        // retrieves all posts from the db
        return postRepository.findAll();
    }

    @GetMapping("/api/post/{id}")
    public Post getPost(@PathVariable String id) {
        // retrieves a post with specified id
        Long postID = Long.parseLong(id);
        Optional<Post> postOptional = postRepository.findById(postID);
        Post post = postOptional.get();
        return post;
    }

    @PostMapping("/api/post/{id}")
    public Post updatePost(@PathVariable String id, @Valid @RequestBody Post updatedPost) {
        // modifies(replaces) the post with the provided new post 
        Long postID = Long.parseLong(id);
        Optional<Post> postOptional = postRepository.findById(postID);

        Post post = postOptional.get();
        post.setTitle(updatedPost.getTitle());
        post.setContent(updatedPost.getContent());
        postRepository.save(post);

        return postRepository.findById(postID).get();
    }

    @DeleteMapping("/api/post/{id}")
    public String deletePost(@PathVariable String id) {
        // deletes the post with specified id
        Long postID = Long.parseLong(id);
        postRepository.deleteById(postID);
        return "Deleted";
    }
}
```

Following annotations are used to implement HTTP request methods. 

@PutMapping = PUT  
@GetMapping = GET  
@PostMapping = POST  
@DeleteMapping = DELETE   

@PathVariable annotation allows each methods to take an input id from specified URL.  
@RequestBody annotation allows the methods to take the body of HTTP request.





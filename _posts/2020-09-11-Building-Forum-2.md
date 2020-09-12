---
title: "Building a simple web forum 2: MVC"
date: 2020-09-11T16:34:30-04:00
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

# What is MVC?

For this application, we are going to implement Model-View-Controller Architecture. 

MVC is one of the design patterns that is widely used in web applications. 

If a user operates a controller, the controller will retrieve data from model. 

Then the data will be displayed through View to the user. 

Below shows the MVC design. 

![image](/assets/images/tutorial1/MVC.png)  

**Entity**  
Entity is a class that will be linked with DB’s tables

**DAO**  
Data Access Object directly interacts with the persistence layer of DB (CRUD DB data). 
We use Repositories to implement this. 

**DTO**  
When data travels to Service or Controller, they transform into Data Transfer Object. 
DTO is just an object that holds data. It consists only of getters and setters (setters are rarely used). 

**Service**  
Service handles buisness logics. If you want to pass the data directly to the View (user) then there is no need for Service. 

**Controller**  
Controller handles view and mapping according to http requests from end-users.  
(ex. gets a request from a user and returns a DTO object)

For example, if a user wants to see an image:  
The application will access DB through DAO, gets the image path data in DTO format, then Service will load the image from the path and sends the image to the user  through Controller 

# Post

Now let's apply MVC design and create the most basic functionality of internet forum: Posting

## Entity

Make an _entity_ package which will link to DB table. 

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
When you run the application, you will see that the DB has created _Post_ table. 

## Repository

The next step is to create _repository_ package and repository interface for post. 

As metioned earlier, repository objects are DAOs that run queries for creating, reading, updating and deleting entries from DBs.

```java
package com.dankunlee.forumapp.repository;

import com.dankunlee.forumapp.entity.Post;
import org.springframework.data.jpa.repository.JpaRepository;

public interface PostRepository extends JpaRepository<Post, Long> {

}
```

By extending _JpaRepository_ interface, you can now use this interface for accessing DB data. 

For example, _findAll()_ method will retrieve all "post"s from the DB and _findById()_ will retrieve a post with the specified id.

See _Controller_ for more details.   

## Controller

To use above repository, you need a controller that will return the data to endpoints.

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

Here, we do not require Serivce for post as we just pass the data(posts) to users. 

You can test what we have so far using Postman. 

![image](/assets/images/tutorial1/postman_post.png) 

# Handling Exceptions

Now let's handle exceptions that might occur when invalid inputs are given. 

Spring Boot provides a convenient way of handling all exceptions acrros the application. 

You can use _@ExceptionHandler_ annotation for error handling and _@ControllerAdvice_ annotation for applying them acrross the whole application.

Here, I'm handling 3 types of exceptions. 

```java
package com.dankunlee.forumapp.controller;

import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.NoSuchElementException;

class Error {
    String errorMsg;
    int errorNumb;

    public Error(String msg, int numb) {
        this.errorMsg = msg;
        this.errorNumb = numb;
    }

    public String getErrorMsg() {
        return errorMsg;
    }

    public void setErrorMsg(String errorMsg) {
        this.errorMsg = errorMsg;
    }

    public int getErrorNumb() {
        return errorNumb;
    }

    public void setErrorNumb(int errorNumb) {
        this.errorNumb = errorNumb;
    }
}

@ControllerAdvice // allows this class to handle all exceptions globally
public class ExceptionController {
    public static final int ITEM_DOES_NOT_EXIST = -1;
    public static final int INVALID_PATH = -2;
    public static final int INVALID_PARAMETER = -3;

    @ResponseBody // a method return value should be bound to the web response body (json format)
    @ExceptionHandler(NoSuchElementException.class)
    public Error noItemErrorHandler(NoSuchElementException e) {
        return new Error("Item Does Not Exist", ITEM_DOES_NOT_EXIST);
    }

    @ResponseBody
    @ExceptionHandler(NumberFormatException.class)
    public Error noItemErrorHandler(NumberFormatException e) {
        return new Error("Path in Invalid Format", INVALID_PATH);
    }

    @ResponseBody
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Error invalidMethodHandler(MethodArgumentNotValidException e) {
        return new Error("Invalid Parameter is Given", INVALID_PARAMETER);
    }
}
```

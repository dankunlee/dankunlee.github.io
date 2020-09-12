---
title: "Building a simple web forum 4: Data Auditing"
date: 2020-09-12T00:34:30-04:00
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

# What is Data Auditing?

Database auditing means tracking and logging events related to entities. 

We need auduiting to maintain and keep track of who or when the data has been entered. 

We'll be using Spring JPA's auditing functionality to record who and when a post or a comment has been created/modified. 

Let's create Auditing entity to do this. 

```java
package com.dankunlee.forumapp.entity;

import org.springframework.data.annotation.CreatedBy;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedBy;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.Column;
import javax.persistence.EntityListeners;
import javax.persistence.MappedSuperclass;
import java.time.LocalDateTime;

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public class Auditing {

    @CreatedDate
    @Column(updatable = false)
    public LocalDateTime createdDate;

    @LastModifiedDate
    public LocalDateTime lastModifiedDate;

    @CreatedBy
    @Column(updatable = false)
    public String createdBy;

    @LastModifiedBy
    public String lastModifiedBy;

    public LocalDateTime getCreatedDate() {
        return createdDate;
    }

    public LocalDateTime getLastModifiedDate() {
        return lastModifiedDate;
    }

    public String getCreatedBy() {
        return createdBy;
    }

    public String getLastModifiedBy() {
        return lastModifiedBy;
    }
}
```
---
Using @CreatedDate, @LastModifiedDate, @CreatedBy, @LastModifiedBy annotations, posts and comments auditing is now enabled. 

@MappedSuperclass annotation will let other entities to inherit this entity. 

If you refer back to Post and Comment entities, they are inheriting Auditing entity as a super class. 

```java
public class Post extends Auditing { ... }

public class Comment extends Auditing { ... }
```

---
Go back to WebConfigurer and add auditorProvider with @EnableJpaAuditing to enable JPA auditing. 

```java
package com.dankunlee.forumapp.config;

import com.dankunlee.forumapp.interceptor.CommentAuthorizationInterceptor;
import com.dankunlee.forumapp.interceptor.LogInInterceptor;
import com.dankunlee.forumapp.interceptor.PostAuthorizationInterceptor;
import com.dankunlee.forumapp.utils.Session;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.Optional;

@Configuration
@EnableJpaAuditing
public class WebConfigurer implements WebMvcConfigurer {
    @Autowired
    LogInInterceptor logInInterceptor;

    @Autowired
    PostAuthorizationInterceptor postAuthorizationInterceptor;

    @Autowired
    CommentAuthorizationInterceptor commentAuthorizationInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Applies interceptors to the following urls
        /*
        1. ? matches one character
        2. * matches zero or more characters
        3. ** matches zero or more 'directories' in a path
        */

        registry.addInterceptor(logInInterceptor)
                .excludePathPatterns("/api/post/page/**")
                .addPathPatterns("/api/post/**")
                .addPathPatterns("/api/post/**/comment/**");

        registry.addInterceptor(postAuthorizationInterceptor)
                .excludePathPatterns("/api/post/page")
                .excludePathPatterns("/api/post/**/comment/**")
                .addPathPatterns("/api/post/**");

        registry.addInterceptor(commentAuthorizationInterceptor)
                .addPathPatterns("/api/post/**/comment/**");
    }

    @Bean
    public AuditorAware<String> auditorProvider() {
        // If a bean of type AuditorAware is exposed to the ApplicationContext,
        // the auditing infrastructure will pick it up automatically
        // and use it to determine the current user to be set on domain types
        return () -> {
            ServletRequestAttributes servletRequestAttributes = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            String user = (String) servletRequestAttributes.getRequest().getSession().getAttribute(Session.SESSION_ID);
            if (user != null) return Optional.of(user); // returns the username of current session
            else return Optional.of("Anonymous"); // should never reach here bc of login interceptor
        };
    }
}
```

We can now see by who and when the post has been created. 
![image](/assets/images/tutorial1/postman_post2.png)  

---
title: "Building a simple web forum 3: Sign Up/Sign In"
date: 2020-09-11T17:34:30-04:00
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

# User Functionality

Steps for signing up is simple:

1. The user will transmit user information including username, password and e-mail. 
2. The application will check for any duplication and add the information to its DB. 
3. All passwords must be encrypted before they are stored. 

However, in order to sign up, User funcationality must be enabled first. 

---
Let's create User entity. 

```java
package com.dankunlee.forumapp.entity;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.validation.constraints.NotNull;
import java.util.Objects;

@Entity
public class User {
    @Id
    @GeneratedValue
    @Column(name = "user_id")
    private long id;

    @NotNull
    @Column(nullable = false)
    private String username;

    @NotNull
    @Column(nullable = false)
    private String password;

    @NotNull
    @Column(nullable = false)
    private String email;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, password, email);
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        User user = (User) obj;
        return Objects.equals(id, user.id) &&
                Objects.equals(username, user.username) &&
                Objects.equals(password, user.password) &&
                Objects.equals(email, user.email);
    }
}
```
---
Then, User repository:

```java
package com.dankunlee.forumapp.repository;

import com.dankunlee.forumapp.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, Long> {
    public User findByUsername(String username);
    public User findByEmail(String email);
    public User findByUsernameAndPassword(String username, String password);
}
```
Note that Spring JPA allows custom queries by following its naming convention.  
(ex. findBy****(String ****);)

---
Now create DTO for User. 

The reason we have DTO for User is because when users try to log in, they are sending their HTTP session through requests. 

This requires more secure way of sending data to services or controllers and sending data through DTO is more secure than sending User entitiy itselft.  

```java
package com.dankunlee.forumapp.dto;

public class UserDTO {
    // Separate DTO class for User
    // Only used when containing http session information of User
    // Sends data to service or controllers (more secure than directly using User entitiy)
    // Used in LogInController
    private String username;
    private String password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

# Password Encryption

Before we create the controller for sign up, we need to be able to encrypt given password. 

Below Password class has 3 methods for encrypting password (all using _SHA-256_). 

```java
package com.dankunlee.forumapp.utils;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.NoSuchProviderException;
import java.security.SecureRandom;

public class Password {
    public static final String randomSalt = "RanDomSalT";

    public static String encode(String password) {
        String encryptedPassword = null;

        try{
            MessageDigest messageDigest = MessageDigest.getInstance("SHA-256");
            byte[] bytes = messageDigest.digest(password.getBytes());

            StringBuilder stringBuilder = new StringBuilder();
            for (int i = 0; i < bytes.length; i++)
                stringBuilder.append(Integer.toString((bytes[i] & 0xff) + 0x100, 16).substring(1));

            encryptedPassword = stringBuilder.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }

        return encryptedPassword;
    }

    public static String encode(String password, String salt) {
        String encryptedPassword = null;

        try{
            MessageDigest messageDigest = MessageDigest.getInstance("SHA-256");
            messageDigest.update(salt.getBytes());
            byte[] bytes = messageDigest.digest(password.getBytes());

            StringBuilder stringBuilder = new StringBuilder();
            for (int i = 0; i < bytes.length; i++)
                stringBuilder.append(Integer.toString((bytes[i] & 0xff) + 0x100, 16).substring(1));

            encryptedPassword = stringBuilder.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }

        return encryptedPassword;
    }

    public static String encode(String password, byte[] salt) {
        String encryptedPassword = null;

        try{
            MessageDigest messageDigest = MessageDigest.getInstance("SHA-256");
            messageDigest.update(salt);
            byte[] bytes = messageDigest.digest(password.getBytes());

            StringBuilder stringBuilder = new StringBuilder();
            for (int i = 0; i < bytes.length; i++)
                stringBuilder.append(Integer.toString((bytes[i] & 0xff) + 0x100, 16).substring(1));

            encryptedPassword = stringBuilder.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }

        return encryptedPassword;
    }

    public static byte[] getSalt() throws NoSuchAlgorithmException, NoSuchProviderException {
        SecureRandom secureRandom = SecureRandom.getInstance("SHA1PRNG", "SUN");
        byte[] salt = new byte[16];
        secureRandom.nextBytes(salt);
        return salt;
    }

    public static void main(String[] args) throws NoSuchProviderException, NoSuchAlgorithmException {
        String password = "password123";
        System.out.println(encode(password));
        System.out.println(encode(password, "randomsalt"));
        System.out.println(encode(password, getSalt()));
    }
}
```

# Sign Up

Below is User controller for signing up. 

```java
package com.dankunlee.forumapp.controller;

import com.dankunlee.forumapp.entity.User;
import com.dankunlee.forumapp.repository.UserRepository;
import com.dankunlee.forumapp.utils.Password;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseBody;

import javax.validation.Valid;

@Controller // @RestController = @Controller + @ResponseBody
public class UserController {
    @Autowired
    private UserRepository userRepository;

    @PostMapping("/api/register")
    @ResponseBody
    public String register(@Valid @RequestBody User user) {
        String username = user.getUsername();
        String rawPassword = user.getPassword();
        String encryptedPassword = Password.encode(rawPassword, Password.randomSalt);
        String email = user.getEmail();

        if (username.equals("") || encryptedPassword.equals("") || email.equals(""))
            return "Empty Parameter is Given";

        if (userRepository.findByUsername(username) != null) return "Fail: Existing Username";
        if (userRepository.findByEmail(email) != null) return "Fail: Existing Email";

        User newUser = new User();
        newUser.setUsername(username);
        newUser.setPassword(encryptedPassword);
        newUser.setEmail(email);
        userRepository.save(newUser);

        return "Success";
    }
}
```

When we run the application, you'll find that the password stored in User table has been encoded. 

![image](/assets/images/tutorial1/SHA-256.png)  

# Sign In

Now it's time to implement signing in. 

Unlike signing up, signing in requires more functionalities such as authentication, session and interceptor. 

## Authentication

We need to validate that a user trying to access the application is in the allowed list. 

We can do this by comparing the user's log-in information to the information we have on DB. 

Remember that we need to hash given password and compare it with the hashed password on DB. 

If only both of username and password match, the user can log-in with a session. 

## Session

A session is a temporary and interactive information interchange between the user and the application. 

It is established at a certain time and at later point destroyed. 

We'll be using sessions to confirm if users are still logged in and only logged in users can access the application. 

Then do we need to implement this for all Controllers? 

## Interceptor

Spring offers interceptor for _intercepting_ HTTP response/request packets before they reach Services or Controllers. 

Using interceptor, we can check if users' sessions are alive before accessing any Controllers. 

Only users with live sessions will be able to use the application. 

Under _utils_ package, let's create a Session class first. 

```java
package com.dankunlee.forumapp.utils;

public class Session {
    public static final String SESSION_ID = "Session ID";
}
```
---
Below is the implementation of log-in interceptor. 

```java
package com.dankunlee.forumapp.interceptor;

import com.dankunlee.forumapp.utils.Session;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

@Component
public class LogInInterceptor implements HandlerInterceptor {
    @Override
    // used to perform operations before sending the request to the controller
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HttpSession session = request.getSession();
        if (session.getAttribute(Session.SESSION_ID) == null) {
            // sends msg when urls from WebConfigurer.addInterceptors are accessed without logging in
            response.getOutputStream().print("Login Required");
            return false;
        }
        return true;
    }

    @Override
    // used to perform operations before sending the response to the client
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    // used to perform operations after completing the request and response
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}
```

---
Now let's create WebConfigurer class and register the interceptor there.

The interceptor will operate at specified URLs and check for verified session. 

```java
package com.dankunlee.forumapp.config;

import com.dankunlee.forumapp.interceptor.CommentAuthorizationInterceptor;
import com.dankunlee.forumapp.interceptor.LogInInterceptor;
import com.dankunlee.forumapp.interceptor.PostAuthorizationInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfigurer implements WebMvcConfigurer {
    @Autowired
    LogInInterceptor logInInterceptor;

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
    }
}
```

## Sign In Controller

It's time to build a log-in controller. 

When a user signs in, a session will be created and when the user signs out, the created session will be destroyed. 

```java
package com.dankunlee.forumapp.controller;

import com.dankunlee.forumapp.dto.UserDTO;
import com.dankunlee.forumapp.entity.User;
import com.dankunlee.forumapp.repository.UserRepository;
import com.dankunlee.forumapp.utils.Password;
import com.dankunlee.forumapp.utils.Session;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseBody;

import javax.servlet.http.HttpSession;

@Controller
public class LogInController {
    @Autowired
    UserRepository userRepository;

    @PostMapping("/api/login")
    @ResponseBody
    public String login(@RequestBody UserDTO userDTO, HttpSession session) {
        String username = userDTO.getUsername();
        String rawPassword = userDTO.getPassword();
        String encryptedPassword = Password.encode(rawPassword, Password.randomSalt);

        User user = userRepository.findByUsernameAndPassword(username, encryptedPassword);

        if (user == null) return "Fail";
        session.setAttribute(Session.SESSION_ID, username);
        return "Logged In";
    }

    @PostMapping("/api/logout")
    @ResponseBody
    public String logout(HttpSession session) {
        session.invalidate();
        return "Logged Out";
    }
}
```

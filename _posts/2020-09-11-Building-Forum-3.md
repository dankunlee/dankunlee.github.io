---
title: "Building a simple web forum 3: Sign Up and Sign In"
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

The steps for signing up is simple:

1. The user will transmit user information including username, password and e-mail. 
2. The application will check for any duplication and add the information to its DB. 
3. All passwords must be encrypted before they are stored. 

In order to sign up, User funcationality must be enabled first. 

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


Then, User repository:

Note that Spring JPA allows custom queries by following its naming convention.  
(ex. findByxxx(String xxx);)

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


Before we create User controller, we need to be able to encrypt given password. 

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

When we run the application, you'll find that the password stored in User table is encoded. 

![image](/assets/images/tutorial1/SHA-256.png)  

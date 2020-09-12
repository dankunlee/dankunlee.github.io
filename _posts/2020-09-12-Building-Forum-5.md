---
title: "Building a simple web forum 5: Paging & Searching"
date: 2020-09-12T16:34:30-04:00
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

All the core funcitonalities of an internet forum has been built. 

Now it's time to add additional functionalities such as paging or searching. 

The simplest way to implement those are using SQL queries. 

# Paging

If we use SQL query, the possible way would be: 

```sql
SELECT * 
FROM Post 
WHERE post_id > pageLength * (pageNumb - 1) 
LIMIT pageLength;
```

As mentioned earlier, in Spring, repositories(DTO) handle accessing data from DB. 

We can modify and re-use the Post repository to handle paging then. 

Let's use Spring's Page and Pageable interfaces to easily implement paging. 

```java
public interface PostRepository extends JpaRepository<Post, Long> {
    public Page<Post> findAll(Pageable pageable); // repository for paging
}
```

---
Then DTO for paging would be:

```java
package com.dankunlee.forumapp.dto;

import java.time.LocalDateTime;

public class PagingDTO {
    private Long id;
    private String title;
    private String createdBy;
    private LocalDateTime createdDate;

    public PagingDTO(Long id, String title, String createdBy, LocalDateTime createdDate) {
        this.id = id;
        this.title = title;
        this.createdBy = createdBy;
        this.createdDate = createdDate;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getCreatedBy() {
        return createdBy;
    }

    public void setCreatedBy(String createdBy) {
        this.createdBy = createdBy;
    }

    public LocalDateTime getCreatedDate() {
        return createdDate;
    }

    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
}
```

---
Controller for Paging:

```java
package com.dankunlee.forumapp.controller;

import com.dankunlee.forumapp.dto.PagingDTO;
import com.dankunlee.forumapp.entity.Post;
import com.dankunlee.forumapp.repository.PostRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class PagingController {
    @Autowired
    PostRepository postRepository;

    @GetMapping("/api/post/page") // ex. "/api/post/page?page=1&size=6"
    @ResponseBody
    public Page<PagingDTO> paging(@PageableDefault(size = 5, sort = "createdDate", direction = Sort.Direction.DESC)
                                   Pageable pageRequest) {
        Page<Post> page = postRepository.findAll(pageRequest);
        Page<PagingDTO> page_dto = page.map((post) -> new PagingDTO(post.getId(), post.getTitle(),
                                                                    post.getCreatedBy(), post.getCreatedDate()));
        return page_dto;
    }
}
```

---
Don't forget to exclude the URL for paging from interceptors. 

```java
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
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
```

---
Try testing the paging functionality: ex. http://localhost:8080/api/post/page?page=0&size=3

# Searching

Unlike paging, the SQL query for searching is a bit complexed. 

```sql
SELECT post 
FROM Post post 
WHERE post.title LIKE %:title% OR post.content LIKE %:content%;

SELECT COUNT(post.id) 
FROM Post post 
WHERE post.title LIKE %:title% OR post.content LIKE %:content%;
```

The first query searches for a certain post and the second query counts the number of found posts (this is required if we want to display the results with paging). 

The modified Post repository will look like this:

```java
public interface PostRepository extends JpaRepository<Post, Long> {
    public Page<Post> findAll(Pageable pageable); // repository for paging

    @Query (
            value = "SELECT post FROM Post post WHERE post.title LIKE %:title% OR post.content LIKE %:content%",
            countQuery = "SELECT COUNT(post.id) FROM Post post WHERE post.title LIKE %:title% OR post.content LIKE %:content%"
    )
    public Page<Post> findAllSearch(String title, String content, Pageable pageable); // repository for searching
}
```

As you can see, Spring JPA allows direct mapping of SQL quries to a repository. 

We continue modifying Post Controller since searching technically belongs to paging. 

```java
@Controller
public class PagingController {
    @Autowired
    PostRepository postRepository;

    @GetMapping("/api/post/page") // ex. "/api/post/page?page=1&size=6"
    @ResponseBody
    public Page<PagingDTO> paging(@PageableDefault(size = 5, sort = "createdDate", direction = Sort.Direction.DESC)
                                   Pageable pageRequest) {
        Page<Post> page = postRepository.findAll(pageRequest);
        Page<PagingDTO> page_dto = page.map((post) -> new PagingDTO(post.getId(), post.getTitle(),
                                                                    post.getCreatedBy(), post.getCreatedDate()));
        return page_dto;
    }

    @GetMapping("/api/post/page/search") // ex. "/api/post/page/search?keyword=post&page=0&size=3"
    @ResponseBody
    public Page<PagingDTO> search(@RequestParam String keyword,
                                  @PageableDefault(size = 5, sort = "createdDate", direction = Sort.Direction.DESC)
                                   Pageable pageRequest) {
        Page<Post> page = postRepository.findAllSearch(keyword, keyword, pageRequest);
        Page<PagingDTO> page_dto = page.map((post) -> new PagingDTO(post.getId(), post.getTitle(),
                post.getCreatedBy(), post.getCreatedDate()));
        return page_dto;
    }
}
```

Try testing searching with:  
"https://localhost:8080/api/post/page/search?keyword=lunch&page=0&size=13".


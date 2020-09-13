---
title: "Building a simple web forum 6: File Uploading and Downloading"
date: 2020-09-12T18:34:30-04:00
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

# How do we send binary data?

HTTP(Hypertext Transfer Protocol) is a stateless protocol. 

Meaning the server won't retain information or status about each user for the duration of multiple requests. 

This does not raise a probelm for regular usages as we are just sending key-value pairs (with content type of _application/x-www-form-urlencoded_).

However if we want to transfer a file(binary data), how do we send the file's metadata and its content in one piece? 

This is where _multipart/form-data_ comes in place. This content type sends each value as a block of data ("body part"), with a user agent-defined delimiter ("boundary") separating each part.

Let's see an example for each case. 

---
_application/x-www-form-urlencoded_:

```
POST /test HTTP/1.1
Host: foo.example
Content-Type: application/x-www-form-urlencoded
Content-Length: 27

field1=value1&field2=value2
```

---
_multipart/form-data_:

```
POST /test HTTP/1.1 
Host: foo.example
Content-Type: multipart/form-data;boundary="boundary" 

--boundary 
Content-Disposition: form-data; name="field1" 

value1 
--boundary 
Content-Disposition: form-data; name="field2"; filename="example.txt" 

value2
--boundary--
```

# Configuration

Let's set the configuration for uploading/downloading files functionality provided by Spring. 

Add following settings to application.yml:

```
  servlet: # under spring
    multipart:
      enabled: true
      file-size-threshold: 2KB
      max-file-size: 200MB
      max-request-size: 200MB

# Path to where uploaded files will be saved
file:
  path: "/Desktop"
```

---
"file - path" can be applied to a configurer class using @ConfigurationProperties annotation. 

```java
package com.dankunlee.forumapp.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration  // v provides easy access to defined properties
@ConfigurationProperties(prefix = "file") // file from application.xml
public class FileConfigurer {
    private String path; // path under file from application.xml

    public String getPath() {
        return path;
    }

    public void setPath(String path) {
        this.path = path;
    }
}
```

## Entity, DTO and Repository

As always, we need an entity. 

```java
package com.dankunlee.forumapp.entity;

import org.hibernate.annotations.OnDelete;
import org.hibernate.annotations.OnDeleteAction;

import javax.persistence.*;

@Entity
public class File {
    @Id
    @GeneratedValue
    @Column(name = "file_id")
    private Long id;

    private String fileName;

    private String fileType;

    private String fileUri;

    private Long fileSize;

    @ManyToOne
    @JoinColumn(name = "post_id")
    @OnDelete(action = OnDeleteAction.CASCADE)
    private Post post;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getFileName() {
        return fileName;
    }

    public void setFileName(String fileName) {
        this.fileName = fileName;
    }

    public String getFileType() {
        return fileType;
    }

    public void setFileType(String fileType) {
        this.fileType = fileType;
    }

    public String getFileUri() {
        return fileUri;
    }

    public void setFileUri(String fileUri) {
        this.fileUri = fileUri;
    }

    public Long getFileSize() {
        return fileSize;
    }

    public void setFileSize(Long fileSize) {
        this.fileSize = fileSize;
    }

    public Post getPost() {
        return post;
    }

    public void setPost(Post post) {
        this.post = post;
    }
}
```

---
Next, we need DTO: 

```java
package com.dankunlee.forumapp.dto;

public class FileDTO {
    private String fileName;
    private String fileType;
    private String fileUri;
    private Long fileSize;

    public FileDTO(String fileName, String fileType, String fileUri, Long fileSize) {
        this.fileName = fileName;
        this.fileType = fileType;
        this.fileUri = fileUri;
        this.fileSize = fileSize;
    }

    public String getFileName() {
        return fileName;
    }

    public void setFileName(String fileName) {
        this.fileName = fileName;
    }

    public String getFileType() {
        return fileType;
    }

    public void setFileType(String fileType) {
        this.fileType = fileType;
    }

    public String getFileUri() {
        return fileUri;
    }

    public void setFileUri(String fileUri) {
        this.fileUri = fileUri;
    }

    public Long getFileSize() {
        return fileSize;
    }

    public void setFileSize(Long fileSize) {
        this.fileSize = fileSize;
    }
}
```

---
Repository:

```java
package com.dankunlee.forumapp.repository;

import com.dankunlee.forumapp.entity.File;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface FileRepository extends JpaRepository<File, Long> {
    @Query("SELECT file FROM File file WHERE post_id = :id") // post_id already exists in UploadedFile entity
    public List<File> findAllByPostId(Long id);
}
```

# Service

Now, we'll be building Service for uploading and downloading files. 

Service will also include exception classes to handle any errors that might occur during file transfer. 

_storeFile_ method is for uploading and _loadFile_ is for downloading files

```java
package com.dankunlee.forumapp.service;

import com.dankunlee.forumapp.config.FileConfigurer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.net.MalformedURLException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;

@Service
public class FileService {
    public final Path fileStoragePath;
    private final String POST = "post_";

    @Autowired
    public FileService(FileConfigurer fileConfigurer) {
        fileStoragePath = Paths.get(fileConfigurer.getPath()).toAbsolutePath().normalize();
        try{
            Files.createDirectories(fileStoragePath);
        } catch (IOException e) {
            throw new FileStorageException("Cannot create directories for uploaded files");
        }
    }

    public String storeFile(Long id, MultipartFile file) {
        String fileName = StringUtils.cleanPath(file.getOriginalFilename());
        try{
            Path fileNamePath = fileStoragePath.resolve(POST + id).resolve(fileName);
            Files.createDirectories(fileNamePath);
            Files.copy(file.getInputStream(), fileNamePath, StandardCopyOption.REPLACE_EXISTING);
            return fileName;
        } catch (IOException e) {
            throw new FileStorageException("Unable to upload " + fileName, e);
        }
    }

    public Resource loadFile(Long id, String fileName) {
        try {
            Path fileNamePath = fileStoragePath.resolve(POST + id).resolve(fileName);
            Resource resource = new UrlResource(fileNamePath.toUri());

            if (resource.exists()) return resource;
            else throw new FileNotFoundException(fileName + " is not found");
        } catch (MalformedURLException e) {
            throw new FileNotFoundException(fileName + " is not found", e);
        }
    }
}

// handles failure of file uploading
class FileStorageException extends RuntimeException {
    public FileStorageException(String message) {
        super(message);
    }

    public FileStorageException(String message, Throwable cause) {
        super(message, cause);
    }
}

// handles failure of file downloading
@ResponseStatus(HttpStatus.NOT_FOUND)
class FileNotFoundException extends RuntimeException {
    public FileNotFoundException(String message) {
        super(message);
    }

    public FileNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## Controller

Controller will use the file Service to interact with users for file related jobs. 

_uploadFile_ method saves the file uploaded by a user and stores the link for downloading the file in DB.  
Then it returns the link with the file information such as file name, size, and type. 

_uploadMultipleFiles_ method is same as _uploadFile_ method, except this method can handle uploading multiple files. 



```java
package com.dankunlee.forumapp.controller;

import com.dankunlee.forumapp.dto.FileDTO;
import com.dankunlee.forumapp.entity.File;
import com.dankunlee.forumapp.entity.Post;
import com.dankunlee.forumapp.repository.FileRepository;
import com.dankunlee.forumapp.repository.PostRepository;
import com.dankunlee.forumapp.service.FileService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@RestController
public class FileController {
    private static final Logger logger = LoggerFactory.getLogger(FileController.class);

    @Autowired
    private FileService fileService;

    @Autowired
    private FileRepository fileRepository;

    @Autowired
    private PostRepository postRepository;

    @PostMapping("/api/post/{id}/upload-file")
    public FileDTO uploadFile(@PathVariable Long id, @RequestParam("file") MultipartFile file) {
        String fileName = fileService.storeFile(id, file);
        String fileDownloadUri = ServletUriComponentsBuilder.fromCurrentContextPath()
                                    .path("/api/post/")
                                    .path(Long.toString(id))
                                    .path("/download-file/")
                                    .path(fileName)
                                    .toUriString();
        File uploadFile = new File();
        uploadFile.setFileName(fileName);
        uploadFile.setFileType(file.getContentType());
        uploadFile.setFileUri(fileDownloadUri);
        uploadFile.setFileSize(file.getSize());

        Post post = postRepository.findById(id).get();
        uploadFile.setPost(post);

        fileRepository.save(uploadFile);

        return new FileDTO(fileName, file.getContentType(), fileDownloadUri, file.getSize());
    }

    @PostMapping("/api/post/{id}/upload-multiple-files")
    public List<FileDTO> uploadMultipleFiles(@PathVariable Long id, @RequestParam("files") MultipartFile[] files) {
        List<FileDTO> list = new ArrayList<>();
        for (MultipartFile file : files)
            list.add(uploadFile(id, file));
        return list;
    }

    @GetMapping("/api/post/{id}/files")
    public List<FileDTO> downloadFilesInfo(@PathVariable Long id) {
        List<File> files = fileRepository.findAllByPostId(id);
        List<FileDTO> fileDTOs = new ArrayList<>();
        for (File file : files)
            fileDTOs.add(new FileDTO(file.getFileName(), file.getFileType(), file.getFileUri(), file.getFileSize()));
        return fileDTOs;
    }

    @GetMapping("/api/post/{id}/download-file/{fileName:.+}") // regex .+ for file extension
    public ResponseEntity<Resource> downloadFile(@PathVariable Long id, @PathVariable String fileName, HttpServletRequest request) {
        Resource resource = fileService.loadFile(id, fileName);

        String contentType = null;
        try{
            contentType = request.getServletContext().getMimeType(resource.getFile().getAbsolutePath());
        } catch (IOException e) {
            logger.info("Unknown File Type", e);
        }

        if (contentType == null) contentType = "application/octect-stream";

        return ResponseEntity.ok().contentType(MediaType.parseMediaType(contentType))
                                  .header(HttpHeaders.CONTENT_DISPOSITION, "attatchment; filename=\""
                                                                            + resource.getFilename() + "\"")
                                  .body(resource);
    }

}
```


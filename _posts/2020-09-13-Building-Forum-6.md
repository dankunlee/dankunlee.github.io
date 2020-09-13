 ---
title: "Building a simple web forum 6: File Uploading and Downloading"
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

# How do we send binary data?

HTTP(Hypertext Transfer Protocol) is a stateless protocol. 

Meaning the server won't retain information or status about each user for the duration of multiple requests. 

This does not raise a probelm for regular usages as we are just sending key-value pairs (with content type of _application/x-www-form-urlencoded_).

However if we want to transfer a file(binary data), how do we send the file's metadata and its content in one piece? 

This is where _multipart/form-data_ comes in place. This content type sends each value as a block of data ("body part"), with a user agent-defined delimiter ("boundary") separating each part.

Let's see examples for each case. 

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
_multipart/form-data_
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




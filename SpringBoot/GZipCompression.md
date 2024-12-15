
### Implementing GZIP Compression in Spring Boot

Problem: Slow-loading websites can frustrate users and increase bounce rates. One way to improve performance is through compression techniques to reduce data size during server-client communication. GZIP is widely used for this purpose in Spring Boot applications.

<img width="696" alt="image" src="https://github.com/user-attachments/assets/c8ea1bbb-b07c-4387-8551-f53c7c231abe">


### Spring Boot Configuration for GZIP Compression

To enable GZIP compression in Spring Boot, add the following to your application.properties


```
server.compression.enabled=true
server.compression.mime-type=text/html,text/xml,text/plain,application/json,application/xml,text/css,text/javascript,application/javascript
server.compression.min-response-size=1024
```

```
server.compression.min-response-size=1024
```
* This sets the minimum response size for compression. Any response smaller than this size (in bytes) will not be compressed, even if the MIME type matches the compression.mime-types list.



### Testing the Impact

* Without GZIP:
    * Response time: 1366 ms
    * Data size: 18.82 MB
    * The server sends the uncompressed data.

* With GZIP:
    * Response time: 159 ms
    * Data size: 1.75 MB
    * GZIP compresses the response before sending it.


### Validating Compression


1. Postman:
    * After compression, check response headers. You should see Content-Encoding: gzip.
2. Browser:
    * Open Developer Tools > Network tab. Inspect the response to confirm GZIP compression with the Content-Encoding: gzip header.



### Browser Decompression

When it comes to decompression, browsers automatically handle GZIP-compressed responses as long as they support it (which most modern browsers do). The browser will check for the Content-Encoding: gzip header in the HTTP response and decompress the data before rendering or using it. This means you don't have to worry about applying anything specific in your Angular code or other frontend frameworks to handle GZIP decompression.


### Reference 

- https://www.baeldung.com/json-reduce-data-size
- https://youtu.be/ppJK7nksGGM?si=uC1anvwIAE7SxKhH 
- https://github.com/Java-Techie-jt/gzip-compression/tree/main 




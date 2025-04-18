
# Remote Debugging Spring Boot App inside Docker

This guide walks you through setting up remote debugging for a Spring Boot application running inside a Docker container.

## 🛠️ Prerequisites
- Docker Desktop
- Maven
- IntelliJ IDEA
- Java 8

## 📦 Step 1: Create and Package Spring Boot Application

Ensure your Spring Boot app is set up. 

## 🐳 Step 2: Create Dockerfile

Create a `Dockerfile` in your project root with:

```Dockerfile
FROM openjdk:8
EXPOSE 8080 80
ADD target/spring-docker-demo.jar spring-docker-demo.jar
ADD entrypoint.sh entrypoint.sh
ENTRYPOINT ["sh", "entrypoint.sh"]
```

## 📝 Step 3: Create Entrypoint Script

Create a file named `entrypoint.sh` in your root directory:

```sh
#!/bin/sh
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,address=8000,suspend=n -jar spring-docker-demo.jar
```

Make it executable:

```bash
chmod +x entrypoint.sh
```

## 🏗️ Step 4: Build Docker Image

```bash
docker build -t spring-docker-debugging:1.0 .
```

## ▶️ Step 5: Run Docker Container

```bash
docker run -p 8080:8080 -p 80:80 spring-docker-debugging:1.0
```

You should see output like:

```
Listening for transport dt_socket at address: 8000
```

## 🧠 Step 6: Enable Remote Debugging in IntelliJ

- Go to `Run > Edit Configurations`
- Click `+` and select **Remote JVM Debug**
- Set:
  - Name: Docker Remote Debugging
  - Port: 8000
  - Host: localhost

Click **OK** and then click **Debug** to attach to the running container.

https://blog.jetbrains.com/wp-content/uploads/2019/04/idea-docker-debug-config.png![image](https://github.com/user-attachments/assets/913d818c-3601-4fbe-8f90-60d630d674fb)


## ✅ Verify Debugging

Set breakpoints in your controller or service classes and hit the endpoints to confirm if breakpoints are triggered.


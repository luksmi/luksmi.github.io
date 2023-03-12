---
title: Quickly build your first app using Spring Boot 3.0
date: 2023-03-12 19:00:00 +0100
categories: [programming, beginner]
tags: [programming, junior, beginner, languages]
---

[Spring Boot](https://spring.io/) is a popular Java framework that makes it easy to build powerful, scalable, and secure web applications quickly. In this tutorial, we will go through the steps to build a simple web application using Spring Boot.

## Prerequisites
Before we start, make sure you have the following software installed on your computer:
- JDK 1.8 or higher
- Maven
- IDE (Integrated Development Environment) - We recommend using IntelliJ IDEA Community Edition, but you can use any IDE of your choice.

## Step 1: Create a new Spring Boot project
The first step is to create a new Spring Boot project. You can use the Spring Initializr website or use the command-line tool to create the project.

### Option 1: Using the Spring Initializr website
1. Go to the [**Spring Initializr website**](https://start.spring.io/).
2. Fill out the project details like Group, Artifact, and Dependencies.
3. Click on the Generate button to download the project.

### Option 2: Using the command-line tool
1. Open the terminal/command prompt on your computer.
2. Run the following command to create a new Spring Boot project:

```console
mvn archetype:generate -DgroupId=com.example -DartifactId=myapp -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

## Step 2: Create a Controller
The next step is to create a controller to handle HTTP requests. A controller is a Java class that is responsible for handling HTTP requests and returning responses. In this example, we will create a simple controller that returns a "Hello, World!" message.

1. Create a new Java class in the `src/main/java/com/example/myapp`{: .filepath} package.
2. Add the following code to the class:

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloWorldController {

    @GetMapping("/")
    public String helloWorld() {
        return "Hello, World!";
    }
}
```

In this code, we have created a new class called `HelloWorldController` that is annotated with `@RestController`. This annotation tells Spring Boot that this class will handle HTTP requests and return responses.

We have also added a `@GetMapping("/")` annotation to the `helloWorld()` method. This annotation tells Spring Boot that this method should handle HTTP GET requests to the root URL ("/") of our application.

The `helloWorld()` method simply returns the string "Hello, World!".

## Step 3: Run the application
The final step is to run the application and test it in a web browser.

1. Open your IDE and import the project you created in Step 1.
2. Navigate to the main class of your application (src/main/java/com/example/myapp/MyAppApplication.java) and run it.
3. Open a web browser and go to `http://localhost:8080/`. You should see the `"Hello, World!"` message displayed on the page.
   

Congratulations! You have just built a simple web application using Spring Boot.

## Conclusion
In this tutorial, we have learned how to create a simple web application using Spring Boot. We started by creating a new Spring Boot project and then created a controller to handle HTTP requests. Finally, we ran the application and tested it in a web browser. Spring Boot makes it easy to build powerful web applications quickly, and we hope this tutorial has given you a good starting point for your own Spring Boot projects.
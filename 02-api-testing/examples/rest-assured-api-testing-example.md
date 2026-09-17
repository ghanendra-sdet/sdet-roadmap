# ☕ REST-assured API Testing Example

![Java](https://img.shields.io/badge/Java-11+-blue.svg)
![REST-assured](https://img.shields.io/badge/REST--assured-5.3.2-green.svg)
![Last Updated](https://img.shields.io/badge/last%20updated-Feb%202026-orange.svg)

> **Learn Java-based API testing with REST-assured framework**

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Setup](#-setup)
- [Project Structure](#-project-structure)
- [GET Request](#-get-request-example)
- [POST Request](#-post-request-example)
- [PUT Request](#-put-request-example)
- [DELETE Request](#-delete-request-example)
- [Validations](#-validations-and-assertions)
- [Authentication](#-authentication)
- [Test Organization](#-test-organization)
- [Best Practices](#-best-practices)

---

## 🎯 Overview

**REST-assured** is a Java library for testing REST APIs with:
- ✅ BDD-style syntax (Given-When-Then)
- ✅ Built-in JSON/XML parsing
- ✅ Request/response specifications
- ✅ Schema validation
- ✅ Integration with TestNG/JUnit

---

## 🔧 Setup

### Maven Dependencies

**pom.xml:**
```xml
<dependencies>
    <!-- REST-assured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.3.2</version>
        <scope>test</scope>
    </dependency>
    
    <!-- TestNG -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.8.0</version>
        <scope>test</scope>
    </dependency>
    
    <!-- JSON Schema Validator -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-schema-validator</artifactId>
        <version>5.3.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle Dependencies

**build.gradle:**
```groovy
dependencies {
    testImplementation 'io.rest-assured:rest-assured:5.3.2'
    testImplementation 'org.testng:testng:7.8.0'
    testImplementation 'io.rest-assured:json-schema-validator:5.3.2'
}
```

---

## 📁 Project Structure

```
src/test/java
├── com.api.tests
│   ├── PostsAPITest.java
│   ├── UsersAPITest.java
│   └── CommentsAPITest.java
├── com.api.utils
│   ├── BaseTest.java
│   ├── Constants.java
│   └── TestData.java
└── resources
    ├── schemas
    │   ├── post-schema.json
    │   └── user-schema.json
    └── testng.xml
```

---

## 📝 Base Test Setup

**BaseTest.java:**
```java
package com.api.tests;

import io.restassured.RestAssured;
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.builder.ResponseSpecBuilder;
import io.restassured.specification.RequestSpecification;
import io.restassured.specification.ResponseSpecification;
import org.testng.annotations.BeforeClass;

public class BaseTest {
    
    protected static RequestSpecification requestSpec;
    protected static ResponseSpecification responseSpec;
    
    @BeforeClass
    public static void setup() {
        RestAssured.baseURI = "https://jsonplaceholder.typicode.com";
        
        // Request Specification
        requestSpec = new RequestSpecBuilder()
            .setContentType("application/json")
            .setAccept("application/json")
            .build();
        
        // Response Specification
        responseSpec = new ResponseSpecBuilder()
            .expectContentType("application/json")
            .expectResponseTime(lessThan(3000L))
            .build();
    }
}
```

---

## 🔵 GET Request Example

**PostsAPITest.java:**
```java
package com.api.tests;

import io.restassured.response.Response;
import org.testng.annotations.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

public class PostsAPITest extends BaseTest {
    
    @Test
    public void test_GetAllPosts() {
        given()
            .spec(requestSpec)
        .when()
            .get("/posts")
        .then()
            .spec(responseSpec)
            .statusCode(200)
            .body("$", hasSize(100))
            .body("[0].userId", notNullValue())
            .body("[0].id", notNullValue())
            .body("[0].title", notNullValue())
            .body("[0].body", notNullValue())
            .log().ifValidationFails();
    }
    
    @Test
    public void test_GetSinglePost() {
        int postId = 1;
        
        given()
            .spec(requestSpec)
            .pathParam("id", postId)
        .when()
            .get("/posts/{id}")
        .then()
            .spec(responseSpec)
            .statusCode(200)
            .body("id", equalTo(postId))
            .body("userId", greaterThan(0))
            .body("title", not(emptyString()))
            .body("body", not(emptyString()));
    }
    
    @Test
    public void test_GetPostsWithQueryParam() {
        given()
            .spec(requestSpec)
            .queryParam("userId", 1)
        .when()
            .get("/posts")
        .then()
            .statusCode(200)
            .body("$", hasSize(greaterThan(0)))
            .body("userId", everyItem(equalTo(1)));
    }
    
    @Test
    public void test_ExtractResponseData() {
        Response response = 
            given()
                .spec(requestSpec)
            .when()
                .get("/posts/1")
            .then()
                .statusCode(200)
                .extract().response();
        
        // Extract specific fields
        int userId = response.path("userId");
        String title = response.path("title");
        
        System.out.println("User ID: " + userId);
        System.out.println("Title: " + title);
        
        // Assertions
        assert userId > 0;
        assert !title.isEmpty();
    }
}
```

---

## 🟢 POST Request Example

```java
@Test
public void test_CreateNewPost() {
    String requestBody = """
        {
            "title": "Test Post",
            "body": "This is a test post",
            "userId": 1
        }
        """;
    
    given()
        .spec(requestSpec)
        .body(requestBody)
    .when()
        .post("/posts")
    .then()
        .statusCode(201)
        .body("title", equalTo("Test Post"))
        .body("body", equalTo("This is a test post"))
        .body("userId", equalTo(1))
        .body("id", notNullValue());
}

@Test
public void test_CreatePostWithPOJO() {
    Post post = new Post();
    post.setTitle("New Post");
    post.setBody("Post content");
    post.setUserId(1);
    
    Response response = 
        given()
            .spec(requestSpec)
            .body(post)
        .when()
            .post("/posts")
        .then()
            .statusCode(201)
            .extract().response();
    
    Post createdPost = response.as(Post.class);
    assert createdPost.getId() != null;
    assert createdPost.getTitle().equals("New Post");
}

@Test
public void test_CreatePostWithMap() {
    Map<String, Object> postData = new HashMap<>();
    postData.put("title", "Map Post");
    postData.put("body", "Created using Map");
    postData.put("userId", 1);
    
    given()
        .spec(requestSpec)
        .body(postData)
    .when()
        .post("/posts")
    .then()
        .statusCode(201)
        .body("title", equalTo("Map Post"));
}
```

---

## 🟡 PUT Request Example

```java
@Test
public void test_UpdatePost() {
    String updatedBody = """
        {
            "id": 1,
            "title": "Updated Title",
            "body": "Updated content",
            "userId": 1
        }
        """;
    
    given()
        .spec(requestSpec)
        .pathParam("id", 1)
        .body(updatedBody)
    .when()
        .put("/posts/{id}")
    .then()
        .statusCode(200)
        .body("title", equalTo("Updated Title"))
        .body("body", equalTo("Updated content"))
        .body("id", equalTo(1));
}

@Test
public void test_PartialUpdate() {
    Map<String, String> updates = Map.of(
        "title", "Partially Updated Title"
    );
    
    given()
        .spec(requestSpec)
        .pathParam("id", 1)
        .body(updates)
    .when()
        .patch("/posts/{id}")
    .then()
        .statusCode(200)
        .body("title", equalTo("Partially Updated Title"));
}
```

---

## 🔴 DELETE Request Example

```java
@Test
public void test_DeletePost() {
    int postId = 1;
    
    given()
        .spec(requestSpec)
        .pathParam("id", postId)
    .when()
        .delete("/posts/{id}")
    .then()
        .statusCode(200);
}

@Test
public void test_DeleteNonExistentPost() {
    given()
        .spec(requestSpec)
        .pathParam("id", 99999)
    .when()
        .delete("/posts/{id}")
    .then()
        .statusCode(200); // JSONPlaceholder returns 200 even for non-existent
}
```

---

## ✅ Validations and Assertions

### Response Body Validation

```java
@Test
public void test_CompleteValidation() {
    given()
        .spec(requestSpec)
    .when()
        .get("/posts/1")
    .then()
        // Status code
        .statusCode(200)
        
        // Response time
        .time(lessThan(2000L))
        
        // Headers
        .header("Content-Type", containsString("application/json"))
        
        // Body validations
        .body("id", equalTo(1))
        .body("userId", greaterThan(0))
        .body("title", notNullValue())
        .body("title", not(emptyString()))
        .body("body.length()", greaterThan(10))
        
        // Log
        .log().all();
}
```

### JSON Schema Validation

**post-schema.json:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema",
  "type": "object",
  "required": ["userId", "id", "title", "body"],
  "properties": {
    "userId": { "type": "integer" },
    "id": { "type": "integer" },
    "title": { "type": "string" },
    "body": { "type": "string" }
  }
}
```

**Test with Schema:**
```java
@Test
public void test_ValidateResponseSchema() {
    given()
        .spec(requestSpec)
    .when()
        .get("/posts/1")
    .then()
        .statusCode(200)
        .body(matchesJsonSchemaInClasspath("schemas/post-schema.json"));
}
```

---

## 🔐 Authentication

### Basic Auth

```java
@Test
public void test_BasicAuth() {
    given()
        .auth().basic("username", "password")
    .when()
        .get("/secure-endpoint")
    .then()
        .statusCode(200);
}
```

### Bearer Token

```java
@Test
public void test_BearerToken() {
    String token = "your-auth-token";
    
    given()
        .header("Authorization", "Bearer " + token)
    .when()
        .get("/protected-resource")
    .then()
        .statusCode(200);
}
```

### OAuth2

```java
@Test
public void test_OAuth2() {
    given()
        .auth()
        .oauth2("access-token")
    .when()
        .get("/oauth-protected")
    .then()
        .statusCode(200);
}
```

---

## 🧪 Test Organization

### POJO Classes

**Post.java:**
```java
package com.api.models;

public class Post {
    private Integer id;
    private Integer userId;
    private String title;
    private String body;
    
    // Getters and Setters
    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }
    
    public Integer getUserId() { return userId; }
    public void setUserId(Integer userId) { this.userId = userId; }
    
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    
    public String getBody() { return body; }
    public void setBody(String body) { this.body = body; }
}
```

### TestNG Configuration

**testng.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="API Test Suite">
    <test name="Posts API Tests">
        <classes>
            <class name="com.api.tests.PostsAPITest"/>
        </classes>
    </test>
    <test name="Users API Tests">
        <classes>
            <class name="com.api.tests.UsersAPITest"/>
        </classes>
    </test>
</suite>
```

---

## 🧠 Best Practices

### ✅ DO:
- ✔️ Use request/response specifications
- ✔️ Extract common setup to BaseTest
- ✔️ Use POJO for complex objects
- ✔️ Validate schema, status, and response time
- ✔️ Use meaningful test names
- ✔️ Log failures for debugging

### ❌ DON'T:
- ✖️ Hardcode URLs and credentials
- ✖️ Skip negative test cases
- ✖️ Ignore response validation
- ✖️ Create dependent tests

### Performance Example

```java
@Test
public void test_ResponseTimeValidation() {
    given()
        .spec(requestSpec)
    .when()
        .get("/posts")
    .then()
        .time(lessThan(2000L)) // Response < 2 seconds
        .statusCode(200);
}
```

---

## 🚀 Running Tests

```bash
# Maven
mvn clean test

# With specific test class
mvn test -Dtest=PostsAPITest

# With TestNG XML
mvn test -DsuiteXmlFile=testng.xml

# Generate reports
mvn surefire-report:report
```

---

## 🎓 Next Steps

1. Practice with different APIs
2. Implement data-driven testing
3. Add logging and reporting
4. Integrate with CI/CD
5. Learn advanced features (filters, interceptors)

**Resources:**
- [REST-assured Documentation](https://rest-assured.io/)
- [REST-assured GitHub](https://github.com/rest-assured/rest-assured)

---

<div align="center" markdown="1">

**⭐ Next: Check out Playwright API testing! ⭐**

Made with ❤️ for QA-to-SDET Journey | February 2026

</div>

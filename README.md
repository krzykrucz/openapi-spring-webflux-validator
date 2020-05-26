# openapi-spring-webflux-validator

![](https://travis-ci.org/cdimascio/openapi-spring-webflux-validator.svg?branch=master)[![Maven Central](https://img.shields.io/maven-central/v/io.github.cdimascio/openapi-spring-webflux-validator.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:%22io.github.cdimascio%22%20AND%20a:%22openapi-spring-webflux-validator%22)[![Codacy Badge](https://api.codacy.com/project/badge/Grade/f78b72ca90104e42b111723a7720adf3)](https://www.codacy.com/app/cdimascio/openapi-spring-webflux-validator?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=cdimascio/openapi-spring-webflux-validator&amp;utm_campaign=Badge_Grade)![](https://img.shields.io/badge/license-Apache%202.0-blue.svg)

A friendly kotlin library to validate API endpoints using an _OpenApi 3.0.0_ or _Swagger 2.0_ specification. Great with webflux functional. 
It **works happily with any JVM language including Java >=8**. 
<p align="center">
	<img src="https://raw.githubusercontent.com/cdimascio/openapi-spring-webflux-validator/master/assets/openapi-spring5-webflux-validator.png" width="600"/>
</p>

Supports specifications in _YAML_ and _JSON_

See this [complete Spring 5 Webflux example that uses openapi-spring-webflux-validator](https://github.com/cdimascio/kotlin-swagger-spring-functional-template).

## Prequisites

Java 8 or greater

## Install

### Maven

```xml
<dependency>
    <groupId>io.github.cdimascio</groupId>
    <artifactId>openapi-spring-webflux-validator</artifactId>
    <version>2.1.1</version>
</dependency>
```

### Gradle

```groovy
compile 'io.github.cdimascio:openapi-spring-webflux-validator:2.1.1'
```

For sbt, grape, ivy and more, see [here](https://search.maven.org/#artifactdetails%7Cio.github.cdimascio%7Copenapi-spring-webflux-validator%7C2.0.0%7Cjar)

## Usage (Kotlin)

This section and the next describe usage with Kotlin and Java respectively.

### Configure (Kotlin)

This one-time configuration requires you to provide the _location of the openapi/swagger specification_ and an optional _custom error handler_.

Supports `JSON` and `YAML`

```kotlin
import io.github.cdimascio.openapi.Validate
val validate = Validate.configure("static/api.yaml")
```

with custom error handler

```kotlin
data class MyError(val id: String, val messages: List<String>)
val validate = Validate.configure("static/api.json") { status, messages ->
   Error(status.name, messages)
}
```

with custom ObjectMapper factory:

```kotlin
val validate = Validate.configure(
   openApiSwaggerPath = "api.yaml",
   errorHandler = { status, message -> ValidationError(status.value(), message[0]) },
   objectMapperFactory = { ObjectMapper()
       .registerKotlinModule()
       .registerModule(JavaTimeModule())
       .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false) }
)
```

### Validate a request (Kotlin + Reactor)

Or you can now validate a request in a coroutine style,
using the `validate` instance created [above](#configure-kotlin):

without a body

```kotlin
validate.request(req) {
    // Do stuff e.g. return a list of names 
    ok().body(Mono.just(listOf("carmine", "alex", "eliana")))
}
```

with body

```kotlin
validate.request(req).withBody(User::class.java) { body ->
    // Note that body is deserialized as User!
    // Now you can do stuff. 
    // For example, lets echo the request as the response 
    ok().body(Mono.just(body))
}
```

### Validate a request (Kotlin + coroutines)

Or you can validate a request in a coroutine style,
using the `validate` instance created [above](#configure-kotlin):


without a body

```kotlin
validate.request(req) {
    // Do stuff e.g. return a list of names 
    ok().body(Mono.just(listOf("carmine", "alex", "eliana")))
}
```

with body

```kotlin
validate.request(req).withBody(User::class.java) { body ->
    // Note that body is deserialized as User!
    // Now you can do stuff. 
    // For example, lets echo the request as the response 
    ok().body(Mono.just(body))
}
```

## Usage (Java 8 _or greater_)

### Configure (Java)
This one-time configuration requires you to provide the _location of the openapi/swagger specification_ and an optional _custom error handler_.

```java
import io.github.cdimascio.openapi.Validate;
Validate<ValidationError> validate = Validate.configure("static/api.json")
```

with custom error handler

```java
class MyError {
    private String id;
    private  String messages;
    public MyError(String id, List<String> messages) {
        this.id = id;
        this.messages = messages;
    }
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }
    public List<String> getMessages() {
        return messages;
    }
    public void setMessages(List<String> messages) {
        this.messages = messages;
    }     
}
```

```java
Validate<ValidationError> validate = Validate.configure("static/api.json", (status, messages) ->
    new MyError(status.getName(), messages)
);
```

### Validate a request (Java)

Using the `validate` instance created above, you can now validate a request:

without a body

```java
ArrayList<String> users = new ArrayList<String>() {{
    add("carmine");
    add("alex");
    add("eliana");
}};

validate.request(null, () -> {
    // Do stuff e.g. return a list of user names
    ServerResponse.ok().body(fromObject(users));
});
```

with body

```java
validate
    .request(null)
    .withBody(User.class, user -> 
        // Note that body is deserialized as User!
        // Now you can do stuff. 
        // For example, lets echo the request as the response
        return ServerResponse.ok().body(fromObject(user))
    );
```

## Example Valiation Output

Let's assume a `POST` request to create a user requires the following request body:

```json
{
  "firstname": "carmine",
  "lastname": "dimasico"
}
```

Let's now assume an API user misspells `lastname` as `lastnam`

```shell
curl -X POST http://localhost:8080/api/users -H "Content-Type: application/json" -d'{ 
  "firstname": "c", 
  "lastnam": "d" 
}'
```

`openapi-spring-webflux-validator` automatically validates the request against a Swagger spect and returns:

```json
{
  "code": 400,
  "messages":[
	  "Object instance has properties which are not allowed by the schema: [\"lastnam\"]",
	  "Object has missing required properties ([\"lastname\"])"
  ]
} 
```

**Woah! Cool!!** :-D 

## Example

Let's say you have an endpoint `/users` that supports both `GET` and `POST` operations.

You can create those routes and validate them like so:

**Create the routes in a reactive or coroutine style:**

```kotlin
package myproject.controllers

import org.springframework.core.io.ClassPathResource
import org.springframework.http.MediaType.*
import org.springframework.web.reactive.function.server.ServerResponse.permanentRedirect
import org.springframework.web.reactive.function.server.coRouter
import org.springframework.web.reactive.function.server.plus
import org.springframework.web.reactive.function.server.router
import java.net.URI

class Routes(private val userHandler: UserHandler) {
    fun router() = router {
        "/api".nest {
            accept(APPLICATION_JSON).nest {
                POST("/users", userHandler::create)
            }
            accept(TEXT_EVENT_STREAM).nest {
                GET("/users", userHandler::findAll)
            }
        }
    } + coRouter { 
        "/coApi".nest {
            accept(APPLICATION_JSON).nest {
                POST("/users", userHandler::coCreate)
            }
            accept(TEXT_EVENT_STREAM).nest {
                GET("/users", userHandler::coFindAll)
            }
        }
    }
}
```

```kotlin
package myproject

import io.github.cdimascio.openapi.Validate
val validate = Validate.configure("static/api.yaml")
```

**Validate with openapi-spring-webflux-validator**

```kotlin
package myproject.controllers

import myproject.models.User
import myproject.validate
import org.springframework.web.reactive.function.server.ServerRequest
import org.springframework.web.reactive.function.server.ServerResponse
import org.springframework.web.reactive.function.server.ServerResponse.ok
import org.springframework.web.reactive.function.server.bodyValueAndAwait
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono

class UserHandler {

    fun findAll(req: ServerRequest): Mono<ServerResponse> {
        return validate.request(req) {
            ok().bodyValue(listOf("carmine", "alex", "eliana"))
        }
    }

    fun create(req: ServerRequest): Mono<ServerResponse> {
        return validate.request(req).withBody(User::class.java) {
            // it is the request body deserialized as User
            ok().bodyValue(it)
        }
    }

    suspend fun coFindAll(req: ServerRequest): ServerResponse {
        return validate.requestAndAwait(req) {
            ok().bodyValueAndAwait(listOf("carmine", "alex", "eliana"))
        }
    }

    suspend fun coCreate(req: ServerRequest): ServerResponse {
        return validate.request(req).awaitBody(User::class.java) {
            // it is the request body deserialized as User
            ok().bodyValueAndAwait(it)
        }
    }
}
```

## License

[Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0)

<a href="https://www.buymeacoffee.com/m97tA5c" target="_blank"><img src="https://bmc-cdn.nyc3.digitaloceanspaces.com/BMC-button-images/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: auto !important;width: auto !important;" ></a>

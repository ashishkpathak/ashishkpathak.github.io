
### Introduction

This article discusses the various challenges the microservice development team faced in integration testing. The integration chapter has been the core team for developing springboot based microservices, which form the backend of MyOptusApp. It has over hundred microservices featuring various capabilities from authentication, authorization to providing various features for managing customer services. Prepaid services involve complex user journeys, such as eSIM/USIM activation, which is a multi-step process. This process requires maintaining a user session throughout the journey. To preserve the context, it relies on Redis and PostgreSQL.


JUnit is the primary unit testing framework used by Java developers, for any integration testing it relies heavily on various mocking libraries, such as WireMock, embedded-jedis, and MockWebServer. These frameworks suffice for rudimentary testing, however for running complex test scenarios it requires testing against the real instances of the services. 


To address this challenge, our development team is adopting Testcontainers, a Java library that provides lightweight, disposable containers for running tests. It facilitates running and managing Docker images from Java code. Testcontainers allows us to spin up real instances of the services we depend on, such as Redis, PostgreSQL, and various AWS services, within Docker containers. 

-   Database Servers: Run real PostgreSQL/MySQL instances as Docker images.
-   Mocked HTTP Servers: Use Docker images from MockServer or WireMock.
-   Caches: Run real Redis instances as Docker images.
-   Messaging: Use Docker images for streams or queues like RabbitMQ.
-   Cloud Managed Services: Emulate AWS components like S3, Kinesis, and DynamoDB with Localstack.

This approach offers several advantages:

-   Realistic Testing Environment: By using actual instances of the services, we ensure that our tests run in an environment that closely mimics production. H2 database is not an equivalent server to test postgres stored procedures. This reduces the likelihood of encountering unexpected issues when deploying to production.

-   Isolation:  Each test can run in a clean, isolated environment, ensuring that tests are repeatable and not affected by previous test runs. A new instance can be spinned up for next set of test cases.

-   Ease of integration: Testcontainers simplifies the setup of complex environments, reducing the overhead associated with configuring and managing local instances of services. Its ease of integration with Springboot, advances its adoption.



### Scope
Additionaly benefit of having test containers is you can add them as dependencies and start them with docker, which is still better than docker-compose.

To implement an integration test, you need to run your application similarly to a unit test (method annotated by @Test).


First thing would be to include the test container dependencies.

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.8.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.17.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.17.3</version>
    <scope>test</scope>
</dependency>

```
In this example I'm using Postgres, but you can use any database as per your requirements. The following postgres driver dependency is added.

```xml
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.4.2</version>
    </dependency>
```

### Test updates

With Springboot, @SpringBootTest would load up the application context.  Jupiter (JUnit 5) integration is provided by means of the @Testcontainers annotation. The extension finds all fields that are annotated with @Container and calls their container lifecycle methods (methods on the Startable interface). Containers declared as static fields will be shared between test methods. They will be started only once before any test method is executed and stopped after the last test method has executed. Containers declared as instance fields will be started and stopped for every test method.

```java
@SpringBootTest
@Testcontainers
public class ApplicationIntegrationTests {


    @Containers
    private static PostgresSQLContainer container = new PostgresSQLContainer();

    @Autowired
    private ArtistDao artists;

    @Test
    public void test_all_artists_exists(){

            List<Artists> arts = artists.findAll();    
            Assert.assertTrue(all.size() == 6);
    }
}
    
```
While running integration testing, we are required to provide the hostname and port of the test database. This is generally configured as spring config


```properties
spring.datasource.username=db_username
spring.datasource.password=db_password
spring.datasource.database=db_name
spring.datasource.url=jdbc:postgresql://localhost:port

```
However the container can start on any random port on the host, so we need means to provide database name, username, password , host and port dynamically. Spring provides means to do this using @DynamicPropertySource annotation.

```java

  @DynamicPropertySource
  public static void overrideProperties(DynamicPropertyRegistry registry){
    registry.add("spring.datasource.username", container::getUsername);
    registry.add("spring.datasource.password", container::getPassword);
    registry.add("spring.datasource.database", container::getDatabaseName);
    registry.add("spring.datasource.url", container::getJdbcUrl);
  }
```



The above approach ensures that the database configuration is dynamically set based on the container's properties.




### Conclusion

TestContainers provide a robust and flexible solution for integration testing, addressing compatibility issues and ensuring consistent test environments across different developer setups. Note that this extension has only been tested with sequential test execution; using it with parallel test execution may have unintended side effects.
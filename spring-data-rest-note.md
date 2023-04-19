## Spring Data REST - Useful notes
Source/Author: https://gist.github.com/TomyJaya/31e48d81af0ca236496582d00192ef80
# How To change baseUri: 
in `application.properties` add: 

```properties
spring.data.rest.basePath=/api
```

- or - 

in `application.yml` add: 

```yaml
spring:
  data:
    rest: 
      basePath: /api
```

*WARNING*: 

1. Some SOs answer will suggest the deprecated `baseUri`. But, as of spring boot 1.5.1.RELEASE, it seems that `baseUri` is removed. 

2. in YAML, make sure you nest the properties otherwise it'll complain duplicate key: 

```
Exception in thread "main" while parsing MappingNode
 in 'reader', line 2, column 1:
    server:
    ^
Duplicate key: spring
 in 'reader', line 29, column 23:
    #      security: DEBUG
```
nesting example: 

```yaml
# JACKSON
spring:
  jackson:
    serialization:
      INDENT_OUTPUT: true
  # TOMY: below added by me    
  data:
    rest: 
      basePath: /api
```


# How to prevent spring data repository from being exported as REST?
```properties
import java.util.List;

import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(exported=false)
public interface PersonRepository extends MongoRepository<Person, String> {
}
```

# How to change path instead of the default pluralized name

```java
import java.util.List;

import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel = "people", path = "people")
public interface PersonRepository extends MongoRepository<Person, String> {
}
```


# How to Make REST Resource Repository read-only  (i.e. only support GET, HEAD, OPTIONS method)

Create and extend the below class. It uses annotation to surpresse `save` and `delete` from being exposed as REST APIs (remove POST and DELETE methods support). 
``` java
import java.io.Serializable;

import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.data.rest.core.annotation.RestResource;

@NoRepositoryBean
public interface GetOnlyMongoRestRepository<T, ID extends Serializable> extends MongoRepository<T, ID> {
    @Override
    @RestResource(exported = false)
    void delete(ID id);

    @Override
    @RestResource(exported = false)
    void delete(T entity);

    @Override
    @RestResource(exported = false)
    <S extends T> S save(S entity);
}
```

# How to project nested object (e.g. DBRef) 

For instance, you have a `Order` object with a nested orderer reference which is of type `Person`. 

``` java
public class Order {
    @Id
    private String id;

    private int quantity;
    private String productName;

    @DBRef
    private Person orderer;
    
    // usual getter and setter
}
```

You can create a projection such as the below: 

``` java
import org.springframework.data.rest.core.config.Projection;

@Projection(name = "inlineOrderer", types = { Order.class })
interface InlineOrderer {
    String getProductName();
    int getQuantity();
    Person getOrderer();
}

```

Now you can query your REST API using: `http://localhost:8080/api/orders/some-order-id?projection=inlineOrderer`

Additionally, if you want this projection to be default when listing the items in the collection, use the below: 

```java
@RepositoryRestResource(excerptProjection = InlineOrderer.class)
public interface OrderRepository extends GetOnlyMongoRestRepository<Order, String> {

}
```


# How to query by a certain attributes (search/ filter function)

In the `CrudRepository` interface, add the query method: 
```java
@RepositoryRestResource(collectionResourceRel = "people", path = "people")
public interface PersonRepository extends MongoRepository<Person, String> {

    List<Person> findByLastNameContainingIgnoreCase(@Param("name") String name);
    
}
```

then, you can access it using: 
`http://localhost:8080/api/people/search/findByLastNameContainingIgnoreCase?name=ang`

You can alias it: 
```java
@RepositoryRestResource(collectionResourceRel = "people", path = "people")
public interface PersonRepository extends MongoRepository<Person, String> {

    @RestResource(path="searchByLastName", rel="searchByLastName")
    List<Person> findByLastNameContainingIgnoreCase(@Param("name") String name);
    
}
```
then, you can access it using alias: 
`http://localhost:8080/api/people/search/searchByLastName?name=ang`

Note:

* *Containing* does \*SEARCH_STRING\*   
* *IgnoreCase* does case-insensitive search

Refer to the below link for all query methods: 
http://docs.spring.io/spring-data/jpa/docs/1.5.1.RELEASE/reference/html/jpa.repositories.html#jpa.query-methods.query-creation


# How to query multiple fields

```java
@RepositoryRestResource(collectionResourceRel = "people", path = "people")
public interface PersonRepository extends MongoRepository<Person, String> {

    // Find multiple fields
    List<Person> findByLastNameOrFirstNameContainingAllIgnoreCase(@Param("name") String firstName,
            @Param("name") String lastName);

}
```

`http://localhost:8080/api/people/search/findByLastNameOrFirstNameContainingAllIgnoreCase?name=test`

Note:

* *All* makes both FirstName and LastName ignore case \*<searchString>\*   
* *Or* joins the 2 criteria


# How to query multiple TextIndexed fields

Set up the index in the POJO:

```java
import org.springframework.data.mongodb.core.index.TextIndexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Document
public class Person {

    @TextIndexed
    private String aboutMe;
    @TextIndexed
    private String address;
    
    // ...
}
```

*NOTES*: 

* Don't forget to annotate `@Document`. Without it, somehow the text index is *not* created. 
* there can only be one text index per collection in MongoDB. So all the fields annotated with @TextIndexed will be combined into one. 
* You can assign weight to the text index: http://docs.spring.io/spring-data/mongodb/docs/current/api/org/springframework/data/mongodb/core/index/TextIndexed.html


Modify the `MongoRepository` to add the method to `findAllBy`:
```java
import org.springframework.data.mongodb.core.query.TextCriteria;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.repository.query.Param;

public interface PersonRepository extends MongoRepository<Person, String> {

    // Find in indexed text
    List<Person> findAllBy(@Param("criteria") TextCriteria criteria);
   
}
```

Since Spring data rest doesn't have the default converter from `String` to `TextCriteria`, create a `RestController` wrapper to invoke this: 

```java
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.query.TextCriteria;
import org.springframework.data.repository.query.Param;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/people")
public class PersonController {

    private static final Logger LOG = LoggerFactory.getLogger(PersonController.class);

    @Autowired
    private PersonRepository personRepository;

    @RequestMapping(value = "search", method = RequestMethod.GET)
    public List<Person> search(@Param("criteria") String criteria) {
        LOG.debug("Search invoked for criteria {}", criteria);
        return personRepository.findAllBy(TextCriteria.forDefaultLanguage().matchingAny(criteria));
    }
}
```


You can now access it here: 

`http://localhost:8080/people/search?criteria=happy`


# How to page the results

E.g. page every 2 results. Access it here: 

`http://localhost:8080/api/orders?size=2`

In addition to the normal results, there will be `_links` property to navigate to the next pages.



# How to sort 

E.g. sort by quantity descending. Access it here: 

`http://localhost:8080/api/orders?sort=quantity,desc`

NOTE: 

* Use comma to separate field name is sort direction

# How to serialize java 8 LocalDate nicely in spring data rest APIs

Add the below maven dependency: 

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

Add the below line in `application.properties`:

```properties
spring.jackson.serialization.write_dates_as_timestamps=false
```

LocalDate will be serialized as `yyyy-MM-dd`. 

Resource: http://stackoverflow.com/questions/29956175/json-java-8-localdatetime-format-in-spring-boot

# Two-way `@DBRef` reference 
Basically, if you have a `User` which has an `Address` and you want the `Address` to have a back pointer to the `User`, Do *NOT* use two-way `@DBRef`! When spring-data-rest serializes the object, you'll run into a infinite recursion and possibly get the below exception: 
`java.lang.IllegalStateException: getOutputStream() has already been called for this response`


# How to generate sequence ID for Spring Data Mongodb
Follow the instructions here: http://stackoverflow.com/questions/36448921/how-we-create-autogenerated-field-for-mongodb-using-springboot

# How to index a field
Just annotate it with `@Indexed`. But remember to annotate the POJO with `@Document` for MongoDB to scan the metadata mapping. E.g. 

```java
@Document
public class Person {

  @Id
  private ObjectId id;
  @Indexed
  private Integer ssn;
```

# How to handle LocalDate in Controller
Annotate the request parameter with `@DateTimeFormat(iso = DateTimeFormat.ISO.DATE)`

DateTimeFormat.ISO.DATE is yyyy-MM-dd format. 

```java
   @RequestMapping(value = "/protected/getPastUserOrders", method = RequestMethod.GET)
    public List<Order> getPastUserOrders(@RequestParam("userId") String userId, @RequestParam("date") @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate date)  throws IOException {
        return orderRepository.findByUser_FacebookIdAndDateBefore(userId, date);
    }
```

# How to pass deserialize JSON object as a java `Map`? 
Just declare the type as Map<K,V>: 
``` java
private Map<String, Boolean> daysOpen;
```

# How to read file in ClassPath to String (One-liner) 
1. First, don't use `ClassPathResource::getFile`. This doesn't work when your file is bundled in a jar. (i.e. only works in an IDE). 
2. Hence, use `ClassPathResource::getInputStream`
3. How to easily convert `InputStream` to `String`? Use `IOUtils`

Sample: 
```java
String templateString = IOUtils.toString(new ClassPathResource("templates/order-receipt.html").getInputStream(), "UTF-8");
```

References: 
* http://stackoverflow.com/questions/25869428/classpath-resource-not-found-when-running-as-jar
* http://stackoverflow.com/questions/309424/read-convert-an-inputstream-to-a-string
1

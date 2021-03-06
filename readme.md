##USING MYSQL IN SPRING BOOT VIA SPRING DATA JPA AND HIBERNATE

[Source](http://blog.netgloo.com/2014/10/27/using-mysql-in-spring-boot-via-spring-data-jpa-and-hibernate/ "Permalink to Using MySQL in Spring Boot via Spring Data JPA and Hibernate")

# Using MySQL in Spring Boot via Spring Data JPA and Hibernate

This post shows how to use a **MySQL** database in a **Spring Boot web application**, using less code and configurations as possible, with the aim to take full advantage from Spring Boot.  
**Spring Data JPA** and **Hibernate** (as JPA implementation) will be used to implement the data access layer.

## Dependencies

Be sure to have following dependencies in the `pom.xml` file:

    
      
        org.springframework.boot
        spring-boot-starter-web
      
      
        org.springframework.boot
        spring-boot-starter-data-jpa
      
      
        mysql
        mysql-connector-java
      
    

See [here][1] an example of a whole `pom.xml`.

## Configuration file

Put in the [`application.properties`][2] file pretty much all the configurations:

    src/main/resources/application.properties

    # DataSource settings: set here your own configurations for the database
    # connection. In this example we have "netgloo_blog" as database name and
    # "root" as username and password.
    spring.datasource.url = jdbc:mysql://localhost:8889/netgloo_blog
    spring.datasource.username = root
    spring.datasource.password = root

    # Keep the connection alive if idle for a long time (needed in production)
    spring.datasource.testWhileIdle = true
    spring.datasource.validationQuery = SELECT 1

    # Show or not log for each sql query
    spring.jpa.show-sql = true

    # Hibernate ddl auto (create, create-drop, update)
    spring.jpa.hibernate.ddl-auto = update

    # Naming strategy
    spring.jpa.hibernate.naming-strategy = org.hibernate.cfg.ImprovedNamingStrategy

    # Use spring.jpa.properties.* for Hibernate native properties (the prefix is
    # stripped before adding them to the entity manager)

    # The SQL dialect makes Hibernate generate better SQL for the chosen database
    spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQL5Dialect

No xml or java config classes are needed.

Using the hibernate configuration `ddl-auto = update` the database schema will be automatically created (and updated), creating tables and columns, accordingly to java entities found in the project.

See [here][3] for other hibernate specific configurations.

## Create an entity

Create an entity class representing a table in your db.

In this example we create an entity `User` composed by three fields: `id`, `email` and `name`.  
An object of this class will be an entry in the `users` table in your MySQL database.

```java
    src/main/java/netgloo/models/User.java

    package netgloo.models;

    // Imports ...

    @Entity
    @Table(name = "users")
    public class User {
    @GenericGenerator(name = "table-hilo-generator", strategy = "org.hibernate.id.TableHiLoGenerator",
            parameters = {@Parameter(value = "hibernate_id_generation", name = "table")})

    @Id
    @GeneratedValue(generator = "table-hilo-generator")
      private long id;

      @NotNull
      private String email;

      @NotNull
      private String name;

      // Public methods

      public User() { }

      public User(long id) {
        this.id = id;
      }

      public User(String email, String name) {
        this.email = email;
        this.name = name;
      }

      // Getter and setter methods
      // ...

    }
```

The `Entity` annotation mark this class as a JPA entity. The `Table` annotation specifies the db table's name (would be "User" as default).

## The Data Access Object

A [**DAO**][4] (aka Repository) is needed to works with entities in database's table, with methods like _save_, _delete_, _update_, etc.

With Spring Data JPA a DAO for your entity is simply created by extending the `CrudRepository` interface provided by Spring. The following methods are some of the ones available from such interface: `save`, `delete`, `deleteAll`, `findOne` and `findAll`.  
The _magic_ is that such methods must **not be implemented**, and moreover it is possible to create new query methods working only by their signature definition!

Here there is the Dao class `UserDao` for our entity `User`:

```java
    src/main/java/netgloo/models/UserDao.java

    package netgloo.models;

    // Imports ...

    @Transactional
    public interface UserDao extends CrudRepository {

      /**
       * This method will find an User instance in the database by its email.
       * Note that this method is not implemented and its working code will be
       * automagically generated from its signature by Spring Data JPA.
       */
      public User findByEmail(String email);

    }
```

See [here][5] for more details on how to create query from method names.

## A controller for testing

That's all! The connection with the database is done. Now we can test it.

In the same way as in some similar previous post ([one][6] and [two][7]) we create a controller class named [UserController][8] to test interactions with the MySQL database using the UserDao class.

```java
    src/main/java/netgloo/controllers/UserController.java

    package netgloo.controllers;

    // Imports ...

    @Controller
    public class UserController {

      /**
       * GET /create  --> Create a new user and save it in the database.
       */
      @RequestMapping("/create")
      @ResponseBody
      public String create(String email, String name) {
        String userId = "";
        try {
          User user = new User(email, name);
          userDao.save(user);
          userId = String.valueOf(user.getId());
        }
        catch (Exception ex) {
          return "Error creating the user: " + ex.toString();
        }
        return "User succesfully created with id = " + userId;
      }

      /**
       * GET /delete  --> Delete the user having the passed id.
       */
      @RequestMapping("/delete")
      @ResponseBody
      public String delete(long id) {
        try {
          User user = new User(id);
          userDao.delete(user);
        }
        catch (Exception ex) {
          return "Error deleting the user:" + ex.toString();
        }
        return "User succesfully deleted!";
      }

      /**
       * GET /get-by-email  --> Return the id for the user having the passed
       * email.
       */
      @RequestMapping("/get-by-email")
      @ResponseBody
      public String getByEmail(String email) {
        String userId = "";
        try {
          User user = userDao.findByEmail(email);
          userId = String.valueOf(user.getId());
        }
        catch (Exception ex) {
          return "User not found";
        }
        return "The user id is: " + userId;
      }

      /**
       * GET /update  --> Update the email and the name for the user in the
       * database having the passed id.
       */
      @RequestMapping("/update")
      @ResponseBody
      public String updateUser(long id, String email, String name) {
        try {
          User user = userDao.findOne(id);
          user.setEmail(email);
          user.setName(name);
          userDao.save(user);
        }
        catch (Exception ex) {
          return "Error updating the user: " + ex.toString();
        }
        return "User succesfully updated!";
      }

      // Private fields

      @Autowired
      private UserDao userDao;

    }
```

Test the controller launching the Spring Boot web application and using these urls:

* /create?email=[email]&name=[name]: create a new user with an auto-generated id and email and name as passed values.
* /delete?id=[id]: delete the user with the passed id.
* /get-by-email?email=[email]: retrieve the id for the user with the given email address.
* /update?id=[id]&email=[email]&name=[name]: update the email and the name for the user identified by the given id.





## References

[1]: https://github.com/netgloo/netgloo-blog/blob/master/spring-boot-mysql-springdatajpa-hibernate/pom.xml
[2]: https://github.com/netgloo/netgloo-blog/blob/master/spring-boot-mysql-springdatajpa-hibernate/src/main/resources/application.properties
[3]: https://docs.jboss.org/hibernate/orm/4.3/manual/en-US/html/ch03.html#configuration-optional
[4]: http://stackoverflow.com/a/19154487
[5]: http://docs.spring.io/spring-data/data-jpa/docs/current/reference/html/#jpa.query-methods.query-creation
[6]: http://blog.netgloo.com/2014/10/06/spring-boot-data-access-with-jpa-hibernate-and-mysql/
[7]: http://blog.netgloo.com/2014/08/17/use-mysql-database-in-a-spring-boot-web-application-through-hibernate/
[8]: https://github.com/netgloo/netgloo-blog/blob/master/spring-boot-mysql-springdatajpa-hibernate/src/main/java/netgloo/controllers/UserController.java
  
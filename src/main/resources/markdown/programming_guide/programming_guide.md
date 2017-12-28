<h1><p style="text-align: center;">Backend Programming Guide</p></h1>

Table of content:
[TOC levels=2-4]

## Overview
This document introduce technical detail about backend programming. It introduce how to implement each
type of technical components and coding standard. Developers who are new to this project should go through
this document first before doing any backend service development tasks.


## Components For Tasks 

When developer want to create a new backend service for new functions, below types of technical components 
will be touched:

* Controllers - Controllers are used to expose HTTP REST endpoints that can be accessed by other services or frontend
program. It implement the C layer of MVC pattern. Usually, it call Services to conduct transactions or retrieve data.
It may transform the data that is retrieved from Services into a format that can be understood by the caller 
(such as an browser based frontend application). Usually, Controllers accept request or generate response in JSON
 format(except for the scenarios that need to handle binary data. Such as file upload, generate PDF).
* Services - Services are used to implement business logic. It call other Services, Repositories or DAOs to manipulate 
data in DB. Transaction control should be done in this layer. 
* Repositories - Spring data is used in this project. Compare with traditional DAOs, Spring data can save effort if
developer just need some simple DB access method. For example, Spring data can provide repositories with default 
CRUD method for one entity. Developer do not need to code any implementation. Spring data
also can generate implementation for simple query method for one entity by parsing method names. Developer just need 
to define that method in repositories interface. Spring data repositories only can be used to do DB access for one
entity only.
* DAOs - In some scenarios, complicated DB access is needed for some reasons. For example, some query functions may 
have performance issue if "join table" is not used. In these scenarios, DAOs will be used. DAOs are used to handle the 
scenario that need cross-table or cross-entity DB access.
* Entities - JPA(Implemented with Hibernate) is used for DB access in this project. All DB access should be done through 
JPA entity except for some special scenarios. For example, an legacy store procedure must be reused becauuse it is too 
expensive to rebuild it in java. In this case, developer should call JPA EntityManager to access that store procedure. 
Developer should never by pass JPA(anyway, multiple technologies for one purpose is always not good for application 
maintenance) until something really cannot be done with JPA or it will cause other issues.
Please note that, Services, Repositories, DAOs and Entities are belong to the M layer of MVC pattern.
* Service Forms - In some cases, there are many input fields for one transaction. For example, 10 fields need to be 
input to create an order. It is not a good practise to define those 10 fields as parameters for methods in Controllers
and Services. An Service Form should be defined for these 10 fields and use this Service Form as parameters for methods
in Controllers or Services. Basically, Service Forms are values objects for passing values only. It should not include 
any logic. Service Forms are used to contain JSON data that is parsed from body in a PUT/POST HTTP requests as well
 (please see 
[@RequestBody usage](https://docs.spring.io/spring/docs/4.3.10.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#mvc-ann-requestbody))  
. In this case, the fields definition in Service Forms should match the format of JSON from frontend program. So, 
Service Forms are used to defined the interface between frontend and backend programs.
* Views - It is similar to Service Forms, Views are values objects and they are used to defined the interface
between frontend and backend. Different from Service Forms that are used as input for Controllers or Services
, Views are used as output. For example, there is a complicated query function need to retrieve 10 fields from 3 tables.
An View class that include 10 fields. Services should call Repositories/DAOs to retrieve data to fill that View object
and return it to Controllers. Controllers will return this View object with annotation 
[@ResponseBody](https://docs.spring.io/spring/docs/4.3.10.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#mvc-ann-responsebody)   
to generate JSON to response frontend program. Basically, Views are used to show data in screens and Service Forms are 
used to submit data. Sometimes, the no of fields shown in screens are more than the no of fields that will be 
submitted in the same screen. For example, in an user editing screen, the role for a user can be choosed. 
The screen need to show the role code and role description. However, when submit button is clicked, only role code 
need to be submitted. In this case, the View class may extend the Service Form class.
* Feign Clients - In micro-service architecture, backend services are deployed into different processes so that cross 
processes communication is needed. Besides the cross processes communication happen in production env, the API testing
(subcutaneous test) also need cross processes communication. The reason is the test case is deployed in an separated process. 
 They communicate with the APIs that are being tested by sending HTTP request(simulate the behaviour of frontend 
 program). In this project, Feign Clients are used for this kind of communication. With Feign Client, developer do not
 need to construct the HTTP request manually which may cause typo mistake easily and is not refactoring friendly.
Developer just need to make sure the methods definitions is exactly the same as the ones in 
Controllers in target process. Then, the Feign Clients can construct proper HTTP request to call the target REST API. 
Besides used as an remote communication channel, Feign Clients also provide load balancing features with 
[Ribbon](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-ribbon).
For simple application, the IP/ports for remote processes can be written in property file. When more and more processes
are involved, it is difficult to use property file to maintain the IP/ports list. To handle this case, Feign Clients 
also can be integrated with 
[Eureka](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-eureka-server) 
for service registration which is widely used in micro-service architecture. 


When an HTTP request is sent to a backend service, this request is handled as below:
![components sequence diagram](components_seq.svg)

**Note**: Because Interceptors are not touched for tasks usually, they are not mentioned in the list above. However,
to illustrate the overall process, it is mentioned in the diagram above.

Below diagram illustrate the relationship between the components mentioned above:
![components class diagram](components_class.svg)

 
### Controllers
Controllers are used to provide HTTP endpoint so that services can be access by components in other processes. 
Most of the Controllers provide JSON endpoints. For these Controllers, `@RestController` should be placed before the 
class name. If `@RestController` is not used but just use `@ResponseBody` and `@Controller`, the JSON endpoint still
work. However, the exception handling will not be done properly because the globally exception handler for JSON endpoints
use `@RestController` to indicate which Controllers are under its control.

For each methods in Controllers that are used to provide HTTP endpoints, `@RequestMapping` must be used. The url for
the HTTP endpoint can be specified in `@RequestMapping`. The naming convention of url should be 
"/{an noun or an noun phrase}/{an verb phrase}".


For Controllers methods that do query only, GET request method should be used. For Controllers methods that are used 
to submitted data(such as
submit a form), POST request method should be used. For Controllers methods that are used to delete something(such as,
inactive an account), DELETE request method should be used.   

If only simple data (such as, there are only 2 fields) is passed to the HTTP endpoint as input, HTTP request parameters
should be used. `@RequestParam` should used to map the HTTP request parameters to java method parameters. 
If compliated data (such as, complicated form data that is in tree structure ) is passed to the HTTP endpoint as input,
the data should be passed through HTTP request body. `@RequestBody` should be used to map the data in HTTP request body
to an ServiceForm object. There are no request body for GET/DELETE request, it is impossible to use `@RequestBody` 
for the endpoints with GET/DELETE request methods. If an HTTP endpoint really need to use `@RequestBody` for some 
reasons, this endpoint should use POST request method. For example, there is an endpoint used to inactive an account. 
Base on the rule above, DELETE request method should be used. However, inactive an account is not that simple. User need
to key in a complicated form for audit reason(such as justify why this account should be inactive). It is trouble to
handle this form data without `@RequestBody`. In this case, this endpoint should used `@RequestBody` and POST request
method.

Below are some example:
```java
    //method that do query. Please note that, the value of @RequestParam should be specified. It is not an good practise
    //to let Spring MVC to guess the value for @RequestParam base on the java method parameter name since it is not
    //refracting friendly. 
    // View is used as returned object to generate JSON here.
    @RequestMapping(value = "/user/queryUser", method = RequestMethod.GET)
    public List<UserQueryView> queryUser(@RequestParam("userName") String userName, 
                                         @RequestParam("userDesc") String userDesc) {
        //...
    }
     
    //method that accept form data to create an user in system. @RequestBody is used here.
    //ServiceForm object is used as method parameter here.
    @RequestMapping(value = "/user/createUser", method = RequestMethod.POST)
    public Boolean createUser(@RequestBody UserServiceForm user) {
        //...
    }
    
    //delete an user. Because only one field is needed, @RequestParam and DELETE request method is used here.
    @RequestMapping(value = "/user/deleteUser", method = RequestMethod.DELETE)
    public Boolean deleteUser(@RequestParam("userName") String userName) {
            //...
    }
    
    //Inactive an account. Since complicated input data is needed. @RequestBody and POST request method is used here.
    @RequestMapping(value = "/account/inactiveAccount", method = RequestMethod.POST)
    public Boolean deleteUser(@RequestBody InactiveAccountServiceForm serviceForm) {
            //...
    }
    
 ```
 
In some cases, non-JSON HTTP endpoints are needed. Such as, upload a file, download PDF from server side. In these
cases, `@Controller` should be used to create Controllers classes. `@ResponseBody` should not be used as well because
it will make the endpoint generate JSON response.

For security reason, `@PreAuthorize` should be used at the beginning of every Controllers methods. This annotation
is used to tell Spring Security framework what access right that is needed to access this endpoint.
For the endpoints that only can be access by authorized users, `@PreAuthorize` should be used as below:
```java
    //usually, one endpoint is mapped to one function point.
    @RequestMapping(value = "/user/queryAllUsers", method = RequestMethod.GET)
    @PreAuthorize("hasPermission(this, '"+FunctionPointConstant.QUERY_ALL_USERS+"')")
    public List<User> queryAllUsers() {
        //...
    }
    
    //if an endpoint can be access with multiple function point, they can be combined with "and"/"or"
    @RequestMapping(value = "/user/queryAllUsers", method = RequestMethod.GET)
    @PreAuthorize("hasPermission(this, '"+FunctionPointConstant.QUERY_ALL_USERS+"') or hasPermission(this, '"+FunctionPointConstant.TEAM_LEAD+"')")
    public List<User> queryAllUsers() {
        //...
    }    
```
The second parameter of "hasPermission" is the name of function point. Please refer "Security" section about how to
config function point and assign them to users. 

The default setting is that all HTTP endpoints cannot be access until user is authenticated. This is used to make sure 
no HTTP endpoint should be protected but it can be access by everyone just because developer forget to assign 
`@PreAuthorize` annotation to it. So, if `@PreAuthorize` is not specified for a Controllers method, that means all 
authenticated user can access it. If there is any HTTP endpoint should be access without authentication, the url should 
be in pattern `/pub/**/*`. If this endpoint cannot use this url pattern, the Spring Security config must be changed to
add new url pattern for endpoints without any security checking.

Please refer [MVC controller](https://docs.spring.io/spring/docs/4.3.7.RELEASE/spring-framework-reference/htmlsingle/#mvc-controller)
for more detail info.

### Services
Services are used to implement business logic. Usually, it call Repositories or DAOs to retrieve data and manipulate 
data inside it. This layer should be the most complicated part of backend program because most of the logic is in this 
layer.

To define a Service class, `@Service` annotation should be placed at the beginning of the class. Because Services need 
to deal with transaction, `@ReadOnlyTransaction` or `@ReadWriteTransaction` should be used for methods that need 
 transaction. 
 
If a method only read data
 from DB, `@ReadOnlyTransaction` should be used. Actually, the `@ReadOnlyTransaction` will let the DB konw current 
 transaction is readonly and the isolation level is READ_UNCOMMITTED. These setting are more optimized for readonly 
 DB access performance.
  
For methods that will write data to DB, `@ReadWriteTransaction` should be used. In this annotation, the isolation level 
is READ_COMMITTED so that optimistic locking must be used used in Entities. 

Both `@ReadWriteTransaction` and `@ReadOnlyTransaction` provide `propagation` attribute. The default value is 
REQUIRED. That means, when current method is called, if there is no transaction, start an new transaction. If a 
transaction has been started, just join the existing transaction. This setting fit the requirement for most of the 
cases. However, in some cases, this setting is not good for performance. For example, there are 10 thousands of records
need to be handled in a batch program. If all those records are handled in one transaction, the performance is very 
bad. Maybe, the transaction will failed because the DB transaction log if full. A better solution is that every 
transaction only handle 1 record. In this case, propagation REQUIRE_NEW should be used. 

Please refer 
[isloation level](https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html#transactions_data_integrity)
and
[transaction propagation](https://docs.spring.io/spring/docs/4.3.7.RELEASE/spring-framework-reference/htmlsingle/#tx-propagation) 
for more detail info.

### Repositories
Repositories are used to implement simple DB access method. In this project, Spring Data JPA is used to implement 
Repositories. To create an Repository, developer do not need to code any method implementation just like what they 
 did for traditional DAO approach but just define an interface as below steps:

* Define an interface that extend `JpaRepository`
* The `JpaRepository` has 2 type parameters. The first one should be the Entity that this Repository should handle. 
Actually, the Repository only can access data about the Entity that is specified here. The second type parameter is 
the primary key type of the Entity. Usually, it is object type for primitive data type. Such as `java.lang.Spring`, 
`java.lang.Long`, etc. If the Entity use composite key, the customized composite data type must be specified here.
Please refer [composite key](http://www.objectdb.com/java/jpa/entity/id#Composite_Primary_Key_) for more detail info 
about composite key of Entity.

Below is an example:  
```java
   
    public interface UserRepository extends JpaRepository<User, String> {
    }
```
In the example above, even though the interface is empty, it is an Repository that already provide full CRUD method
for User Entity because it extend `JpaRepository`. Please read the source of `JpaRepository` to find the CRUD methods
it provide.

Please refer [query methods](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods)
for more detail about how to query data with repositories.

### DAOs
Sometimes, developer have to access multple entity or table in one QL statement. Otherwise, it will cause performance
issue. For example, there is a screen need to show a table list. There are many records may be shown in the list and
 data for each record come from 6 tables. If "JOIN TABLE" is not used, performance issue will be cause. In such cases,
 DAOs should be used.

To create a DAO, please follow below steps:

* Define a class extend `BaseJpaDao`. The `BaseJpaDao` define `EntityManager` which can be used to access JPA API and
some util methods that are convenient for DB access.
* Place `@Repository` at the DAO class that just created. This is a little confused because DAO class use annotation
`@Repository`. If you check the java doc for `@Repository`, you will find that Spring framework expect DDD-style
repositories use `@Repository`. However, Spring framework also suggest DAO use this annotation. Otherwise, the
data access exception translation will not work. This is why `@Repository` is still used here.
* Define a set method for `this.entityManger`. `@Autowire` can be used for this set method so that Spring container will
 inject an instance of `EntityManager`. If there are multple instance of `EntityManager` in the Spring container,
 `@Qualifier` should be used as well or developer can just use java config approach to setup the `this.entityManager`.
 Anyway, the `this.entityManager` cannot be null for DAOs classes. Below is an example:
 ```java
     @Repository
     public class UserDao extends BaseJpaDao {

         @Autowired
         public void setEntityManager(EntityManager entityManager) {
             this.entityManager = entityManager;
         }

         public List<UserWithRole> queryUsersWithRoles(String userName, String userDesc) {
             //prepare the QL.
             String ql =  ...;

             Query query = this.entityManager.createQuery(al);
             //prepare parameters
             Map<String, Object> params = new HashMap<>();
             ...
             //insert parameter into query object.
             DbAccessUtil.setQueryParameter(query, params);

             List rawResultList = query.getResultList();

             //define a mapper object that convert the raw result to expected data type.
             Function<Object[], UserWithRole> mapper = new Function<Object[], UserWithRole>() {

                 @Override
                 public UserWithRole apply(Object[] row) {

                     UserWithRole view = new UserWithRole();

                     //convert the raw data record to UserWithRole.

                      return view;
                 }

             };

             List<UserWithRole> resultList = DbAccessUtil.convertRawListToTargetList(rawResultList, mapper);

             return resultList;
         }
     }
 ```

### Entities
For every tables, Entity must be created because Entities are not only used for DB access but also used for DB design.
Developer should design the DB structure by editing the source of Entities. Then, use source of Entities to generate
DB design and deployment documents.

Please follow below steps to create an Entity:

* Create an value object java class. That means there are only fields and get/set methods in this class.
* The Entity class should extend `BaseEntity`. This base class provide fields for audit trail and version number for
optimistic locking.
* Put `@Entity` and `@Table` annotations at the beginning of this class.
* In `@Table`, specify the table name.
* In `@Table`, use `@Index` to define indexes for this table. Actually, JPA do not need this info for DB access. It is
used to generate liquibase deployment file only
* Put `@Column` at the get method of a field to specify the column name for that field. Please also specify the value of
`nullable`, `columnDefinition`, `length`(for string field), `precision` and `scale`(for decimal field) attributes.
They are used for liquibase deployment file generation. If `columnDefinition` attribute is not specified, the build
 tool will determine the column data type base on the data type of returned value of the get method. For example,
 if the get method return `java.lang.String`, the column data type in the generated liquibase file will be
 `java.sql.Types.VARCHAR` and the data type in DB will be `VARCHAR`. For some cases that need to override this default
data type, please do specify the value of `columnDefinition`. For example, some string field need NVARCHAR in DB,
  please sepcify `java.sql.Types.NVARCHAR` in `columnDefinition`. If the DB design need DB specified data type,
 developer also can specify DB specified data type in this attribute. For example, in Oracle, `XMLType` can be used
 to store XML but this data type cannot be mapped to any type defined in `java.sql.Types`. In this case, developer
 can just put `XMLType` in `columnDefinition` and the build tool will not do any translation when it generate the
 liquibase files. It is not recommended to use DB specified data type in DB design because H2 will be used for
 DB UT and this kind of DB specified data type may cannot be deployed to H2 memory DB.
* Put `@Id` at the get method for primary key field. If there are multiple fields for primary key, please refer
(composite key)[http://www.objectdb.com/java/jpa/entity/id#Composite_Primary_Key_]. Embedded primary key
should be used. Please note that
the `equals()` and `hashcode()` method of the ID class must be implemented properly because the Hibernate
 will use these 2 methods to check objects in first level cache. Please use `java.util.Objects` should be used
 to implement these 2 methods. Below is an example(they are generated by IDEA):
```java

@Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(userName, user.userName) &&
                Objects.equals(userDescription, user.userDescription);
    }

    @Override
    public int hashCode() {
        return Objects.hash(userName, userDescription);
    }
```
For composite ID, below is an example:
```java
    //in the entity, define a field for the primary key.
    @EmbeddedId
    public UserToUserRoleMapKey getPk() {
        return primaryKey;
    }

    public void setPk(UserToUserRoleMapKey primaryKey) {
        this.primaryKey = primaryKey;
    }

    //then define the composite key class
    @Embeddable
    public static class UserToUserRoleMapKey implements Serializable {

        private String userName;

        private String roleCode;


        @Column(name = "USER_NAME", unique = true, nullable = false, length = 100)
        public String getUserName() {
            return userName;
        }

        public void setUserName(String userName) {
            this.userName = userName;
        }


        @Column(name = "ROLE_CODE", unique = true, nullable = false, length = 100)
        public String getRoleCode() {
            return roleCode;
        }

        public void setRoleCode(String roleCode) {
            this.roleCode = roleCode;
        }

        @Override
        public boolean equals(Object obj) {
            if (this == obj) {
                return true;
            }

            if (obj == null || getClass() != obj.getClass()) {
                return false;
            }

            UserToUserRoleMapKey that = (UserToUserRoleMapKey) obj;
            return Objects.equals(userName, that.userName)
                   && Objects.equals(roleCode, that.roleCode);
        }

        @Override
        public int hashCode() {
            return Objects.hash(userName, roleCode);
        }
    }


```

* If an Entity can be linked to another Entity, create an association get method and put `@ManyToMany`,
`@ManyToOne`, `@OneToOne` or `@OneToMany` annotation at it. Please refer
(relationship)[http://www.objectdb.com/api/java/jpa/annotations/relationship] for more detail. When, association
annotation is used, make sure the `cascade` attribute is empty(no entity will be persist until developer make it
explicitly) and `fetch` attribute is LAZY(no unexpected DB query will be done automatically.). That means, when
`@OneToOne` and `@ManyToOne` is used, the `fetch` attribute must be set to LAZY explicitly. 

* Assume there are 2 entities, User and UserGroup. One UserGroup may contain many users. The User entity should include
below source
```java
    @Column(name = "GROUP_CODE", length = 100)
    public String getGroupCode() {
        return groupCode;
    }

    public void setGroupCode(String groupCode) {
        this.groupCode = groupCode;
    }

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "GROUP_CODE", insertable = false, updatable = false)
    public UserGroup getGroup() {
        return group;
    }
    
    public void setGroup(UserGroup group) {
        this.group = group;
    }
```
That means, if developer call setGroup(), nothing in DB will be changed. Developer need to call setGroupCode() to 
change UserGroup for an User. The reason for this design is that when developer do problem diagnosis, they still need
to understand the DB structure and understand what are the correct values for each fields. To keep things simple, we
try to use JPA as mapper that only map fields from java object fields to DB fields. Association mapping are defined for
 documentation(auto ER diagram generation) or query purpose only.

* Assume there are 2 entities, User and UserRole. They are in many to many relationship. To save the relationship data,
 association mapping should not be used. Please create an "link entity" and use this "link entity" to save
  relationship. The reason for this design is if use assocatiation mapping field to save the relationship, some audit
trail (such as created by, last modified by, last modified time) cannot be saved for the relationship modification.
Furthermore, the build tool also cannot generate the DB deployment script properly if "link entity" is not defined.
Below is an example:
```java

@Entity()
@Table(name = "USER_TO_USER_ROLE_MAP",
        //need to define indexes as below, otherwise, the query performance will be bad when developer
        //want to do an query with "user join user role" or "user role join user".
        indexes = {@Index(name = "USER_TO_USER_ROLE_INDEX_1", columnList = "USER_NAME ASC", unique = false),
                @Index(name = "USER_TO_USER_ROLE_INDEX_2", columnList = "ROLE_CODE", unique = false)})
public class UserToUserRoleMap extends BaseEntity {

    private UserToUserRoleMapKey primaryKey;

    private User user;

    private UserRole userRole;

    @EmbeddedId
    public UserToUserRoleMapKey getPk() {
        return primaryKey;
    }

    public void setPk(UserToUserRoleMapKey primaryKey) {
        this.primaryKey = primaryKey;
    }

    //delcare this link entity can link to User entity.
    @ManyToOne(optional = false)
    @JoinColumn(name = "USER_NAME", insertable = false, updatable = false)
    public User getUser() {
        return user;
    }

    //delcare this link entity can link to UserRole entity. So, User can link to UserRole
    //with many to many relationship.
    @ManyToOne(optional = false)
    @JoinColumn(name = "ROLE_CODE", insertable = false, updatable = false)
    public UserRole getUserRole() {
        return userRole;
    }

    public void setUser(User user) {
        this.user = user;
    }

    public void setUserRole(UserRole userRole) {
        this.userRole = userRole;
    }

    @Embeddable
    public static class UserToUserRoleMapKey implements Serializable {

        private String userName;

        private String roleCode;


        @Column(name = "USER_NAME", unique = true, nullable = false, length = 100)
        public String getUserName() {
            return userName;
        }

        public void setUserName(String userName) {
            this.userName = userName;
        }


        @Column(name = "ROLE_CODE", unique = true, nullable = false, length = 100)
        public String getRoleCode() {
            return roleCode;
        }

        public void setRoleCode(String roleCode) {
            this.roleCode = roleCode;
        }

        @Override
        public boolean equals(Object obj) {
            if (this == obj) {
                return true;
            }

            if (obj == null || getClass() != obj.getClass()) {
                return false;
            }

            UserToUserRoleMapKey that = (UserToUserRoleMapKey) obj;
            return Objects.equals(userName, that.userName)
                   && Objects.equals(roleCode, that.roleCode);
        }

        @Override
        public int hashCode() {
            return Objects.hash(userName, roleCode);
        }
    }
}

```



Please refer the (JPA manual)[http://www.objectdb.com/java/jpa] for detail info about JPA development.



### Views And Service Forms

### Feign Clients
Feign clients provide an strongly typed http client to call web service. When using Feign clients, you define an interface of
the web service and annotated with `@FeignClient` and specify its configuration like url, context path etc.

After that you just define the method the same as the web service including the request mapping.
Then you can call the web service method like a local method and Feign client will help to call the web service.
By saying `strongly typed`, it means that feign client will mapped the response body with the return type of the web service method.

```java
@FeignClient(value = ADMIN_SERVICE, path = ADMIN_SERVICE_CONTEXT)
public interface UserClient {

    @RequestMapping(value = "/user/{userId}", method = RequestMethod.GET)
    UserDetailView getUser(@PathVariable("userId") String userName);

    @RequestMapping(value = "/user/queryUser", method = RequestMethod.GET)
    List<UserDetailView> getUsers();

    @RequestMapping(value = "/user/createUser", method = RequestMethod.POST)
    Boolean createUser(@RequestBody UserServiceForm user);

}
```

## Logging
* For backgroup jobs, log message must be written in "INFO" level when the jobs are started or completed.
* When exceptions are caught, they should be logged in "ERROR" level. Since there is global exception handler and
this handler will log all exceptions thrown to it, for most of the cases, developer do not need to catch and log
exception. The exception for this point is is that if an exception is not a kind of error, when it is caught, it
 should not be logged. For example, there is a program just want to validate whether an string is an valid date.
 This program use SimpleDateFormat.parse() to validate the string. If SimpleDateFormat.parse() throw ParseException,
 the program do not need to log the ParseException but just return false.
* Please use below approach to log exception:
```java
//correct approach
LOG.error("exception was caught:", be);

//wrong approach
LOG.error("exception was caught:" + be);

//wrong approach
LOG.error("exception was caught:" + be.getMessage());

//wrong approach
LOG.error("exception was caught {}", be);

```
* For most of the cases, do not need to use `if(LOG.isXXXEnable())` before logging message. This approach is for old
logging framework only(such as legacy common logging). The correct approach is:
```java
//correct approach, please note that the toString() for queryRequest must be defined.
LOG.debug("queryRequest = {}", queryRequest);

//wrong approach
if(LOG.isDebugEnabled()) {
    LOG.debug("queryRequest = " + queryRequest);
}
```

* If an message logging is time consuming, `if(LOG.isXXXEnable())` should be used. For example:
```java
if(LOG.isDebugEnabled()) {
    //here, need to do some complicated processing to work out the queryResult
    queryResult = ...;
    LOG.debug("queryRequest = {}" + queryRequest);
}
```

* All info is critical for problem diagnosis should be logged in "DEBUG" level. For example, when
an entity is retrieved from DB, log it in "DEBUG" level will be helpful for problem diagnosis sometimes.

* When developer want to log an object that have many fields, the toString() method of this object should be
implemented. `MoreObjects.toStringHelper()` should be used. Compare with print all fields of an object with
reflection, this approach has better performance. Furthermore, developer can control which field should be printed
 exactly. This is important if that object include sensitive fields. Below is an example:
```java
    @Override
    public String toString() {
        return MoreObjects.toStringHelper(this)
                .add("userName", userName)
                .add("userDescription", userDescription)
                // for sensitive fields (such as password), the value should be "masked"
                .add("password", "MASKED")
                .add("locked", locked)
                .add("groupCode", groupCode)
                .toString();
    }

```

* All info is critical for problem diagnosis in production should be logged in "INFO" level because "DEBUG" level
should not be enabled in production.

## Exception Handling


## Java Doc
The java doc will be used as technical spec. The main purpose of writting java doc are:
* Illustrate the structure or usage of programs to make them can be maintained easily.
* Make the source and the document of source are maintained in the same place and under the same VCS's control so that 
it is easier to make them in sync.

Please follow below guideline to write javadoc:
* For some common features, they cannot work until a few classes work together. In such case, create package-info.java
under that package and write java doc in it to explain the relatiopnship of those classes and how they work together to 
make the feature work. 
* Only public and protected methods need javadoc.
* For all javadoc, they must be started with one sentence that summarize the usage or purpose of a 
method/class/package.
* If the javadoc need to explain a lot of things and you want to create some sections in it, please use 
`<h1/> <h2/> <h3/>` to create some headers. For example:
```text
<h1>features</h1>
```
* If you want to create a new line, please use `<br>`(just like key in 'enter' in an text editor). For example:
```text
This is the basic exception for this system. All other customized exception type should
extend this class. <br>
```           
* If you want to create a new paragraph, please use `<p>`. Please note that, the `<p>` should be placed at the 
beginning of the new paragraph. For example:
```text
<p>This package include below key classes:
```            
* If you want to create some bullet points in the doc, please use `<ol>`,`<ul>` and `<li>`. For example:
```text
<ul>
  <li> Dynamic search criteria. For example, the search criteria can be changed base on the user input in UI.
  <li> Allow HTTP client to specify the range of records that should be returned.
  <li> Allow HTTP client to specify the sorting criteria. Multiple sorting criteria can be specified.
  <li> Return the no of total records for that query.
</ul>
```

* If you feel it is very difficult to write a javadoc because the java class/method is too simple so that you feel have 
nothing to say, please just mention the usage of the class/method or what is should do. The principle is that other 
developer can code the implementation of the class/method after they read the javadoc. This approach will be used by 
 controller often since they just provide HTTP endpoints and do not include complex logic. For example:

```text
<!-- this is a java doc for a class -->
Provide HTTP endpoints for user maintenance functions.

<!-- this is a java doc for a method -->
Query user detail info by user ID. It should be used by screen that show
the detail info of one user. Such as, detail user query screen or user detail maintain screen.
It should call {@link UserService#queryUser(String)}.
```
 
* If a class/method has relationship with or depends on other class/method, plesae use `{@link}` or `@see` to link 
them. For example:
```text
To enable those pagination features, please place annotation
{@link org.ckr.msdemo.pagination.EnablePagination} in any java configuration class in an application.
```

* Please use `@param` and `@return` to explain the meaning of parameters and returned result. For example:
```text
@return the new status of an order. 'A' means approved. 'C' means closed.
```

* If some class/method usage is too complicated to explain, please use source example to explain. The source example
must be placed within `<pre><code>... </code></pre>`. For example:
```text
<p>To enable pagination feature, please place this annotation in any java configuration class as below:
  <pre>
      <code>
          &#064;EnablePagination
          public class PaginationConfig {
          ...
          }
      </code>
 </pre>
```
Please note that, some char in the source need to be replaced with escape char. For example `@` must be replaced 
by `&#064;`. Otherwise, the javadoc cannot be generated. Please refer "Escape Sequence Numbers" in appendix.

* If you want to show an UML in javadoc, please use plantuml. For example:
```text
 * Below is a sequence diagram illustrate how all those key classes work together:
 * <p><img src="paginationSequence.svg" alt="sequence diagram">
 * <!--
@startuml paginationSequence.svg
autonumber

participant  HTTPClient
participant  PaginationInterceptor
participant  RestPaginationResponseAdvice
participant  PaginationContext
participant  Controller
participant  Service

participant  JpaRestPaginationService
"HTTPClient" -> "PaginationInterceptor": Send request
"PaginationInterceptor" -> "PaginationContext": Parse and store range of records and sorting fields from \
HTTP request.


"PaginationInterceptor" -> "Controller": Pass HTTP request to controller.                                         \
                                                            \t
note right
  Controller is the controller
  object that provide HTTP
  endpoint for this
  pagination query.
end note
"Controller" -> "Service": Construct QL and parameters for the query.
note right
  Service is the service object that implement
  the query logic. It should launch the DB
  transaction.
end note
"Service" -> "JpaRestPaginationService" : Pass search criteria to JpaRestPaginationService
"JpaRestPaginationService" -> "PaginationContext": Retrieve pagination request info.
"JpaRestPaginationService" -> "JpaRestPaginationService": Retrieve query content from DB.
"JpaRestPaginationService" -> "JpaRestPaginationService": Retrieve  and total number of \
records for this query from DB.
"JpaRestPaginationService" -> "PaginationContext": Save the total number of records.
"Service" <- "JpaRestPaginationService" : Return result list
"Controller" <- "Service" : Return result. Commit DB transaction.
"Controller" -> "RestPaginationResponseAdvice" : Pass HTTP response to it
"PaginationContext" <- "RestPaginationResponseAdvice": Save total number of records in HTTP response header.
"RestPaginationResponseAdvice" -> "PaginationInterceptor": Pass HTTP response to it
"PaginationInterceptor" -> "PaginationContext": Clear context data(pagination info) to release memory.
"HTTPClient" <- "PaginationInterceptor": Return result list with pagination info.
@enduml .
-->
 *
```
Please note that, 
> * an <img/> must be there, otherwise the UML cannot be shown. 
> * The <img/> must have an alt attribute.
> * All plantuml statements must be placed within `<!-- -->` . `-->` cannot be used in the plantuml statements. All of them
 must be replaced with `<--`.
> * For other javadoc statements, there is a `*` before them. For plantuml statement, make sure there is no `*` before
them. The reason is that, sometimes, the IDEA planuml editor is not available in local machine, we need to copy the plantuml 
  statement to online editor and edit there. If will be very trouble to remove the `*`.

* If you want to place an image to java doc. create a `doc-files` foler under current package. Copy the image you  want to show 
to this folder. Then, place an `<img/>` in the java doc. For example:
```text
<img src="doc-files/abc.png" alt="demo diagram">
```

If you need more example, please refer
[Collections.java](http://www.docjar.net/html/api/java/util/Collections.java.html). Please note that this example is for an java basic class,
it is too long and too complicated for this project.

For more detail, please refer
[How to Write Doc Comments for the Javadoc Tool](http://www.oracle.com/technetwork/java/javase/documentation/index-137868.html)

## DB Deployment

This section introduces DB deployment from development and release management's perspectives.
Since Liquibase is used in our project, we will utilise this tool to manage database from developers' and release team's perspective.

### Development team's Perspective

Usually developers will get tasks related to database, like developing new function which require either create/modify existing table,
or fixing some ticket that need to insert data to some tables. Developer will follow below process to do DB deployment.
1. Create/modify JPA entity if needed; (Refer to section `Components For Tasks` > `Entities` above)
2. Generate change log using maven `mvn clean compile -Dmsdemo.skipGenLiquibaseXml=false`;
3. Check and merge generated changelog in `target\liquibaseXml` into main changelog in `resources` folder;
4. Add changelog/changeset for ticket/defect fixing;
5. Using `mvn liquibase:update` to sync your change to development database.

**Remember to double check your database url since `mvn liquibase:update` will update database**

### Release team's Perspective

For release team to perform database deployment, we would like to choose a different appoach.
Normally those deployment environments are sensitive like UAT/production database.
Liquibase should not change anything to these enviroment directly.
Liquibase provides `liquibase:updateSQL` command to generate SQL scripts.
After properly check to these scripts, they could be package and send to release team to deploy using corresponding tools, like `sqlplus` for oracle.
Refer to [Liquibase Offline Database Support](http://www.liquibase.org/documentation/offline.html) for more detail.



## Testing

### Mocked Unit Test

Most of the logic should be tested by unit test case because the cost of unit test is cheap. However, most of the programs depends on DB so that unit test cannot be done until something is mocked. So, jmockit is used
for mocking.
To create an unit test case, a junit test class should be create. The package should be the same as the tested class and the class name is {the name of the tested class} + "MockedTests".

#### Basic Annotation of Jmockit

* `@Tested`
    The `@Tested` annotation provides support for the automatic creation of an object under test, with its dependencies automatically filled with mocked and/or real instances.
* `@Injectable`
    Only one particular instance is mocked.
* `@Mocked`
    All current and future instances are mocked.

#### Basic Usage of Jmockit

In a classic spring application, many dependencies will be injected into classes. In order to focus on the logic of our class, we use `@Injectable` and `@Mocked` to mock those dependencies so that we could define the result from call for depenencies instead of calling in real.

```java

@Tested
ClassUnderTest sut;

@Test
public void aTestMethod(@Mocked MyCollaborator anyCollaborator, @Injectable MyCollaborator mock, <any number of mock parameters>) {
   // Record phase: expectations on mocks are recorded; empty if nothing to record.
   new Expectations() {{
      anyCollaborator.getData(); result = "my test data";
      anyCollaborator.doSomething(anyInt, "some expected value", anyString); times = 1;
   }};
   // Replay phase: invocations on mocks are "replayed"; code under test is exercised.
    sut.testMethod();
   // Verify phase: expectations on mocks are verified; empty if nothing to verify.
   new Verifications() {{
      // Verify the "MyCollaborator#doSomething()" method was executed at least once:
      mock.doSomething();

      // Even constructor invocations can be verified:
      new MyCollaborator(); times = 0; // verifies there were no matching invocations

      // Another verification, which allows up to three matching invocations:
      mock.someOtherMethod(anyBoolean, any, withInstanceOf(Xyz.class)); maxTimes = 3;
   }};

}

```


#### Partial mocking of Jmockit

When we need some method in a class to be mocked and others not, we need to apply partial mock technique. For example, for a method that has  heavy IO operation like query all users from database, we would like to just return a predefined list insteady of query in actual database, we would like to mock such method.

```java

public class PartialMockingMockTests
{
   static class Collaborator
   {
      final int value;

      Collaborator() { value = -1; }
      Collaborator(int value) { this.value = value; }

      int getValue() { return value; }
      final boolean simpleOperation(int a, String b, Date c) { return true; }
      static void doSomething(boolean b, String s) {
          // some IO heavy operation like some CRUD
       }
   }

   @Test
   public void testPartiallyMockingAClassAndItsInstances() {
      Collaborator anyInstance = new Collaborator();

      new Expectations(Collaborator.class) {{
         anyInstance.getValue(); result = 123;

         // mylist is some predefine data;
         Collaborator.doSomething(anyBoolean, "test"); result = mylist;
      }};

      // Not mocked, as no constructor expectations were recorded:
      Collaborator c1 = new Collaborator();
      Collaborator c2 = new Collaborator(150);

      // Mocked, as a matching method expectation was recorded:
      assertEquals(123, c1.getValue());
      assertEquals(123, c2.getValue());

      // Not mocked:
      assertTrue(c1.simpleOperation(1, "b", null));
      assertEquals(45, new Collaborator(45).value);
   }

   @Test
   public void testPartiallyMockingASingleInstance() {
      Collaborator collaborator = new Collaborator(2);

      new Expectations(collaborator) {{
         collaborator.getValue(); result = 123;
         collaborator.simpleOperation(1, "", null); result = false;

         // Static methods can be dynamically mocked too.
         Collaborator.doSomething(anyBoolean, "test");
      }};

      // Mocked:
      assertEquals(123, collaborator.getValue());
      assertFalse(collaborator.simpleOperation(1, "", null));
      Collaborator.doSomething(true, "test");

      // Not mocked:
      assertEquals(2, collaborator.value);
      assertEquals(45, new Collaborator(45).getValue());
      assertEquals(-1, new Collaborator().getValue());
   }
}

```

#### Test Private method

During development, there is some complicated logic which you divide the logic into several small private methods and call these methods in a method. How do we test such method in our project? Since Jmockit could not support mocking for private methods, we could do it this way,
1. we give package access to these methods instead of `private` so that these method could be mocked.
2. then we test these package accessible methods seperatly.
3. test the overall method that call these methods.

```java

public class Complex {
    // give package access instead of "private"
    int add(int a, int b){
        // IO heavy operation or complicated logic
        return a + b;
    }

    // give package access instead of "private"
    int sub(int a, int b){
        // IO heavy operation or complicated logic
        return a - b;
    }

    public int addSubMul(int a, int b, int c, int d){
        int addValue = add(a, b);
        int subValue = sub(c, d);
        // IO heavy operation or complicated logic
        return addValue * subValue;
    }
}

```

```java

public class ComplexTests {
    @Tested
    Complex complex;

    @Test
    //test package accessible method
    public void testAdd() throws Exception {
        int number = complex.add(1, 2);
        new Verifications(){{
            assertThat(number).isEqualTo(3);
        }};
    }

    @Test
    //test package accessible method
    public void testSub() throws Exception {
        int number = complex.sub(3, 4);
        new Verifications(){{
            assertThat(number).isEqualTo(-1);
        }};
    }

    @Test
    // test the overall method that call package accessible methods.
    public void testAddSubMul() throws Exception {
        new Expectations(complex){{
            complex.add(anyInt, anyInt); result = 3;
            complex.sub(anyInt, anyInt); result = -1;
        }};
        int number = complex.addSubMul(1, 2, 3, 4);
        new Verifications(){{
            assertThat(number).isEqualTo(-3);
        }};
    }
}

```



### DB Unit Test
For those classes query directly to DB like `***Repository` or `***Dao`, we should create an unit test case for each class. The package should be the same as the tested class and the class name is {the name of the tested class} + "DbTests".
In a DB unit test, we annotate the test class with `@DbUnitTest`. When running a test case, a temperate in-memory database will start up automatically and sync the data using Liquibase.
And when your test is done, the database will be shutdown. In this way, there will be no impact to any real application database and you have no worry about polluting database with testing data.

```java

@RunWith(SpringRunner.class)
@DbUnitTest
public class UserRepositoryDbTests {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private UserGroupRepository userGroupRepository;

    @Autowired
    private TestEntityManager testEntityManager;

    @Test
    public void testfindByUserName() {
        this.testEntityManager.persist(new User("xiaoming", "sb"));
        User user = this.userRepository.findByUserName("xiaoming");
        assertThat(user.getUserName()).isEqualTo("xiaoming");
        assertThat(user.getUserDescription()).isEqualTo("sb");
    }
}

```

### Assembly Test
After unit test passed, we need to have an upper level test called Assembly test.
Unlike mocked unit test that will mock target service's dependencies, this kind of test will simulate a real call to the target service without mocking.
We create an assembly test case with name {the name of the tested class} + "AsmTests".
Usually we will annotate the test class with `@AssemblyTest` which actually using `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`.
With this annotation, you also run your test in an embedded in-memory database and use `TestRestTemplate` to perform call to the service's url.

```java

@RunWith(SpringRunner.class)
@AssemblyTest
public class JpaRestPaginationServiceAsmTests {

    @Autowired
    TestRestTemplate testRestTemplate;

    @LocalServerPort
    private int port;

    private static Logger LOG = LoggerFactory.getLogger(JpaRestPaginationService.class);

    /**
     * JpaRestPaginationServiceAsmTests template for test case.
     *
     * @param userName user name
     * @param userDesc user description
     * @param range range
     * @param sortBy sort by
     * @param length length
     * @param firstName first name
     * @param statusCode status code
     */
    public void testTemplate(String userName, String userDesc, String range, String sortBy,
                             int length, String firstName, int statusCode) {

        HttpHeaders headers = new HttpHeaders();
        if (!StringUtils.isEmpty(range)) {
            headers.add("Range", range);
        }
        if (!StringUtils.isEmpty(sortBy)) {
            headers.add("SortBy", sortBy);
        }
        headers.setContentType(MediaType.TEXT_PLAIN);
        HttpEntity<String> entity = new HttpEntity<String>(headers);

        ParameterizedTypeReference<List<UserWithRole>> userWithRoleBean = new ParameterizedTypeReference
            <List<UserWithRole>>() {
        };

        String url = "http://localhost:" + port + "/dbaccesstest/user/queryUsersWithRoles?userName=" + userName
            + "&userDesc=" + userDesc;

        ResponseEntity<List<UserWithRole>> response = testRestTemplate.exchange(url, HttpMethod.GET, entity,
            userWithRoleBean);

        List<UserWithRole> userWithRoles = response.getBody();

        for (UserWithRole userWithRole :
            userWithRoles) {
            LOG.info(userWithRole.toString());
        }
        assertThat(userWithRoles.size()).isEqualTo(length);
        if (userWithRoles != null && userWithRoles.size() > 0 && !StringUtils.isEmpty(sortBy)) {
            assertThat(userWithRoles.get(0).getUserName()).isEqualTo(firstName);
        }

        MediaType contentType1 = response.getHeaders().getContentType();
        LOG.info(contentType1.toString());
        HttpStatus statusCode1 = response.getStatusCode();
        LOG.info("statusCode is {}", statusCode1.toString());
        assertThat(statusCode1.value()).isEqualTo(statusCode);
    }

    @Test
    public void testSample() {
        this.testTemplate("", "", "items=1-20", "-userName",
            15, "DEF", 200);
    }

    @Test
    public void testPageSize() {
        this.testTemplate("", "", "items=1-14", "-userName",
            14, "DEF", 200);
    }

    @Test
    public void testOrder() {
        this.testTemplate("", "", "items=1-20", "+userName",
            15, "ABC", 200);
    }
}

```

### API Test

API test is constructed to simulate the use of the API by end-user applications. We use Feign client and Robot framework for API test.
Feign client provides a easy to use http client and Robot framework provides key words to perform test.

#### Robot framework

Robot Framework is a generic test automation framework using build-in or user defined keywords to write test case.
For user defined keywords, Refer to below example,

1. We create a class with name `UserKeyword` with constructor to initialise `userClient`.
2. Then create two keywords `Query User` and `Create User` using normal java methods `queryUser` and`createUser`.
3. In the test case we import our `UserKeyword` and define the test case with keywords `Query User` and `Create User`.
4. After run this test case, test result will be shown in the generated log/report.

*note:* robot framework also supports return value from keyword.

```java

// UserKeyword.java
public class UserKeyword {
    private static final Logger LOG = LoggerFactory.getLogger(UserKeyword.class);

    UserClient userClient;

    public UserKeyword() {
        this.userClient = (UserClient) BeanContainer.getBean(UserClient.class);
    }

    public void queryUser(String userName) {

        UserDetailView userDetailView = this.userClient.queryUserById(userName);

        assertThat(userDetailView.getUserName()).isEqualTo(userName);

    }

    public void createUser(String userName, String userDescription, String password, Boolean locked) {
        UserServiceForm userServiceForm = new UserServiceForm();
        userServiceForm.setUserName(userName);
        userServiceForm.setUserDescription(userDescription);
        userServiceForm.setPassword(password);
        userServiceForm.setLocked(locked);
        this.userClient.createUser(userServiceForm);
    }

    public String myKeyword(String arg) {
        return helperMethod(arg);
    }

    private String helperMethod(String arg) {
        return arg.toUpperCase();
    }
}

```


```robot

# user_maintenance.robot
*** Settings ***
Library   org.ckr.msdemo.adminservice.apitest.keywords.UserKeyword

*** Variables ***

*** Test Cases ***
create user
    # robot framework supports return value from keyword.
    ${username} =    My Keyword      abc
    Query User    ${username}
    Create User     haiyan  haiyan lu   haiyangpassword     true

```

##### Robot framework - Handle whitespace

Refer to [Robot framework - Handle whitespace](http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#handling-whitespace)

```robot

# user_maintenance.robot
*** Settings ***
*** Variables ***
*** Test Cases ***
One Space
    Should Be Equal    ${SPACE}          \ \

Four Spaces
    Should Be Equal    ${SPACE * 4}      \ \ \ \ \

Ten Spaces
    Should Be Equal    ${SPACE * 10}     \ \ \ \ \ \ \ \ \ \ \

Quoted Space
    Should Be Equal    "${SPACE}"        " "

Quoted Spaces
    Should Be Equal    "${SPACE * 2}"    " \ "

Empty
    Should Be Equal    ${EMPTY}          \

```

##### Robot framework - Default values to keywords

It is often useful that some of the arguments that a keyword uses have default values.
In Java one method can have several implementations with different signatures. Robot Framework regards all these implementations as one keyword, which can be used with different arguments. This syntax can thus be used to provide support for the default values. This is illustrated by the example below:

```java

public class UserKeyword {
    private static final Logger LOG = LoggerFactory.getLogger(UserKeyword.class);

    UserClient userClient;

    public UserKeyword() {
        this.userClient = (UserClient) BeanContainer.getBean(UserClient.class);
    }

    public void queryUser(String userName) {
        UserDetailView userDetailView = this.userClient.queryUserById(userName);
        assertThat(userDetailView.getUserName()).isEqualTo(userName);
    }

    public void queryUser() {
        queryUser("defaultUser");
    }

    public void createUser(String userName, String userDescription, String password, Boolean locked) {
        UserServiceForm userServiceForm = new UserServiceForm();
        userServiceForm.setUserName(userName);
        userServiceForm.setUserDescription(userDescription);
        userServiceForm.setPassword(password);
        userServiceForm.setLocked(locked);
        this.userClient.createUser(userServiceForm);
    }

    public void createUser(String userName, String userDescription, String password) {
        createUser(userName, userDescription, password, false);
    }

    public void createUser(String userName, String userDescription) {
        createUser(userName, userDescription, "defaultPassword");
    }

    public void createUser(String userName) {
        createUser(userName, "default description");
    }

    public void createUser() {
        createUser("defaultUser");
    }
}

```

```robot

# user_maintenance.robot
*** Settings ***
Library   org.ckr.msdemo.adminservice.apitest.keywords.UserKeyword

*** Variables ***

*** Test Cases ***
create user default
    # query user with username defaultUser
    Query User
    # query user with all prameter specified
    Create User     haiyan  haiyan lu   haiyangpassword     true
    # query user with default locked=false
    Create User     haiyan  haiyan lu   haiyangpassword
    # query user with default locked=false and password=defaultPassword
    Create User     haiyan  haiyan lu
    # query user with default locked, password and description
    Create User     haiyan
    # query user with default locked, password, description and user name
    Create User

```

##### Robot framework - Free keyword arguments

What this `Free keyword arguments` means is that keywords can receive all arguments that use the `name=value` syntax. We could easily define our test case with less password for method with lots of parameters.
An limitation is that free keyword argument names must always be strings.

> Note
1. A keyword supporting kwargs cannot have more than one signature.
2. The type of the kwargs argument must be exactly java.util.Map, not any of its sub types.

```java

public class UserKeyword {
    private static final Logger LOG = LoggerFactory.getLogger(UserKeyword.class);

    UserClient userClient;

    public UserKeyword() {
        this.userClient = (UserClient) BeanContainer.getBean(UserClient.class);
    }

    public void queryUser(String userName) {
        UserDetailView userDetailView = this.userClient.queryUserById(userName);
        assertThat(userDetailView.getUserName()).isEqualTo(userName);
    }

    public void queryUser() {
        queryUser("defaultUser");
    }

    public void createUser(String userName, String userDescription, String password, Boolean locked) {
        UserServiceForm userServiceForm = new UserServiceForm();
        userServiceForm.setUserName(userName);
        userServiceForm.setUserDescription(userDescription);
        userServiceForm.setPassword(password);
        userServiceForm.setLocked(locked);
        this.userClient.createUser(userServiceForm);
    }

    private static String nvl(String value, String alternateValue) {
        if (value == null)
            return alternateValue;

        return value;
    }

    public void createUser2(Map<String, Object> kwargs) {

        String userName = (String) kwargs.get("userName");
        String userDescription = (String) kwargs.get("userDescription");
        String password = (String) kwargs.get("password");
        String locked = (String) kwargs.get("locked");

        if(userName != null){
            UserServiceForm userServiceForm = new UserServiceForm();
            userServiceForm.setUserName(userName);
            userServiceForm.setUserDescription(nvl(userDescription, "default desc"));
            userServiceForm.setPassword(nvl(password, "defaultpassword"));
            userServiceForm.setLocked(Boolean.valueOf(nvl(locked, "false")));
            this.userClient.createUser(userServiceForm);
        }else
            throw new ApplicationException("no user name specified.");

    }
}

```




## Configuration

### Security

### Swagger


## Naming Convention

### Package

**Should be lowercase and not underscores/dash/uppercase letters.**

* Good: `package org.springframework;`
* Bad: `package org.springFramework;` and `package org.spring_framework;`

### Class
**Class and Interfaces names are in UpperCamelCase.The name of java classes should be a noun or a noun phrase.**

* Good: `UserRole`
* Bad: `userRole`

**Common component class like controller/service/repository should end with component type name**

* controller like `HomeController`
* service like `UserService`
* repository `RoleRepository`

**Test classes are named starting with the name of the class they are testing, and ending with `Tests`.**

* Good: `HomeController` and `HomeControllerMockTests`
* Bad: `HomeController` and `HomeControllerMockTest`

**Different kind of test should be indicated in the name.**

* test case that will connect to database. Good: `HomeController` and `HomeControllerDbTests`. Bad: `HomeControllerTest`
* test case that will use mocking tool.    Good: `HomeController` and `HomeControllerMockTests`. Bad: `HomeControllerTest`

### Method
**All java methods should be an verb or verb phrase. Method names are written in lowerCamelCase.**

* Good: `adjustQueryString`
* Bad: `adjustquerystring` or `AdjustQueryString`

**Method names are typically verbs or verb phrases.**

* Good: `configure` or `findPolicyByPolicyNo`
* Bad: `configuration` or `policyNo`

**For unit test(include mocked test and DB test), the method name pattern should be
"test{the name method being tested}". If a methods are tested multiple times, the method name
pattern should be "test{the name method being tested}_{remark for scenario}".**

* Good: `testFindPolicyByPolicyNo` or `testFindPolicyByPolicyNo_found` or `testFindPolicyByPolicyNo_NotFound`
* Bad: `findPolicyByPolicyNo_found`

### Variable
**non-constant variable should define in lowerCamelCase.**

* Good: `errorResponse` or `headers`
* Bad: `ErrorResponse`

**avoid using `is****` for boolean type variable since IDE default getter/setter will become `isGood()`/`setGood(boolean good)`**

* Good: `good`
* Bad: `isBad`

**all letters of constant variable is uppercase , with words separated by underscores.**

* Good: `ADMIN_SERVICE` or `ADMIN_SERVICE_CONTEXT`
* Bad: `admin_service`


## Appendix
### Escape Sequence Numbers
|   Original Char     |Should be replaced by |
|:-------------------:|:--------------------:|
|                     | &#38;#32;                | 
|!                    | &#38;#33;                |
|"                    | &#38;#34;                |
|#                    | &#38;#35;                |
|$                    | &#38;#36;                |
|%                    | &#38;#37;                |
|&                    | &#38;#38;                |
|'                    | &#38;#39;                |
|(                    | &#38;#40;                |
|)                    | &#38;#41;                |
|*                    | &#38;#42;                |
|+                    | &#38;#43;                |
|,                    | &#38;#44;                |
|-                    | &#38;#45;                |
|.                    | &#38;#46;                |
|/                    | &#38;#47;                |
|0                    | &#38;#48;                |
|1                    | &#38;#49;                |
|2                    | &#38;#50;                |
|3                    | &#38;#51;                |
|4                    | &#38;#52;                |
|5                    | &#38;#53;                |
|6                    | &#38;#54;                |
|7                    | &#38;#55;                |
|8                    | &#38;#56;                |
|9                    | &#38;#57;                |
|:                    | &#38;#58;                |
|;                    | &#38;#59;                |
|<                    | &#38;#60;                |
|=                    | &#38;#61;                |
|>                    | &#38;#62;                |
|?                    | &#38;#63;                |
|@                    | &#38;#64;                |
|A                    | &#38;#65;                |
|B                    | &#38;#66;                |
|C                    | &#38;#67;                |
|D                    | &#38;#68;                |
|E                    | &#38;#69;                |
|F                    | &#38;#70;                |
|G                    | &#38;#71;                |
|H                    | &#38;#72;                |
|I                    | &#38;#73;                |
|J                    | &#38;#74;                |
|K                    | &#38;#75;                |
|L                    | &#38;#76;                |
|M                    | &#38;#77;                |
|N                    | &#38;#78;                |
|O                    | &#38;#79;                |
|P                    | &#38;#80;                |
|Q                    | &#38;#81;                |
|R                    | &#38;#82;                |
|S                    | &#38;#83;                |
|T                    | &#38;#84;                |
|U                    | &#38;#85;                |
|V                    | &#38;#86;                |
|W                    | &#38;#87;                |
|X                    | &#38;#88;                |
|Y                    | &#38;#89;                |
|Z                    | &#38;#90;                |
|[                    | &#38;#91;                |
|\                    | &#38;#92;                |
|]                    | &#38;#93;                |
|^                    | &#38;#94;                |
|_                    | &#38;#95;                |
|`                    | &#38;#96;                |
|a                    | &#38;#97;                |
|b                    | &#38;#98;                |
|c                    | &#38;#99;                |
|d                    | &#38;#100;               |
|e                    | &#38;#101;               |
|f                    | &#38;#102;               |
|g                    | &#38;#103;               |
|h                    | &#38;#104;               |
|i                    | &#38;#105;               |
|j                    | &#38;#106;               |
|k                    | &#38;#107;               |
|l                    | &#38;#108;               |
|m                    | &#38;#109;               |
|n                    | &#38;#110;               |
|o                    | &#38;#111;               |
|p                    | &#38;#112;               |
|q                    | &#38;#113;               |
|r                    | &#38;#114;               |
|s                    | &#38;#115;               |
|t                    | &#38;#116;               |
|u                    | &#38;#117;               |
|v                    | &#38;#118;               |
|w                    | &#38;#119;               |
|x                    | &#38;#120;               |
|y                    | &#38;#121;               |
|z                    | &#38;#122;               |
|{                    | &#38;#123;               |
||                    | &#38;#124;               |
|}                    | &#38;#125;               |
|~                    | &#38;#126;               |
|                    | &#38;#127;               |
|€                    | &#38;#128;               |
|                    | &#38;#129;               |
|‚                    | &#38;#130;               |
|ƒ                    | &#38;#131;               |
|„                    | &#38;#132;               |
|…                    | &#38;#133;               |
|†                    | &#38;#134;               |
|‡                    | &#38;#135;               |
|ˆ                    | &#38;#136;               |
|‰                    | &#38;#137;               |
|Š                    | &#38;#138;               |
|‹                    | &#38;#139;               |
|Œ                    | &#38;#140;               |
|                    | &#38;#141;               |
|Ž                    | &#38;#142;               |
|                    | &#38;#143;               |
|                    | &#38;#144;               |
|‘                    | &#38;#145;               |
|’                    | &#38;#146;               |
|“                    | &#38;#147;               |
|”                    | &#38;#148;               |
|•                    | &#38;#149;               |
|–                    | &#38;#150;               |
|—                    | &#38;#151;               |
|˜                    | &#38;#152;               |
|™                    | &#38;#153;               |
|š                    | &#38;#154;               |
|›                    | &#38;#155;               |
|œ                    | &#38;#156;               |
|                    | &#38;#157;               |
|ž                    | &#38;#158;               |
|Ÿ                    | &#38;#159;               |
|                     | &#38;#160;               |
|¡                    | &#38;#161;               |
|¢                    | &#38;#162;               |
|£                    | &#38;#163;               |
|¤                    | &#38;#164;               |
|¥                    | &#38;#165;               |
|¦                    | &#38;#166;               |
|§                    | &#38;#167;               |
|¨                    | &#38;#168;               |
|©                    | &#38;#169;               |
|ª                    | &#38;#170;               |
|«                    | &#38;#171;               |
|¬                    | &#38;#172;               |
|                     | &#38;#173;               |
|®                    | &#38;#174;               |
|¯                    | &#38;#175;               |
|°                    | &#38;#176;               |
|±                    | &#38;#177;               |
|²                    | &#38;#178;               |
|³                    | &#38;#179;               |
|´                    | &#38;#180;               |
|µ                    | &#38;#181;               |
|¶                    | &#38;#182;               |
|·                    | &#38;#183;               |
|¸                    | &#38;#184;               |
|¹                    | &#38;#185;               |
|º                    | &#38;#186;               |
|»                    | &#38;#187;               |
|¼                    | &#38;#188;               |
|½                    | &#38;#189;               |
|¾                    | &#38;#190;               |
|¿                    | &#38;#191;               |
|À                    | &#38;#192;               |
|Á                    | &#38;#193;               |
|Â                    | &#38;#194;               |
|Ã                    | &#38;#195;               |
|Ä                    | &#38;#196;               |
|Å                    | &#38;#197;               |
|Æ                    | &#38;#198;               |
|Ç                    | &#38;#199;               |
|È                    | &#38;#200;               |
|É                    | &#38;#201;               |
|Ê                    | &#38;#202;               |
|Ë                    | &#38;#203;               |
|Ì                    | &#38;#204;               |
|Í                    | &#38;#205;               |
|Î                    | &#38;#206;               |
|Ï                    | &#38;#207;               |
|Ð                    | &#38;#208;               |
|Ñ                    | &#38;#209;               |
|Ò                    | &#38;#210;               |
|Ó                    | &#38;#211;               |
|Ô                    | &#38;#212;               |
|Õ                    | &#38;#213;               |
|Ö                    | &#38;#214;               |
|×                    | &#38;#215;               |
|Ø                    | &#38;#216;               |
|Ù                    | &#38;#217;               |
|Ú                    | &#38;#218;               |
|Û                    | &#38;#219;               |
|Ü                    | &#38;#220;               |
|Ý                    | &#38;#221;               |
|Þ                    | &#38;#222;               |
|ß                    | &#38;#223;               |
|à                    | &#38;#224;               |
|á                    | &#38;#225;               |
|â                    | &#38;#226;               |
|ã                    | &#38;#227;               |
|ä                    | &#38;#228;               |
|å                    | &#38;#229;               |
|æ                    | &#38;#230;               |
|ç                    | &#38;#231;               |
|è                    | &#38;#232;               |
|é                    | &#38;#233;               |
|ê                    | &#38;#234;               |
|ë                    | &#38;#235;               |
|ì                    | &#38;#236;               |
|í                    | &#38;#237;               |
|î                    | &#38;#238;               |
|ï                    | &#38;#239;               |
|ð                    | &#38;#240;               |
|ñ                    | &#38;#241;               |
|ò                    | &#38;#242;               |
|ó                    | &#38;#243;               |
|ô                    | &#38;#244;               |
|õ                    | &#38;#245;               |
|ö                    | &#38;#246;               |
|÷                    | &#38;#247;               |
|ø                    | &#38;#248;               |
|ù                    | &#38;#249;               |
|ú                    | &#38;#250;               |
|û                    | &#38;#251;               |
|ü                    | &#38;#252;               |
|ý                    | &#38;#253;               |
|þ                    | &#38;#254;               |
|ÿ                    | &#38;#255;               |
|Ā                    | &#38;#256;               |
|ā                    | &#38;#257;               |
|Ă                    | &#38;#258;               |
|ă                    | &#38;#259;               |
|Ą                    | &#38;#260;               |
|ą                    | &#38;#261;               |
|Ć                    | &#38;#262;               |
|ć                    | &#38;#263;               |
|Ĉ                    | &#38;#264;               |
|ĉ                    | &#38;#265;               |
|Ċ                    | &#38;#266;               |
|ċ                    | &#38;#267;               |
|Č                    | &#38;#268;               |
|č                    | &#38;#269;               |
|Ď                    | &#38;#270;               |
|ď                    | &#38;#271;               |
|Đ                    | &#38;#272;               |
|đ                    | &#38;#273;               |
|Ē                    | &#38;#274;               |
|ē                    | &#38;#275;               |
|Ĕ                    | &#38;#276;               |
|ĕ                    | &#38;#277;               |
|Ė                    | &#38;#278;               |
|ė                    | &#38;#279;               |
|Ę                    | &#38;#280;               |
|ę                    | &#38;#281;               |
|Ě                    | &#38;#282;               |
|ě                    | &#38;#283;               |
|Ĝ                    | &#38;#284;               |
|ĝ                    | &#38;#285;               |
|Ğ                    | &#38;#286;               |
|ğ                    | &#38;#287;               |
|Ġ                    | &#38;#288;               |
|ġ                    | &#38;#289;               |
|Ģ                    | &#38;#290;               |
|ģ                    | &#38;#291;               |
|Ĥ                    | &#38;#292;               |
|ĥ                    | &#38;#293;               |
|Ħ                    | &#38;#294;               |
|ħ                    | &#38;#295;               |
|Ĩ                    | &#38;#296;               |
|ĩ                    | &#38;#297;               |
|Ī                    | &#38;#298;               |
|ī                    | &#38;#299;               |
|Ĭ                    | &#38;#300;               |
|ĭ                    | &#38;#301;               |
|Į                    | &#38;#302;               |
|į                    | &#38;#303;               |
|İ                    | &#38;#304;               |
|ı                    | &#38;#305;               |
|Ĳ                    | &#38;#306;               |
|ĳ                    | &#38;#307;               |
|Ĵ                    | &#38;#308;               |
|ĵ                    | &#38;#309;               |
|Ķ                    | &#38;#310;               |
|ķ                    | &#38;#311;               |
|ĸ                    | &#38;#312;               |
|Ĺ                    | &#38;#313;               |
|ĺ                    | &#38;#314;               |
|Ļ                    | &#38;#315;               |
|ļ                    | &#38;#316;               |
|Ľ                    | &#38;#317;               |
|ľ                    | &#38;#318;               |
|Ŀ                    | &#38;#319;               |
|ŀ                    | &#38;#320;               |
|Ł                    | &#38;#321;               |
|ł                    | &#38;#322;               |
|Ń                    | &#38;#323;               |
|ń                    | &#38;#324;               |
|Ņ                    | &#38;#325;               |
|ņ                    | &#38;#326;               |
|Ň                    | &#38;#327;               |
|ň                    | &#38;#328;               |
|ŉ                    | &#38;#329;               |
|Ŋ                    | &#38;#330;               |
|ŋ                    | &#38;#331;               |
|Ō                    | &#38;#332;               |
|ō                    | &#38;#333;               |
|Ŏ                    | &#38;#334;               |
|ŏ                    | &#38;#335;               |
|Ő                    | &#38;#336;               |
|ő                    | &#38;#337;               |
|Œ                    | &#38;#338;               |
|œ                    | &#38;#339;               |
|Ŕ                    | &#38;#340;               |
|ŕ                    | &#38;#341;               |
|Ŗ                    | &#38;#342;               |
|ŗ                    | &#38;#343;               |
|Ř                    | &#38;#344;               |
|ř                    | &#38;#345;               |
|Ś                    | &#38;#346;               |
|ś                    | &#38;#347;               |
|Ŝ                    | &#38;#348;               |
|ŝ                    | &#38;#349;               |
|Ş                    | &#38;#350;               |
|ş                    | &#38;#351;               |
|Š                    | &#38;#352;               |
|š                    | &#38;#353;               |
|Ţ                    | &#38;#354;               |
|ţ                    | &#38;#355;               |
|Ť                    | &#38;#356;               |
|ť                    | &#38;#357;               |
|Ŧ                    | &#38;#358;               |
|ŧ                    | &#38;#359;               |
|Ũ                    | &#38;#360;               |
|ũ                    | &#38;#361;               |
|Ū                    | &#38;#362;               |
|ū                    | &#38;#363;               |
|Ŭ                    | &#38;#364;               |
|ŭ                    | &#38;#365;               |
|Ů                    | &#38;#366;               |
|ů                    | &#38;#367;               |
|Ű                    | &#38;#368;               |
|ű                    | &#38;#369;               |
|Ų                    | &#38;#370;               |
|ų                    | &#38;#371;               |
|Ŵ                    | &#38;#372;               |
|ŵ                    | &#38;#373;               |
|Ŷ                    | &#38;#374;               |
|ŷ                    | &#38;#375;               |
|Ÿ                    | &#38;#376;               |
|Ź                    | &#38;#377;               |
|ź                    | &#38;#378;               |
|Ż                    | &#38;#379;               |
|ż                    | &#38;#380;               |
|Ž                    | &#38;#381;               |
|ž                    | &#38;#382;               |
|ſ                    | &#38;#383;               |
|ƀ                    | &#38;#384;               |
|Ɓ                    | &#38;#385;               |
|Ƃ                    | &#38;#386;               |
|ƃ                    | &#38;#387;               |
|Ƅ                    | &#38;#388;               |
|ƅ                    | &#38;#389;               |
|Ɔ                    | &#38;#390;               |
|Ƈ                    | &#38;#391;               |
|ƈ                    | &#38;#392;               |
|Ɖ                    | &#38;#393;               |
|Ɗ                    | &#38;#394;               |
|Ƌ                    | &#38;#395;               |
|ƌ                    | &#38;#396;               |
|ƍ                    | &#38;#397;               |
|Ǝ                    | &#38;#398;               |
|Ə                    | &#38;#399;               |
|Ɛ                    | &#38;#400;               |
|Ƒ                    | &#38;#401;               |
|ƒ                    | &#38;#402;               |
|Ɠ                    | &#38;#403;               |
|Ɣ                    | &#38;#404;               |
|ƕ                    | &#38;#405;               |
|Ɩ                    | &#38;#406;               |
|Ɨ                    | &#38;#407;               |
|Ƙ                    | &#38;#408;               |
|ƙ                    | &#38;#409;               |
|ƚ                    | &#38;#410;               |
|ƛ                    | &#38;#411;               |
|Ɯ                    | &#38;#412;               |
|Ɲ                    | &#38;#413;               |
|ƞ                    | &#38;#414;               |
|Ɵ                    | &#38;#415;               |
|Ơ                    | &#38;#416;               |
|ơ                    | &#38;#417;               |
|Ƣ                    | &#38;#418;               |
|ƣ                    | &#38;#419;               |
|Ƥ                    | &#38;#420;               |
|ƥ                    | &#38;#421;               |
|Ʀ                    | &#38;#422;               |
|Ƨ                    | &#38;#423;               |
|ƨ                    | &#38;#424;               |
|Ʃ                    | &#38;#425;               |
|ƪ                    | &#38;#426;               |
|ƫ                    | &#38;#427;               |
|Ƭ                    | &#38;#428;               |
|ƭ                    | &#38;#429;               |
|Ʈ                    | &#38;#430;               |
|Ư                    | &#38;#431;               |
|ư                    | &#38;#432;               |
|Ʊ                    | &#38;#433;               |
|Ʋ                    | &#38;#434;               |
|Ƴ                    | &#38;#435;               |
|ƴ                    | &#38;#436;               |
|Ƶ                    | &#38;#437;               |
|ƶ                    | &#38;#438;               |
|Ʒ                    | &#38;#439;               |
|Ƹ                    | &#38;#440;               |
|ƹ                    | &#38;#441;               |
|ƺ                    | &#38;#442;               |
|ƻ                    | &#38;#443;               |
|Ƽ                    | &#38;#444;               |
|ƽ                    | &#38;#445;               |
|ƾ                    | &#38;#446;               |
|ƿ                    | &#38;#447;               |
|ǀ                    | &#38;#448;               |
|ǁ                    | &#38;#449;               |
|ǂ                    | &#38;#450;               |
|ǃ                    | &#38;#451;               |
|Ǆ                    | &#38;#452;               |
|ǅ                    | &#38;#453;               |
|ǆ                    | &#38;#454;               |
|Ǉ                    | &#38;#455;               |
|ǈ                    | &#38;#456;               |
|ǉ                    | &#38;#457;               |
|Ǌ                    | &#38;#458;               |
|ǋ                    | &#38;#459;               |
|ǌ                    | &#38;#460;               |
|Ǎ                    | &#38;#461;               |
|ǎ                    | &#38;#462;               |
|Ǐ                    | &#38;#463;               |
|ǐ                    | &#38;#464;               |
|Ǒ                    | &#38;#465;               |
|ǒ                    | &#38;#466;               |
|Ǔ                    | &#38;#467;               |
|ǔ                    | &#38;#468;               |
|Ǖ                    | &#38;#469;               |
|ǖ                    | &#38;#470;               |
|Ǘ                    | &#38;#471;               |
|ǘ                    | &#38;#472;               |
|Ǚ                    | &#38;#473;               |
|ǚ                    | &#38;#474;               |
|Ǜ                    | &#38;#475;               |
|ǜ                    | &#38;#476;               |
|ǝ                    | &#38;#477;               |
|Ǟ                    | &#38;#478;               |
|ǟ                    | &#38;#479;               |
|Ǡ                    | &#38;#480;               |
|ǡ                    | &#38;#481;               |
|Ǣ                    | &#38;#482;               |
|ǣ                    | &#38;#483;               |
|Ǥ                    | &#38;#484;               |
|ǥ                    | &#38;#485;               |
|Ǧ                    | &#38;#486;               |
|ǧ                    | &#38;#487;               |
|Ǩ                    | &#38;#488;               |
|ǩ                    | &#38;#489;               |
|Ǫ                    | &#38;#490;               |
|ǫ                    | &#38;#491;               |
|Ǭ                    | &#38;#492;               |
|ǭ                    | &#38;#493;               |
|Ǯ                    | &#38;#494;               |
|ǯ                    | &#38;#495;               |
|ǰ                    | &#38;#496;               |
|Ǳ                    | &#38;#497;               |
|ǲ                    | &#38;#498;               |
|ǳ                    | &#38;#499;               |
|Ǵ                    | &#38;#500;               |
|ǵ                    | &#38;#501;               |
|Ƕ                    | &#38;#502;               |
|Ƿ                    | &#38;#503;               |
|Ǹ                    | &#38;#504;               |
|ǹ                    | &#38;#505;               |
|Ǻ                    | &#38;#506;               |
|ǻ                    | &#38;#507;               |
|Ǽ                    | &#38;#508;               |
|ǽ                    | &#38;#509;               |
|Ǿ                    | &#38;#510;               |
|ǿ                    | &#38;#511;               |
|Ȁ                    | &#38;#512;               |
|ȁ                    | &#38;#513;               |
|Ȃ                    | &#38;#514;               |
|ȃ                    | &#38;#515;               |
|Ȅ                    | &#38;#516;               |
|ȅ                    | &#38;#517;               |
|Ȇ                    | &#38;#518;               |
|ȇ                    | &#38;#519;               |
|Ȉ                    | &#38;#520;               |
|ȉ                    | &#38;#521;               |
|Ȋ                    | &#38;#522;               |
|ȋ                    | &#38;#523;               |
|Ȍ                    | &#38;#524;               |
|ȍ                    | &#38;#525;               |
|Ȏ                    | &#38;#526;               |
|ȏ                    | &#38;#527;               |
|Ȑ                    | &#38;#528;               |
|ȑ                    | &#38;#529;               |
|Ȓ                    | &#38;#530;               |
|ȓ                    | &#38;#531;               |
|Ȕ                    | &#38;#532;               |
|ȕ                    | &#38;#533;               |
|Ȗ                    | &#38;#534;               |
|ȗ                    | &#38;#535;               |
|Ș                    | &#38;#536;               |
|ș                    | &#38;#537;               |
|Ț                    | &#38;#538;               |
|ț                    | &#38;#539;               |
|Ȝ                    | &#38;#540;               |
|ȝ                    | &#38;#541;               |
|Ȟ                    | &#38;#542;               |
|ȟ                    | &#38;#543;               |
|Ƞ                    | &#38;#544;               |
|ȡ                    | &#38;#545;               |
|Ȣ                    | &#38;#546;               |
|ȣ                    | &#38;#547;               |
|Ȥ                    | &#38;#548;               |
|ȥ                    | &#38;#549;               |
|Ȧ                    | &#38;#550;               |
|ȧ                    | &#38;#551;               |
|Ȩ                    | &#38;#552;               |
|ȩ                    | &#38;#553;               |
|Ȫ                    | &#38;#554;               |
|ȫ                    | &#38;#555;               |
|Ȭ                    | &#38;#556;               |
|ȭ                    | &#38;#557;               |
|Ȯ                    | &#38;#558;               |
|ȯ                    | &#38;#559;               |
|Ȱ                    | &#38;#560;               |
|ȱ                    | &#38;#561;               |
|Ȳ                    | &#38;#562;               |
|ȳ                    | &#38;#563;               |
|ȴ                    | &#38;#564;               |
|ȵ                    | &#38;#565;               |
|ȶ                    | &#38;#566;               |
|ȷ                    | &#38;#567;               |
|ȸ                    | &#38;#568;               |
|ȹ                    | &#38;#569;               |
|Ⱥ                    | &#38;#570;               |
|Ȼ                    | &#38;#571;               |
|ȼ                    | &#38;#572;               |
|Ƚ                    | &#38;#573;               |
|Ⱦ                    | &#38;#574;               |
|ȿ                    | &#38;#575;               |
|ɀ                    | &#38;#576;               |
|Ɂ                    | &#38;#577;               |
|ɂ                    | &#38;#578;               |
|Ƀ                    | &#38;#579;               |
|Ʉ                    | &#38;#580;               |
|Ʌ                    | &#38;#581;               |
|Ɇ                    | &#38;#582;               |
|ɇ                    | &#38;#583;               |
|Ɉ                    | &#38;#584;               |
|ɉ                    | &#38;#585;               |
|Ɋ                    | &#38;#586;               |
|ɋ                    | &#38;#587;               |
|Ɍ                    | &#38;#588;               |
|ɍ                    | &#38;#589;               |
|Ɏ                    | &#38;#590;               |
|ɏ                    | &#38;#591;               |
|ɐ                    | &#38;#592;               |
|ɑ                    | &#38;#593;               |
|ɒ                    | &#38;#594;               |
|ɓ                    | &#38;#595;               |
|ɔ                    | &#38;#596;               |
|ɕ                    | &#38;#597;               |
|ɖ                    | &#38;#598;               |
|ɗ                    | &#38;#599;               |
|ɘ                    | &#38;#600;               |
|ə                    | &#38;#601;               |
|ɚ                    | &#38;#602;               |
|ɛ                    | &#38;#603;               |
|ɜ                    | &#38;#604;               |
|ɝ                    | &#38;#605;               |
|ɞ                    | &#38;#606;               |
|ɟ                    | &#38;#607;               |
|ɠ                    | &#38;#608;               |
|ɡ                    | &#38;#609;               |
|ɢ                    | &#38;#610;               |
|ɣ                    | &#38;#611;               |
|ɤ                    | &#38;#612;               |
|ɥ                    | &#38;#613;               |
|ɦ                    | &#38;#614;               |
|ɧ                    | &#38;#615;               |
|ɨ                    | &#38;#616;               |
|ɩ                    | &#38;#617;               |
|ɪ                    | &#38;#618;               |
|ɫ                    | &#38;#619;               |
|ɬ                    | &#38;#620;               |
|ɭ                    | &#38;#621;               |
|ɮ                    | &#38;#622;               |
|ɯ                    | &#38;#623;               |
|ɰ                    | &#38;#624;               |
|ɱ                    | &#38;#625;               |
|ɲ                    | &#38;#626;               |
|ɳ                    | &#38;#627;               |
|ɴ                    | &#38;#628;               |
|ɵ                    | &#38;#629;               |
|ɶ                    | &#38;#630;               |
|ɷ                    | &#38;#631;               |
|ɸ                    | &#38;#632;               |
|ɹ                    | &#38;#633;               |
|ɺ                    | &#38;#634;               |
|ɻ                    | &#38;#635;               |
|ɼ                    | &#38;#636;               |
|ɽ                    | &#38;#637;               |
|ɾ                    | &#38;#638;               |
|ɿ                    | &#38;#639;               |
|ʀ                    | &#38;#640;               |
|ʁ                    | &#38;#641;               |
|ʂ                    | &#38;#642;               |
|ʃ                    | &#38;#643;               |
|ʄ                    | &#38;#644;               |
|ʅ                    | &#38;#645;               |
|ʆ                    | &#38;#646;               |
|ʇ                    | &#38;#647;               |
|ʈ                    | &#38;#648;               |
|ʉ                    | &#38;#649;               |
|ʊ                    | &#38;#650;               |
|ʋ                    | &#38;#651;               |
|ʌ                    | &#38;#652;               |
|ʍ                    | &#38;#653;               |
|ʎ                    | &#38;#654;               |
|ʏ                    | &#38;#655;               |
|ʐ                    | &#38;#656;               |
|ʑ                    | &#38;#657;               |
|ʒ                    | &#38;#658;               |
|ʓ                    | &#38;#659;               |
|ʔ                    | &#38;#660;               |
|ʕ                    | &#38;#661;               |
|ʖ                    | &#38;#662;               |
|ʗ                    | &#38;#663;               |
|ʘ                    | &#38;#664;               |
|ʙ                    | &#38;#665;               |
|ʚ                    | &#38;#666;               |
|ʛ                    | &#38;#667;               |
|ʜ                    | &#38;#668;               |
|ʝ                    | &#38;#669;               |
|ʞ                    | &#38;#670;               |
|ʟ                    | &#38;#671;               |
|ʠ                    | &#38;#672;               |
|ʡ                    | &#38;#673;               |
|ʢ                    | &#38;#674;               |
|ʣ                    | &#38;#675;               |
|ʤ                    | &#38;#676;               |
|ʥ                    | &#38;#677;               |
|ʦ                    | &#38;#678;               |
|ʧ                    | &#38;#679;               |
|ʨ                    | &#38;#680;               |
|ʩ                    | &#38;#681;               |
|ʪ                    | &#38;#682;               |
|ʫ                    | &#38;#683;               |
|ʬ                    | &#38;#684;               |
|ʭ                    | &#38;#685;               |
|ʮ                    | &#38;#686;               |
|ʯ                    | &#38;#687;               |
|ʰ                    | &#38;#688;               |
|ʱ                    | &#38;#689;               |
|ʲ                    | &#38;#690;               |
|ʳ                    | &#38;#691;               |
|ʴ                    | &#38;#692;               |
|ʵ                    | &#38;#693;               |
|ʶ                    | &#38;#694;               |
|ʷ                    | &#38;#695;               |
|ʸ                    | &#38;#696;               |
|ʹ                    | &#38;#697;               |
|ʺ                    | &#38;#698;               |
|ʻ                    | &#38;#699;               |
|ʼ                    | &#38;#700;               |
|ʽ                    | &#38;#701;               |
|ʾ                    | &#38;#702;               |
|ʿ                    | &#38;#703;               |
|ˀ                    | &#38;#704;               |
|ˁ                    | &#38;#705;               |
|˂                    | &#38;#706;               |
|˃                    | &#38;#707;               |
|˄                    | &#38;#708;               |
|˅                    | &#38;#709;               |
|ˆ                    | &#38;#710;               |
|ˇ                    | &#38;#711;               |
|ˈ                    | &#38;#712;               |
|ˉ                    | &#38;#713;               |
|ˊ                    | &#38;#714;               |
|ˋ                    | &#38;#715;               |
|ˌ                    | &#38;#716;               |
|ˍ                    | &#38;#717;               |
|ˎ                    | &#38;#718;               |
|ˏ                    | &#38;#719;               |
|ː                    | &#38;#720;               |
|ˑ                    | &#38;#721;               |
|˒                    | &#38;#722;               |
|˓                    | &#38;#723;               |
|˔                    | &#38;#724;               |
|˕                    | &#38;#725;               |
|˖                    | &#38;#726;               |
|˗                    | &#38;#727;               |
|˘                    | &#38;#728;               |
|˙                    | &#38;#729;               |
|˚                    | &#38;#730;               |
|˛                    | &#38;#731;               |
|˜                    | &#38;#732;               |
|˝                    | &#38;#733;               |
|˞                    | &#38;#734;               |
|˟                    | &#38;#735;               |
|ˠ                    | &#38;#736;               |
|ˡ                    | &#38;#737;               |
|ˢ                    | &#38;#738;               |
|ˣ                    | &#38;#739;               |
|ˤ                    | &#38;#740;               |
|˥                    | &#38;#741;               |
|˦                    | &#38;#742;               |
|˧                    | &#38;#743;               |
|˨                    | &#38;#744;               |
|˩                    | &#38;#745;               |
|˪                    | &#38;#746;               |
|˫                    | &#38;#747;               |
|ˬ                    | &#38;#748;               |
|˭                    | &#38;#749;               |
|ˮ                    | &#38;#750;               |
|˯                    | &#38;#751;               |
|˰                    | &#38;#752;               |
|˱                    | &#38;#753;               |
|˲                    | &#38;#754;               |
|˳                    | &#38;#755;               |
|˴                    | &#38;#756;               |
|˵                    | &#38;#757;               |
|˶                    | &#38;#758;               |
|˷                    | &#38;#759;               |
|˸                    | &#38;#760;               |
|˹                    | &#38;#761;               |
|˺                    | &#38;#762;               |
|˻                    | &#38;#763;               |
|˼                    | &#38;#764;               |
|˽                    | &#38;#765;               |
|˾                    | &#38;#766;               |
|˿                    | &#38;#767;               |
|̀                    | &#38;#768;               |
|́                    | &#38;#769;               |
|̂                    | &#38;#770;               |
|̃                    | &#38;#771;               |
|̄                    | &#38;#772;               |
|̅                    | &#38;#773;               |
|̆                    | &#38;#774;               |
|̇                    | &#38;#775;               |
|̈                    | &#38;#776;               |
|̉                    | &#38;#777;               |
|̊                    | &#38;#778;               |
|̋                    | &#38;#779;               |
|̌                    | &#38;#780;               |
|̍                    | &#38;#781;               |
|̎                    | &#38;#782;               |
|̏                    | &#38;#783;               |
|̐                    | &#38;#784;               |
|̑                    | &#38;#785;               |
|̒                    | &#38;#786;               |
|̓                    | &#38;#787;               |
|̔                    | &#38;#788;               |
|̕                    | &#38;#789;               |
|̖                    | &#38;#790;               |
|̗                    | &#38;#791;               |
|̘                    | &#38;#792;               |
|̙                    | &#38;#793;               |
|̚                    | &#38;#794;               |
|̛                    | &#38;#795;               |
|̜                    | &#38;#796;               |
|̝                    | &#38;#797;               |
|̞                    | &#38;#798;               |
|̟                    | &#38;#799;               |
|̠                    | &#38;#800;               |
|̡                    | &#38;#801;               |
|̢                    | &#38;#802;               |
|̣                    | &#38;#803;               |
|̤                    | &#38;#804;               |
|̥                    | &#38;#805;               |
|̦                    | &#38;#806;               |
|̧                    | &#38;#807;               |
|̨                    | &#38;#808;               |
|̩                    | &#38;#809;               |
|̪                    | &#38;#810;               |
|̫                    | &#38;#811;               |
|̬                    | &#38;#812;               |
|̭                    | &#38;#813;               |
|̮                    | &#38;#814;               |
|̯                    | &#38;#815;               |
|̰                    | &#38;#816;               |
|̱                    | &#38;#817;               |
|̲                    | &#38;#818;               |
|̳                    | &#38;#819;               |
|̴                    | &#38;#820;               |
|̵                    | &#38;#821;               |
|̶                    | &#38;#822;               |
|̷                    | &#38;#823;               |
|̸                    | &#38;#824;               |
|̹                    | &#38;#825;               |
|̺                    | &#38;#826;               |
|̻                    | &#38;#827;               |
|̼                    | &#38;#828;               |
|̽                    | &#38;#829;               |
|̾                    | &#38;#830;               |
|̿                    | &#38;#831;               |
|̀                    | &#38;#832;               |
|́                    | &#38;#833;               |
|͂                    | &#38;#834;               |
|̓                    | &#38;#835;               |
|̈́                    | &#38;#836;               |
|ͅ                    | &#38;#837;               |
|͆                    | &#38;#838;               |
|͇                    | &#38;#839;               |
|͈                    | &#38;#840;               |
|͉                    | &#38;#841;               |
|͊                    | &#38;#842;               |
|͋                    | &#38;#843;               |
|͌                    | &#38;#844;               |
|͍                    | &#38;#845;               |
|͎                    | &#38;#846;               |
|͏                    | &#38;#847;               |
|͐                    | &#38;#848;               |
|͑                    | &#38;#849;               |
|͒                    | &#38;#850;               |
|͓                    | &#38;#851;               |
|͔                    | &#38;#852;               |
|͕                    | &#38;#853;               |
|͖                    | &#38;#854;               |
|͗                    | &#38;#855;               |
|͘                    | &#38;#856;               |
|͙                    | &#38;#857;               |
|͚                    | &#38;#858;               |
|͛                    | &#38;#859;               |
|͜                    | &#38;#860;               |
|͝                    | &#38;#861;               |
|͞                    | &#38;#862;               |
|͟                    | &#38;#863;               |
|͠                    | &#38;#864;               |
|͡                    | &#38;#865;               |
|͢                    | &#38;#866;               |
|ͣ                    | &#38;#867;               |
|ͤ                    | &#38;#868;               |
|ͥ                    | &#38;#869;               |
|ͦ                    | &#38;#870;               |
|ͧ                    | &#38;#871;               |
|ͨ                    | &#38;#872;               |
|ͩ                    | &#38;#873;               |
|ͪ                    | &#38;#874;               |
|ͫ                    | &#38;#875;               |
|ͬ                    | &#38;#876;               |
|ͭ                    | &#38;#877;               |
|ͮ                    | &#38;#878;               |
|ͯ                    | &#38;#879;               |
|Ͱ                    | &#38;#880;               |
|ͱ                    | &#38;#881;               |
|Ͳ                    | &#38;#882;               |
|ͳ                    | &#38;#883;               |
|ʹ                    | &#38;#884;               |
|͵                    | &#38;#885;               |
|Ͷ                    | &#38;#886;               |
|ͷ                    | &#38;#887;               |
|͸                    | &#38;#888;               |
|͹                    | &#38;#889;               |
|ͺ                    | &#38;#890;               |
|ͻ                    | &#38;#891;               |
|ͼ                    | &#38;#892;               |
|ͽ                    | &#38;#893;               |
|;                    | &#38;#894;               |
|Ϳ                    | &#38;#895;               |
|΀                    | &#38;#896;               |
|΁                    | &#38;#897;               |
|΂                    | &#38;#898;               |
|΃                    | &#38;#899;               |
|΄                    | &#38;#900;               |
|΅                    | &#38;#901;               |
|Ά                    | &#38;#902;               |
|·                    | &#38;#903;               |
|Έ                    | &#38;#904;               |
|Ή                    | &#38;#905;               |
|Ί                    | &#38;#906;               |
|΋                    | &#38;#907;               |
|Ό                    | &#38;#908;               |
|΍                    | &#38;#909;               |
|Ύ                    | &#38;#910;               |
|Ώ                    | &#38;#911;               |
|ΐ                    | &#38;#912;               |
|Α                    | &#38;#913;               |
|Β                    | &#38;#914;               |
|Γ                    | &#38;#915;               |
|Δ                    | &#38;#916;               |
|Ε                    | &#38;#917;               |
|Ζ                    | &#38;#918;               |
|Η                    | &#38;#919;               |
|Θ                    | &#38;#920;               |
|Ι                    | &#38;#921;               |
|Κ                    | &#38;#922;               |
|Λ                    | &#38;#923;               |
|Μ                    | &#38;#924;               |
|Ν                    | &#38;#925;               |
|Ξ                    | &#38;#926;               |
|Ο                    | &#38;#927;               |
|Π                    | &#38;#928;               |
|Ρ                    | &#38;#929;               |
|΢                    | &#38;#930;               |
|Σ                    | &#38;#931;               |
|Τ                    | &#38;#932;               |
|Υ                    | &#38;#933;               |
|Φ                    | &#38;#934;               |
|Χ                    | &#38;#935;               |
|Ψ                    | &#38;#936;               |
|Ω                    | &#38;#937;               |
|Ϊ                    | &#38;#938;               |
|Ϋ                    | &#38;#939;               |
|ά                    | &#38;#940;               |
|έ                    | &#38;#941;               |
|ή                    | &#38;#942;               |
|ί                    | &#38;#943;               |
|ΰ                    | &#38;#944;               |
|α                    | &#38;#945;               |
|β                    | &#38;#946;               |
|γ                    | &#38;#947;               |
|δ                    | &#38;#948;               |
|ε                    | &#38;#949;               |
|ζ                    | &#38;#950;               |
|η                    | &#38;#951;               |
|θ                    | &#38;#952;               |
|ι                    | &#38;#953;               |
|κ                    | &#38;#954;               |
|λ                    | &#38;#955;               |
|μ                    | &#38;#956;               |
|ν                    | &#38;#957;               |
|ξ                    | &#38;#958;               |
|ο                    | &#38;#959;               |
|π                    | &#38;#960;               |
|ρ                    | &#38;#961;               |
|ς                    | &#38;#962;               |
|σ                    | &#38;#963;               |
|τ                    | &#38;#964;               |
|υ                    | &#38;#965;               |
|φ                    | &#38;#966;               |
|χ                    | &#38;#967;               |
|ψ                    | &#38;#968;               |
|ω                    | &#38;#969;               |
|ϊ                    | &#38;#970;               |
|ϋ                    | &#38;#971;               |
|ό                    | &#38;#972;               |
|ύ                    | &#38;#973;               |
|ώ                    | &#38;#974;               |
|Ϗ                    | &#38;#975;               |
|ϐ                    | &#38;#976;               |
|ϑ                    | &#38;#977;               |
|ϒ                    | &#38;#978;               |
|ϓ                    | &#38;#979;               |
|ϔ                    | &#38;#980;               |
|ϕ                    | &#38;#981;               |
|ϖ                    | &#38;#982;               |
|ϗ                    | &#38;#983;               |
|Ϙ                    | &#38;#984;               |
|ϙ                    | &#38;#985;               |
|Ϛ                    | &#38;#986;               |
|ϛ                    | &#38;#987;               |
|Ϝ                    | &#38;#988;               |
|ϝ                    | &#38;#989;               |
|Ϟ                    | &#38;#990;               |
|ϟ                    | &#38;#991;               |
|Ϡ                    | &#38;#992;               |
|ϡ                    | &#38;#993;               |
|Ϣ                    | &#38;#994;               |
|ϣ                    | &#38;#995;               |
|Ϥ                    | &#38;#996;               |
|ϥ                    | &#38;#997;               |
|Ϧ                    | &#38;#998;               |
|ϧ                    | &#38;#999;               |




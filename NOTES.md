## Spring Data Jpa Notes

**Generation tables with Spring Data Jpa** - we should create POJO classes:

```java
@Entity
@Data
@Table(name = "accounts")
public class Account {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(name = "email_address")
    private String emailAddress;

    @OneToMany(mappedBy = "account", cascade = CascadeType.ALL)
    private List<AccountSettings> accountSettings = new ArrayList<>();

    public Account() {
    }

    public Account(String name, String emailAddress, List<AccountSettings> accountSettings) {
        this.name = name;
        this.emailAddress = emailAddress;
        this.accountSettings = accountSettings;
    }
}
```
and
```java
@Entity
@Data
@Table(name = "account_settings")
public class AccountSettings {

    @Id
    @GeneratedValue
    private Long id;

    @Column(name = "name", nullable = false)
    private String settingName;

    @Column(name = "value", nullable = false)
    private String settingValue;

    @ManyToOne
    @JoinColumn(name = "account_id", nullable = false)
    private Account account;

    public AccountSettings() {
    }

    public AccountSettings(String settingName, String settingValue, Account account) {
        this.settingName = settingName;
        this.settingValue = settingValue;
        this.account = account;
    }
}
```
then connect to DB and fill the application.properties file:
```properties
spring.jpa.properties.javax.persistence.schema-generation.scripts.action=create
spring.jpa.properties.javax.persistence.schema-generation.scripts.create-target=src/main/resources/create.sql
spring.jpa.properties.javax.persistence.schema-generation.scripts.create-source=metadata
# MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/demo_mysql?useJvmCharsetConverters=true?useUnicode=true&serverTimezone=UTC&useSSL=true&verifyServerCertificate=false
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=root
# JPA
spring.jpa.database=mysql
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
```
result - the file with SQL scripts - created:
```sql
create table account_settings (
    id bigint not null,
    name varchar(255) not null,
    value varchar(255) not null,
    account_id bigint not null,
    primary key (id)
                              ) engine=InnoDB;

create table accounts (
    id bigint not null,
    email_address varchar(255),
    name varchar(100) not null,
    primary key (id)
                      ) engine=InnoDB;

create table hibernate_sequence (next_val bigint) engine=InnoDB;

insert into hibernate_sequence values ( 1 );

alter table account_settings add constraint FK54uo82jnot7ye32pyc8dcj2eh foreign key
    (account_id) references accounts (id);
```
### Getting Started with Spring Data JPA

```properties
spring.jpa.hibernate.ddl-auto=update//updates all tables in DB
```
Always use import javax.persistence.*; not hibernate imports.

@Entity(name = "Student") - mapping class to DB, should have @Id necessarily.Minimum what we need in order to map class to DB.
The name attribute will use in JPQL queries.

@Table - mapping class to certain table in DB.

@Column - mapping field to column in table.

@Id - marks a field in a model class as the primary key:

```java
class Person {
    @Id
    Long id;

    // ...
}
```

Since it's implementation-independent, it makes a model class easy to use with multiple data store engines.

@Transient - we can use this annotation to mark a field in a model class as transient. Hence the data store engine won't read or write this field's value:

```java
class Person {
    @Transient
    int age;
}
```
Like @Id, @Transient is also implementation-independent, which makes it convenient to use with multiple data store implementations.

### Querying Data

@Query - allows write JPQL queries as well as native queries.

- ####  JPQL

By default, the query definition uses JPQL.
Let's look at a simple repository method that returns active User entities from the database:
```springdataql
@Query("SELECT u FROM User u WHERE u.status = 1")
Collection<User> findAllActiveUsers();
```

- ####  Native

We can use also native SQL to define our query. All we have to do is set the value of the nativeQuery attribute to true and define the native SQL query in the value attribute of the annotation:

```springdataql
@Query(
value = "SELECT * FROM USERS u WHERE u.status = 1",
nativeQuery = true)
Collection<User> findAllActiveUsersNative();
```
Better to use JPQL because if we change DB we should not change queries.

##### @Param

We can also pass method parameters to the query using named parameters. We define these using the **@Param** annotation inside our repository method declaration.

Each parameter annotated with @Param must have a value string matching the corresponding JPQL or SQL query parameter name. A query with named parameters is easier to read and is less error-prone in case the query needs to be refactored.

- JPQL

We use the @Param annotation in the method declaration to match parameters defined by name in JPQL with parameters from the method declaration:

```springdataql
@Query("SELECT u FROM User u WHERE u.status = :status and u.name = :name")
User findUserByStatusAndNameNamedParams(
@Param("status") Integer status,
@Param("name") String name);
```

- Native

```springdataql
@Query(value = "SELECT * FROM Users u WHERE u.status = :status and u.name = :name", 
  nativeQuery = true)
User findUserByStatusAndNameNamedParamsNative(
  @Param("status") Integer status, @Param("name") String name);
```

### @Modifying

The @Modifying annotation is used to enhance the @Query annotation so that we can execute not only SELECT queries, but also INSERT, UPDATE, DELETE, and even DDL queries.

First, let's look at an example of a @Modifying UPDATE query:

```springdataql
@Modifying
@Query("update User u set u.active = false where u.lastLoginDate < :date")
void deactivateUsersNotLoggedInSince(@Param("date") LocalDate date);
```

Here we're deactivating the users that haven't logged in since a given date.

Let's try another one, where we'll delete deactivated users:


```springdataql
@Modifying
@Query("delete User u where u.active = false")
int deleteDeactivatedUsers();
```

As we can see, this method returns an integer. It's a feature of Spring Data JPA @Modifying queries that provides us with the number of updated entities.

### Sorting and Pagination

For the methods we get out of the box such as findAll(Sort) or the ones that are generated by parsing method signatures, we can only use object properties to define our sort:

```springdataql
userRepository.findAll(Sort.by(Sort.Direction.ASC, "name"));
```

Now imagine that we want to sort by the length of a name property:

```springdataql
userRepository.findAll(Sort.by("LENGTH(name)"));
```

When we execute the above code, we'll receive an exception:

```springdataql
org.springframework.data.mapping.PropertyReferenceException: No property LENGTH(name) found for type User!
```

- JPQL

  When we use JPQL for a query definition, then Spring Data can handle sorting without any problem — all we have to do is add a method parameter of type Sort:

```springdataql
@Query(value = "SELECT u FROM User u")
List<User> findAllUsers(Sort sort);
```

- Native

When the @Query annotation uses native SQL, then it's not possible to define a Sort.

If we do, we'll receive an exception:

```springdataql
org.springframework.data.jpa.repository.query.InvalidJpaQueryMethodException: Cannot use native queries with dynamic sorting and/or pagination

```

As the exception says, the sort isn't supported for native queries. The error message gives us a hint that pagination will cause an exception too.

#### Pagination

- JPQL

Using pagination in the JPQL query definition is straightforward:

```springdataql
@Query(value = "SELECT u FROM User u ORDER BY id")
Page<User> findAllUsersWithPagination(Pageable pageable);
```
We can pass a PageRequest parameter to get a page of data.

Pagination is also supported for native queries but requires a little bit of additional work.

- Native

We can enable pagination for native queries by declaring an additional attribute countQuery.

This defines the SQL to execute to count the number of rows in the whole result:

```springdataql
@Query(
value = "SELECT * FROM Users ORDER BY id",
countQuery = "SELECT count(*) FROM Users",
nativeQuery = true)
Page<User> findAllUsersWithPagination(Pageable pageable);
```

### One to One Relationships


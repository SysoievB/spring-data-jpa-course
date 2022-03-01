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

1. **Modeling With a Foreign Key**

![This is an image](https://www.baeldung.com/wp-content/uploads/2018/12/1-1_FK.png)

In this example, the address_id column in users is the foreign key to address.

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id")
    private Long id;
    //... 

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "address_id", referencedColumnName = "id")
    private Address address;

    // ... getters and setters
}

@Entity
@Table(name = "address")
public class Address {

  @Id
  @GeneratedValue(strategy = GenerationType.AUTO)
  @Column(name = "id")
  private Long id;
  //...

  @OneToOne(mappedBy = "address")
  private User user;

  //... getters and setters
}
```

2. **Modeling With a Shared Primary Key** 


In this strategy, instead of creating a new column address_id, we'll mark the primary key column (user_id) of the address table as the foreign key to the users table:


![This is an image](https://www.baeldung.com/wp-content/uploads/2018/12/1-1-SK.png)

An ER diagram with Users Tied to Addresses where they share the same primary key values
We've optimized the storage space by utilizing the fact that these entities have a one-to-one relationship between them.

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id")
    private Long id;

    //...

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    @PrimaryKeyJoinColumn
    private Address address;

    //... getters and setters
}
@Entity
@Table(name = "address")
public class Address {

    @Id
    @Column(name = "user_id")
    private Long id;

    //...

    @OneToOne
    @MapsId
    @JoinColumn(name = "user_id")
    private User user;
   
    //... getters and setters
}
```

The mappedBy attribute is now moved to the User class since the foreign key is now present in the address table. We've also added the @PrimaryKeyJoinColumn annotation, which indicates that the primary key of the User entity is used as the foreign key value for the associated Address entity.

We still have to define an @Id field in the Address class. But note that this references the user_id column, and it no longer uses the @GeneratedValue annotation. Also, on the field that references the User, we've added the @MapsId annotation, which indicates that the primary key values will be copied from the User entity.

3. **Modeling With a Join Table**

The strategies that we have discussed until now force us to put null values in the column to handle optional relationships.

Typically, we think of many-to-many relationships when we consider a join table, but using a join table in this case can help us to eliminate these null values:

![This is an image](https://www.baeldung.com/wp-content/uploads/2018/12/1-1-JT.png)

```java
@Entity
@Table(name = "employee")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id")
    private Long id;

    //...

    @OneToOne(cascade = CascadeType.ALL)
    @JoinTable(name = "emp_workstation", 
      joinColumns = 
        { @JoinColumn(name = "employee_id", referencedColumnName = "id") },
      inverseJoinColumns = 
        { @JoinColumn(name = "workstation_id", referencedColumnName = "id") })
    private WorkStation workStation;

    //... getters and setters
}
@Entity
@Table(name = "workstation")
public class WorkStation {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id")
    private Long id;

    //...

    @OneToOne(mappedBy = "workStation")
    private Employee employee;

    //... getters and setters
}
```

@JoinTable instructs Hibernate to employ the join table strategy while maintaining the relationship.

Also, Employee is the owner of this relationship, as we chose to use the join table annotation on it.

#### Difference between FetchType LAZY and EAGER in Java Persistence API not for OneToOne

```java
LAZY//fetch when needed
EAGER//fetch immediately
```

#### Cascade Type

- CascadeType.PERSIST  – It means if the parent entity is saved then the related entity will also be saved.
- CascadeType.MERGE  – It means if the parent entity is merged then the related entity will also be merged.
- CascadeType.REMOVE  – It means if the parent entity is deleted then the related entity will also be deleted.
- CascadeType.DETACH  – It means if the parent entity is detached then the related entity will also be detached.
- CascadeType.REFRESH  – It means if the parent entity is refreshed then the related entity will also be refreshed.
- CascadeType.ALL  – It is equivalent to cascade={DETACH, MERGE, PERSIST, REFRESH, REMOVE}.

#### Orphan removal & Cascading Remove

Marking a reference field with CascadeType.REMOVE (or CascadeType.ALL, which includes REMOVE) indicates that remove operations should be cascaded automatically to entity objects that are referenced by that field (multiple entity objects can be referenced by a collection field):

```java
@Entity
class Employee {

@OneToOne(cascade=CascadeType.REMOVE)
private Address address;

}
```
In the example above, the Employee entity class contains an address field that references an instance of Address, which is another entity class. Due to the CascadeType.REMOVE setting, when an Employee instance is removed the operation is automatically cascaded to the referenced Address instance, which is then automatically removed as well. Cascading may continue recursively when applicable (e.g. to entity objects that the Address object references, if any).

JPA 2 supports an additional and more aggressive remove cascading mode which can be specified using the orphanRemoval element of the @OneToOne and @OneToMany annotations:

```java
@Entity
class Employee {
     
    @OneToOne(orphanRemoval=true)
    private Address address;
     
}
```

When an Employee entity object is removed the remove operation is cascaded to the referenced Address entity object. In this regard, orphanRemoval=true and cascade=CascadeType.REMOVE are identical, and if orphanRemoval=true is specified, CascadeType.REMOVE is redundant.

The difference between the two settings is in the response to disconnecting a relationship. For example, such as when setting the address field to null or to another Address object.

- If orphanRemoval=true is specified the disconnected Address instance is automatically removed. This is useful for cleaning up dependent objects (e.g. Address) that should not exist without a reference from an owner object (e.g. Employee).
- If only cascade=CascadeType.REMOVE is specified no automatic action is taken since disconnecting a relationship is not a remove operation.

#### @ForeignKey

```java
import javax.persistence.*;
import java.time.LocalDateTime;

import static javax.persistence.GenerationType.SEQUENCE;

@Entity(name = "Book")
@Table(name = "book")
public class Book {

    @Id
    @SequenceGenerator(
            name = "book_sequence",
            sequenceName = "book_sequence",
            allocationSize = 1
    )
    @GeneratedValue(
            strategy = SEQUENCE,
            generator = "book_sequence"
    )
    @Column(
            name = "id",
            updatable = false
    )
    private Long id;

    @Column(
            name = "created_at",
            nullable = false,
            columnDefinition = "TIMESTAMP WITHOUT TIME ZONE"
    )
    private LocalDateTime createdAt;

    @Column(
            name = "book_name",
            nullable = false
    )
    private String bookName;

    @ManyToOne
    @JoinColumn(
            name = "student_id",
            nullable = false,
            referencedColumnName = "id",
            foreignKey = @ForeignKey(
                    name = "student_book_fk"
            )
    )
    private Student student;

    public Book() {
    }

    public Book(LocalDateTime createdAt, String bookName, Student student) {
        this.createdAt = createdAt;
        this.bookName = bookName;
        this.student = student;
    }

}


@Entity(name = "Student")
@Table(
        name = "student",
        uniqueConstraints = {
                @UniqueConstraint(name = "student_email_unique", columnNames = "email")
        }
)
public class Student {

    @Id
    @SequenceGenerator(
            name = "student_sequence",
            sequenceName = "student_sequence",
            allocationSize = 1
    )
    @GeneratedValue(
            strategy = SEQUENCE,
            generator = "student_sequence"
    )
    @Column(
            name = "id"
    )
    private Long id;

    @Column(
            name = "first_name",
            nullable = false,
            columnDefinition = "TEXT"
    )
    private String firstName;

    @Column(
            name = "last_name",
            nullable = false,
            columnDefinition = "TEXT"
    )
    private String lastName;

    @Column(
            name = "email",
            nullable = false,
            columnDefinition = "TEXT"
    )
    private String email;

    @Column(
            name = "age",
            nullable = false

    )
    private Integer age;

    @OneToOne(
            mappedBy = "student",
            orphanRemoval = true
    )
    private StudentIdCard studentIdCard;

    @OneToMany(
            mappedBy = "student",
            orphanRemoval = true,
            cascade = CascadeType.ALL
    )
    private final List<Book> books = new ArrayList<>();

    public Student(String firstName,
                   String lastName,
                   String email,
                   Integer age) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.age = age;
    }

    public Student() {
}
```

### One to Many Relationships

Simply put, one-to-many mapping means that one row in a table is mapped to multiple rows in another table.

![image](https://www.baeldung.com/wp-content/uploads/2017/02/C-1.png)

One cart can have many items, so here we have a one-to-many mapping.

```java
public class Cart {

    //...     
 
    @OneToMany(mappedBy="cart")
    private Set<Items> items;
	
    //...
}
```

#### @OneToMany Annotation
A one-to-many relationship between two entities is defined by using the @OneToMany annotation in Spring Data JPA. It declares the mappedBy element to indicate the entity that owns the bidirectional relationship. Usually, the child entity is one that owns the relationship and the parent entity contains the @OneToMany annotation.

#### @ManyToOne Annotation
The @ManyToOne annotation is used to define a many-to-one relationship between two entities in Spring Data JPA. The child entity, that has the join column, is called the owner of the relationship defined using the @ManyToOne annotation.

#### @JoinColumn Annotation
The @JoinColumn annotation is used to specify the foreign key column in the owner of the relationship. The inverse-side of the relationship sets the mappedBy attribute to indicate that the relationship is owned by the other entity.

```java
@Entity
@Table(name = "books")
public class Book implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String author;
    @Column(unique = true)
    private String isbn;

    @OneToMany(mappedBy = "book", fetch = FetchType.LAZY,
            cascade = CascadeType.ALL)
    private Set<Page> pages;

    public Book() {
    }

    public Book(String title, String author, String isbn) {
        this.title = title;
        this.author = author;
        this.isbn = isbn;
    }

    // getters and setters, equals(), toString() .... (omitted for brevity)
}

@Entity
@Table(name = "pages")
public class Page implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private int number;
    private String content;
    private String chapter;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "book_id", nullable = false)
    private Book book;

    public Page() {
    }

    public Page(int number, String content, String chapter, Book book) {
        this.number = number;
        this.content = content;
        this.chapter = chapter;
        this.book = book;
    }

    // getters and setters, equals(), toString() .... (omitted for brevity)
}
```

### Many to Many Relationships

- **Using Join table**

![image](https://www.baeldung.com/wp-content/uploads/2018/11/simple-model-updated.png)

Such a table is called a join table. In a join table, the combination of the foreign keys will be its composite primary key.

```java
@Entity
class Student {

    @Id
    Long id;

  @ManyToMany
  @JoinTable(
          name = "course_like",
          joinColumns = @JoinColumn(name = "student_id"),
          inverseJoinColumns = @JoinColumn(name = "course_id"))
  Set<Course> likedCourses;

    // additional properties
    // standard constructors, getters, and setters
}

@Entity
class Course {

    @Id
    Long id;

  @ManyToMany(mappedBy = "likedCourses")
  Set<Student> likes;

    // additional properties
    // standard constructors, getters, and setters
}
```

- **Using a composite key**

We can model it almost the same way as the simple many-to-many relationship. The only difference is that we attach a new attribute to the join table:

![image](https://www.baeldung.com/wp-content/uploads/2018/11/relation-attribute-model-updated.png)

Note that a composite key class has to fulfill some key requirements:

- We have to mark it with @Embeddable.
- It has to implement java.io.Serializable.
- We need to provide an implementation of the hashcode() and equals() methods.
- None of the fields can be an entity themselves.

```java
@Embeddable
class CourseRatingKey implements Serializable {

    @Column(name = "student_id")
    Long studentId;

    @Column(name = "course_id")
    Long courseId;

    // standard constructors, getters, and setters
    // hashcode and equals implementation
}
@Entity
class CourseRating {

  @EmbeddedId
  CourseRatingKey id;

  @ManyToOne
  @MapsId("studentId")
  @JoinColumn(name = "student_id")
  Student student;

  @ManyToOne
  @MapsId("courseId")
  @JoinColumn(name = "course_id")
  Course course;

  int rating;

  // standard constructors, getters, and setters
}
```

This code is very similar to a regular entity implementation. However, we have some key differences:

We used @EmbeddedId to mark the primary key, which is an instance of the CourseRatingKey class.
We marked the student and course fields with @MapsId.
@MapsId means that we tie those fields to a part of the key, and they're the foreign keys of a many-to-one relationship. We need it because, as we mentioned, we can't have entities in the composite key.

- **With a new entity**

![image](https://www.baeldung.com/wp-content/uploads/2018/11/relation-entity-model-updated.png)

Since the course_registration became a regular table, we can create a plain old JPA entity modeling it:

```java
@Entity
class CourseRegistration {

    @Id
    Long id;

    @ManyToOne
    @JoinColumn(name = "student_id")
    Student student;

    @ManyToOne
    @JoinColumn(name = "course_id")
    Course course;

    LocalDateTime registeredAt;

    int grade;
    
    // additional properties
    // standard constructors, getters, and setters
}
//We also need to configure the relationships in the Student and Course classes:
class Student {

    // ...

    @OneToMany(mappedBy = "student")
    Set<CourseRegistration> registrations;

    // ...
}

class Course {

    // ...

    @OneToMany(mappedBy = "courses")
    Set<CourseRegistration> registrations;

    // ...
}
```
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

@Entity - mapping class to DB, should have @Id necessarily.Minimum what we need in order to map class to DB.

@Table - 

@Column - 
# Database schema changes with Spring Boot and Hibernate

## Let's refactor!!!!

When writing applications, we tend to promote practices such as [TDD](https://en.wikipedia.org/wiki/Test-driven_development) that allow us to be confident when refactoring our code. 
What about our database? Should we be able to refactor our production database schema in a jiffy, just as easily as we would change a method name? May be not exactly, but let's try to get there!

## Setup

Here we will discuss the various ways to update your database schema in production for a `Spring Boot` / `JPA` / `Hibernate` application in Java.
All code samples are available in [our dedicated github repository](https://github.com/michaelisvy/java-db-schema-updates).

To start with, we have a `Spring Boot` application that uses `H2` for `JUnit` tests and connects to `MySql` as its staging/production database. 

```.properties
spring.datasource.url=jdbc:mysql://localhost:3306/addressBook
spring.datasource.username=santa
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
```

The above example shows our `application-mysql.properties` file. 
The first 3 lines explain how to connect to MySql. Our password is hardcoded for simplicity sake, but in real life we would store it in a secret.

The last line shows that our MySql schema should be updated at application startup (to be discussed in the next paragraph). 


## Generating a Database schema from scratch

At this stage, our application only has a single entity class called `User`.

```java
@Entity @Data
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    private String firstName;
    private String lastName;
    private LocalDate dateOfBirth;
}
```
> Note: the @Data annotation comes from [Lombok](https://projectlombok.org/) and auto-generates our getter/setter methods.

As seen in the previous section, we have configured database schema auto-update as follows:

```.properties
spring.jpa.hibernate.ddl-auto=update
```

At application startup, the following query is run:

```sql
create table user (
id integer not null auto_increment, 
date_of_birth date, 
first_name varchar(255), 
last_name varchar(255), 
primary key (id)) engine=InnoDB
```

How does it work? At startup time, Hibernate parses the `User` class and generates a table based on the information it found (class name, attribute names and types, annotations…).
A `create table` query is generated based on our database's `dialect`.

Our application does not specify any dialect in its configuration because it relies on the default one. We can see the dialect used in the application startup logs:
```
HHH000400: Using dialect: org.hibernate.dialect.MySQL8Dialect
```

## Adding a non-conflicting change to a database
Let us now add the `Address` entity to our model.

```java
@Data @Entity
public class Address {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String streetAddress;
    //...
}
```

We are also adding a relationship from `User` to `Address` as follows:

```java
@Entity @Data
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    private String firstName;
    private String lastName;
    private LocalDate dateOfBirth;

    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = "user_id", foreignKey = @ForeignKey(name="FK_USER_ID"))
    private List<Address> addressList = new ArrayList<>();
}
```

When the application starts (still in `auto-update` mode), Hibernate creates the `Address` table as follows:

```sql
create table address (
       id integer not null auto_increment,
        city varchar(255),
        street_address varchar(255),
        zip_code varchar(255),
        user_id integer,
        primary key (id)
    ) engine=InnoDB

 alter table address 
       add constraint FK_USER_ID 
       foreign key (user_id) 
       references user (id)
```

Our non-conflicting change has been added as expected. 

It is fine to use `auto-update` even for a change in production. However, you should always backup your database and plan for a restore procedure. In MySql, that can be done with the [mysqldump](https://www.thegeekstuff.com/2008/09/backup-and-restore-mysql-database-using-mysqldump/) command.

## Adding a conflicting change
Let’s now consider that we would like to rename the `address` table into `postal_address`. We are adding the `@Table` annotation as follows:

```java
@Data @Entity @Table(name="postal_address")
public class Address {
    //...
}
```

Doing such a change using Hibernate’s `auto-update` feature would create the following behaviour:
* Hibernate leaves aside the existing `address` table and creates a new one called `postal_address`
* All data inside `address` stay with `address`. In the future, all new data will be created inside `postal_address`

If you have an existing production database, the above behaviour is *not* what you’re after. 
Let's disable auto-schema generation: 

```.properties
spring.jpa.hibernate.ddl-auto=validate
```

> Note: at startup time, Hibernate will now `validate` that the database schema is compatible with our `JPA/Hibernate` mapping. 

When starting the application in `staging`, we now see the following error:

```java
Caused by: org.hibernate.tool.schema.spi.SchemaManagementException: 
Schema-validation: missing table [postal_address]
	at org.hibernate.tool.schema.internal.AbstractSchemaValidator
    .validateTable(AbstractSchemaValidator.java:121)
```

In order to fix this issue, we need to run the below `sql` query:
```.sql
RENAME TABLE address to postal_address
```

Note: as usual, do not forget to backup your database before making any change in production!


## Using Liquibase for a conflicting change

We've seen in the previous section that you can run an sql script and use the setting `ddl-auto=validate` when adding a conflicting change to your database (renaming a table, renaming a column, deleting a table etc). 

You can also use a database migration tool such as [Liquibase](https://www.liquibase.org/) or [Flyway](https://flywaydb.org/). In this section, we will show how to work with `Liquibase` for that.

`Liquibase` allows to keep track of database schema changes so you have a version number for each and every schema change that you make in your project lifecycle. 
With `Liquibase`, we can rename `address` into `postal_address` using the below script:

```xml
<databaseChangeLog  ...>
   <changeSet author="Michael" id="1.2">
       <renameTable oldTableName="address" newTableName="postal_address" />
   </changeSet>
</databaseChangeLog>
```
(file `db.changelog-master.xml`)

As you can notice:
* Each change has a unique version number (set as `1,2` in our example)
* The change author is tracked manually

`Liquibase` is seamlessly integrated into `Spring Boot`. You just need 2 configuration steps.

1) pom.xml
If you're using `Maven`, you just have to add the following dependency:
```xml
<dependency>
   <groupId>org.liquibase</groupId>
   <artifactId>liquibase-core</artifactId>
</dependency>
```
(file pom.xml)

2)  application-mysql.properties
You can add the below lines into your `Spring Boot` configuration file. 

```.properties
spring.liquibase.enabled=true
spring.liquibase.change-log=classpath:/db/changelog/db.changelog-master.xml
```

When starting our application, we can see in the logs that our changes have been made:
```
liquibase.changelog.ChangeSet            : Table address renamed to postal_address
```

Liquibase also creates a table called `databasechangelog` and tracks all changes inside it:

liquibase.executor.jvm.JdbcExecutor      : INSERT INTO addressBook.DATABASECHANGELOG (ID, AUTHOR, FILENAME, ..., `DESCRIPTION`, ...) 
VALUES ('1.3', 'Michael', 'classpath:/db/changelog/db.changelog-master.xml', ..., 'renameTable newTableName=postal_address, oldTableName=address', ...)

On the long term, we will be able to track all changes happening to our database inside this table.

## Going further with Liquibase

## Conclusion
we are a startup. Big companies: might not be able to update production tables
The above also implies that you are able to take your application offline for a few minutes. Would be interesting to get feedback on how it's done for apps that need to be up 24/7.

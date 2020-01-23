# Database schema changes with Spring Boot and Hibernate

## Target audience
This article has been written for people who have at least basic understanding of `Java` and its most common backend frameworks (`Spring Boot`, `Spring`, `Hibernate`...). All examples are using MySql and can easily be migrated to a different relational database.

## Introduction

The Java ecosystem gives you a lot of tools to magically update your database schemas. Should they be considered as development tools or should they also be used in production? 
In this first article, we will focus on general best practices and `Hibernate`'s auto-schema generation feature. We will explain what we've learned from it and where it is suitable to be used.
In a subsequent article (to be published soon), we will discuss how database schema changes can be done with a database migration tool such as `Liquibase`.

## Setup

Here we will discuss the various ways to update your database schema in production for a `Spring Boot` / `JPA` / `Hibernate` application in Java.
All code samples are available in [our dedicated github repository](https://github.com/michaelisvy/java-db-schema-updates).

Let's first create a new database schema using the `MySql` command-line client:

```
>mysql -u santa -p
Enter password: ******
mysql> CREATE DATABASE addressBook;
Query OK, 1 row affected (0.12 sec)
```


Let's now open our application. We have a `Spring Boot` application that uses `MySql` as its staging/production database. 

```.properties
spring.datasource.url=jdbc:mysql://localhost:3306/addressBook
spring.datasource.username=santa
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
```

The above example shows our `application.properties` file. 
The first 3 lines explain how to connect to MySql. Our password is hardcoded for simplicity sake, but in real life we would store it in a secret.

The last line shows that our MySql schema should be updated at application startup (to be discussed in the next paragraph). 


## Generating a Database schema from scratch

At this stage, our database schema has just been created. Our application only has a single entity class called `User`.

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

How does it work? At startup time, `Hibernate` parses the `User` class and generates a table based on the information it found (class name, attribute names and types, annotations…).
A `create table` query is generated based on our database's `dialect`.

Inside our `pom.xml` we have configured the `mysql` jdbc driver as a dependency. `Spring Boot` then assumes that we use the default `MySql` dialect as shown in the logs:
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

## Should use use Hibernate's auto-update feature on a production database?
It is fine to use it for `development` and staging `databases`. 
However, the Hibernate team recommends that you should be more cautious when working with a `production` database. 
In their [official documentation](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#schema-generation) they say:
```
Although the automatic schema generation is very useful for testing and prototyping purposes, in a production environment, it’s much more flexible to manage the schema using incremental migration scripts.
```

Here is the flow that we typically use in our teams:
* For Unit tests, the whole database is created in memory at startup time (`create-drop`).
* For tests on a local development environment, we run `update` and save all the update scripts that have been generated (such as for the Address table in our example). 
* In a staging environment, we use `validate`. We try to replicate the behaviour that we will have in production. We therefore update our schema manually using the scripts collected in our local dev environment. 
* In production: we add a backup/restore procedure.

you should always backup your database and plan for a restore procedure. In MySql, that can be done with the [mysqldump](https://www.thegeekstuff.com/2008/09/backup-and-restore-mysql-database-using-mysqldump/) command.

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

In order to fix this issue, we need to use our database client in order to run a `rename` SQL query:
```.sql
mysql -umichael -p  -- password will be prompted
mysql> use addressBook; -- choose the database schema to be used

mysql> RENAME TABLE address to postal_address;
```

Note: as usual, do not forget to backup your database before making any change in production!


## Using Liquibase for database schema change management

We've seen in the previous section that you can run an sql script and use the setting `ddl-auto=validate` when adding a conflicting change to your database (renaming a table, renaming a column, deleting a table etc). 

You can also use a database schema change management tool such as [Liquibase](https://www.liquibase.org/) or [Flyway](https://flywaydb.org/). In this section, we will show how to work with `Liquibase`.

`Liquibase` allows to keep track of database schema changes so you have a version number for each and every schema change that you make in your project lifecycle. 
With `Liquibase`, we can rename `address` into `postal_address` using the below script:

```xml
<databaseChangeLog  ...>
   <changeSet author="Michael" id="1.2">
       <renameTable oldTableName="address" newTableName="postal_address" />
   </changeSet>
</databaseChangeLog>
```
`1.2-changelog-rename-table.xml`

As you can notice:
* Each change has a unique version number (set as `1.2` in our example)
* The change author is tracked manually

`Liquibase` is seamlessly integrated into `Spring Boot`. When using Maven, You just need to declare your `liquibase-core` dependency in your `pom.xml` and Liquibase is automagically used at startup time. 

```xml
<dependency>
   <groupId>org.liquibase</groupId>
   <artifactId>liquibase-core</artifactId>
</dependency>
```
(file pom.xml)

If you need to customise the default configuration (as we do in our example), you can do so inside Spring Boot's configuration file (called `application.properties` in our example):

```.properties
spring.liquibase.change-log=classpath:/db/changelog/1.2-changelog-rename-table.xml
```

When starting our application, we can see in the logs that our changes have been made:
```
liquibase.changelog.ChangeSet            : Table address renamed to postal_address
```

Liquibase also creates a table called `databasechangelog` and tracks all changes inside it:

```
liquibase.executor.jvm.JdbcExecutor      : INSERT INTO addressBook.DATABASECHANGELOG (ID, AUTHOR, FILENAME, ..., `DESCRIPTION`, ...) 
VALUES ('1.2', 'Michael', 'classpath:/db/changelog/1.2-changelog-rename-table.xml', ..., 'renameTable newTableName=postal_address, oldTableName=address', ...)
```

On the long term, we will be able to track all changes happening to our database inside this table.

## Going further with Liquibase

Our blog only shows basic features of `Liquibase`. You can explore Baeldung's excellent blog series [here](https://www.baeldung.com/liquibase-refactor-schema-of-java-app) and [here](https://www.baeldung.com/liquibase-rollback) in order to understand how to work with `Liquibase`'s Maven plugin and rollback procedures.

## Conclusion
We have seen that your database schema changes can be handled:
* Using `Hibernate`'s auto-update feature (non-conflicting changes only)
* By running `sql` queries manually 
* by using a database migration tool such as `Liquibase`. 

If you are working on a simple application, you might be happy with auto-update and manual `sql` queries only. 
If you are making changes to your schema on a regular basis and would like to be able to track your changes, `Liquibase` will be a better option for you.

Thanks for reading our blog and we hope our blog has given you a better understanding of database migrations with `Java` / `Spring Boot` / `Hibernate`.


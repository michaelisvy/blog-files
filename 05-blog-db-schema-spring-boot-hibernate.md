# Database schema changes with Hibernate and Spring Boot

## Target audience
This article has been written for readers who have experience with `Java`, `Hibernate` and `Spring Boot`. All examples use `MySql` but you could also use other relational databases that you are comfortable with.

## Introduction

The Java ecosystem gives you a lot of tools to magically update your database schemas, but are all of these tools reliable enough to be used with a production database? 

In this article - the first in a series - we will focus on industry best practices and `Hibernate`'s auto-schema generation feature. We will explain what we've learned from it and where it is suitable to be used.

In a subsequent article, we will discuss how database schema changes can be made with a database migration tool such as `Liquibase`.
All code samples are available in [our dedicated GitHub repository](https://github.com/michaelisvy/java-db-schema-updates).

## Setup

Let's first create a new database schema called `addressBook` using the `MySql` command-line client:

```
>mysql -u santa -p
Enter password: ******
mysql> CREATE DATABASE addressBook;
Query OK, 1 row affected (0.12 sec)
```

Let's now open our [Java application](https://github.com/michaelisvy/java-db-schema-updates), which uses `Spring Boot` and `MySql`. The configurations for `MySql` can be found inside `application.yml`: 

```.yml
spring:
  jpa:
    database: mysql
    hibernate:
      ddl-auto: update
  datasource:
    url: jdbc:mysql://localhost:3306/addressBook
    username: santa
    password: secret
```

The first 3 lines explain how to connect to `MySql`. Our password is hardcoded for simplicity's sake, but in real life we would store it in a secret.

`ddl-auto: update` shows that our `MySql` schema should be updated at application startup (to be discussed in the next paragraph). 

## Generating a database schema from scratch

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

```.yml
spring.jpa.hibernate.ddl-auto: update
```

Let us run our `JUnit` test suite:

```
mvn clean test
```

In the logs, we can see that the following database query has been run:

```sql
create table user (
id integer not null auto_increment, 
date_of_birth date, 
first_name varchar(255), 
last_name varchar(255), 
primary key (id)) engine=InnoDB
```

How are tests run with `Hibernate`? 

At startup, `Hibernate` parses all classes that have been decorated with the `@Entity` annotation. It then scans the `User` class and generates an `SQL` table creation query.
The table name, column names, types, and etc. are based on the information found in the User class (class name, attribute names and types, annotations, etc.).

### Starting the application one more time

The `addressBook` database schema has been generated and it contains the `User` table. 

What behaviour should we expect when we start the application one more time?

When Hibernate runs the tests again, it compares the class `User` against the table `user`. It then sees that class and table are in sync and it does not make any further changes. 

### Which SQL?

While SQL looks similar when working with various database providers, there is no such thing as [completely interoperable SQL](https://en.wikipedia.org/wiki/SQL#Interoperability_and_standardization).
There are subtle differences in how each SQL handles dates, string concatenation, etc. 
Hibernate elegantly abstracts these differences as "dialects".

Inside our `pom.xml` we have configured the `mysql` jdbc driver as a dependency for MySql 8. `Spring Boot` then assumes that we use the default `MySql 8` dialect and configures `Hibernate` accordingly as shown in the startup logs:
```
HHH000400: Using dialect: org.hibernate.dialect.MySQL8Dialect
```

## Adding a change to an existing database
Let us now add the `Address` entity to our model.

```java
@Data @Entity
public class Address {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String streetAddress;
    private String zipCode;
    private String city;
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

When the application starts (still in `auto-update` mode), Hibernate creates the `address` table as follows:

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

The `address` table and its relationship to `user` have been added as expected. 
While `Hibernate`'s `auto-update` works fine most of the time, it is quite magical and error-prone. From our experience, it is easy to rename a class or a field and to then forget about the fact that a new table or column will be generated the next time the application is deployed. 

In the next section we will discuss about best practices and safeguards when making a change in your production database schema.

## Schema auto-update in production?

In their [official documentation](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#schema-generation), the `Hibernate` team recommends the below:

> Although the automatic schema generation is very useful for testing and prototyping purposes, in a production environment, it’s much more flexible to manage the schema using incremental migration scripts.

Here is the approach that we commonly use:

|  		 | JUnit tests    | Local webapp | Staging webapp  | Production webapp |
| :------------- | :------------- | :----------: | -----------:    | -----------: |
|Database	 |  H2		  | MySql        | MySql           | MySql        |
|Hibernate auto-update setting | create-drop    | update       | validate     | validate     |
|DB backup | none    | none       | mysqldump     | mysqldump     |

* For Unit tests, we use `H2`. The whole database is created in memory at startup time and deleted after all tests have been run (`create-drop`).
* When running a local web application (on `localhost`), we run `update` and copy from the logs all the update scripts that have been generated (such as for the Address table in our example). We will reuse those scripts for our `staging` and `production` environments.
* In `staging` and `production` environments, we use the following setup:
```.properties
spring.jpa.hibernate.ddl-auto=validate
```

At startup time, Hibernate `validate`s that the database schema is compatible with our `JPA/Hibernate` mapping. If any class or attribute is not mapped properly, `Hibernate` throws an exception and the application does not start. 
We try to replicate the behaviour that we will have in production, therefore we update our schema manually using the scripts collected in our local dev environment. 

In `staging` and `production`, we always backup our database and plan for a restore procedure. In MySql, that can be done with the [mysqldump](https://www.thegeekstuff.com/2008/09/backup-and-restore-mysql-database-using-mysqldump/) command.

> Note: You can see that the suggested processes are the same for `staging` and `production` environments. Breaking our application's `staging` environment should not be a big deal. However it is an opportunity to do a dry run before updating our database schema in production.

## Adding a conflicting change
A `conflicting change` is a change that *involves renaming a table or column*.
Let’s imagine, for example, that an `address`, which is being used in our existing schema, is not specific enough, and that we would like to rename the `address` table to `postal_address`. Let's change the name of the `Address` class as follows:

```java
@Data @Entity
public class PostalAddress {
    //...
}
```

Hibernate’s `auto-update` feature does not work well with conflicting changes. If we restart our application in `update` mode, it creates a new table called `postal_address` and still keeps the existing `address` table.

Let's disable auto-schema `update` and use `validate` instead as explained in the previous paragraph: 

```.yml
spring.jpa.hibernate.ddl-auto: validate
```

When starting the application, Hibernate would detect that our classes are not in sync with the database schema and would throw the following exception:

```java
Caused by: org.hibernate.tool.schema.spi.SchemaManagementException: 
Schema-validation: missing table [postal_address]
	at org.hibernate.tool.schema.internal.AbstractSchemaValidator
    .validateTable(AbstractSchemaValidator.java:121)
```

In order to avoid this issue we need to stop the application and launch the below script before the new version of the application is deployed:
```.sql
mysqldump --defaults-file="/var/.../extraparams.cnf"  ... 
>mysql -u santa -p
Enter password: ******
mysql> use addressBook; -- choose the database schema to be used
mysql> RENAME TABLE address to postal_address;
```

We have now made our table name change and deployed the updated version of the application.

The above assumes that we are able to take our application offline for a few minutes. It is extremely hard to make a conflicting change to a database while it's running in production. 

## Conclusion
We have seen that `Hibernate`'s `auto-update` is a great development tool and should *not* be used in staging and production. 
In staging and production, we have seen that you can run your sql queries manually. 
In our follow-up blog (to be published by March 1st 2020), we will discuss how to use `Liquibase` as a database schema migration tool for your staging and production environments.

Thanks for reading our blog!

Michael Isvy.
(thanks to my colleagues Nicolas Guignard, Liew Min Shan and many others on reviewing this article!).


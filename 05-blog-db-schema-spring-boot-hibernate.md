# Database schema changes with Spring Boot and Hibernate

## Let's refactor!!!!

When writing applications, we tend to promote practices such as TDD that allow us to be confident when refactoring our code. 
What about our database? Should we be able to refactor our production database schema in a jiffy, just as easily as we would change a method name? May be not, but let's try to get there!

## Setup

Here we will discuss the various ways to update your database schema in production for a `Spring Boot` / `JPA` / `Hibernate` application in Java.
All code samples are available in [this repository](https://github.com/michaelisvy/java-db-schema-updates).

To start with, we have a `Spring Boot` application that uses `H2` for `JUnit` tests and connects to `MySql` as its staging/production database. 

```.properties
spring.datasource.url=jdbc:mysql://localhost:3306/addressBook
spring.datasource.username=santa
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
```

The above example shows our `application-mysql.properties` file. 
The first 3 lines explain how to connect to MySql. Our password is hardcoded for simplicity sake, but in real life we would store it in a secret.
The last line shows that the MySql schema should be updated at application startup (to be discussed in the next section). 


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

`DDL` stands for [Data Definition Language](https://en.wikipedia.org/wiki/Data_definition_language).

At application startup, the following query is run:

```sql
create table user (
id integer not null auto_increment, 
date_of_birth date, 
first_name varchar(255), 
last_name varchar(255), 
primary key (id)) engine=InnoDB
```

How does it work? At startup time, Hibernate parses the `User` class and generates a table based on the information it found (class name, attributes, annotations…).
A `create table` query is generated based on your database's `dialect`.

Our application does not specify any dialect in its configuration because it relies on the default one. However, we can see the dialect used in the application startup logs:
```
HHH000400: Using dialect: org.hibernate.dialect.MySQL8Dialect
```

## Adding non conflicting changes to a database
Add address table.
Works well. Don’t forget to backup your database before making changes.
Note: use of ForeignKey annotation.

Show SQL generated (create table and alter table for constraint)

No matter which solution you prefer to use: you should always backup your database and plan for a rollback procedure before changing your database schema. It’s also good to practice your DB update and rollback procedure on a staging environment before making a production change.

## Adding a conflicting change using Hibernate’s auto schema generation
Let’s now consider that we would like to rename the address table into postal_address.

Doing such a change using Hibernate’s auto schema generation would create the following behaviour:
Hibernate will ignore the existing table (address) and will create a new one called postal_address
All data inside address will stay with address. In the future, all new data will be created inside (postal_address)

If you have an existing production database, the above behaviour is probably not what you’re after. 

We now need to tweak our configuration so we still have auto-generation  for h2 (database used for JUnit Tests) and not for mysql (our staging / production database).

H2:
spring.jpa.hibernate.ddl-auto=update

Mysql:
spring.jpa.hibernate.ddl-auto=validate


Before updating your application, you just need to rename your table as follows:
RENAME TABLE address2 to address3


Note: the above implies that you are able to take your application offline for a few minutes (so the database can be updated and the application redeployed). 


## Using Liquibase to rename a table
 ALTER TABLE addressBook.address4 RENAME addressBook.address5

Liquibase allows to keep track of database schema changes so you have a version number for each and every schema change that you make in your project lifecycle. 
With Liquibase, renaming address into postal_address would work like this:

<databaseChangeLog  ...>
   <changeSet author="John" id="1.2">
       <renameTable oldTableName="address" newTableName="postal_address" />
   </changeSet>
</databaseChangeLog>

Each change has a unique version number (referred to as `id` in the above example)

Database changelog table:

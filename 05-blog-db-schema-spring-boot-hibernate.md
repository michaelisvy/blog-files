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

We are also adding a relationship from User to Address as follows:

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

> Note: Never forget to backup your database and plan for a restore procedure before making a change in production. In MySql, that can be done with the [mysqldump](https://www.thegeekstuff.com/2008/09/backup-and-restore-mysql-database-using-mysqldump/) command.

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

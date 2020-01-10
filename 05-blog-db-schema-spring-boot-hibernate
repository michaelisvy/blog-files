# Database schema changes with Spring Boot and Hibernate

## let's refactor!!!!

When writing applications, we tend to promote practices such as TDD that allow us to be confident when refactoring our applications. 
What about our database? Should we be able to refactor our production database schema in a jiffy, just as easily as we would change a method name? May be not, but let's try to get there!

Here we will discuss the various ways to update your database schema in production for a `Spring Boot` / `JPA` / `Hibernate` application in Java.

Show the initial setup: both h2 and mysql use spring.jpa.hibernate.ddl-auto=update


No matter which solution you prefer to use: you should always backup your database and plan for a rollback procedure before changing your database schema. It’s also good to practice your DB update and rollback procedure on a staging environment before making a production change.


Creating a Database schema using Hibernate’s auto schema generation
Show application.properties and User class
Note: DDL stands for Data Definition Language (https://en.wikipedia.org/wiki/Data_definition_language)
Show create table happening at startup time
How does it work? The User class has been read and its information (class name, attributes, relationships…) have been used to generate a simple database schema.

Adding non conflicting changes to a database
Add address table.
Works well. Don’t forget to backup your database before making changes.
Note: use of ForeignKey annotation.

Show SQL generated (create table and alter table for constraint)

Adding a conflicting change using Hibernate’s auto schema generation
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


Using Liquibase to rename a table
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

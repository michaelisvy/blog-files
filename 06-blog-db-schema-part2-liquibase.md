# Database schema changes with Liquibase

## Target audience
This article has been written for people who have at least basic understanding of the Java/Spring/Spring Boot/Hibernate ecosystem. 

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

# Testing a Java backend application

## Target audience
This blog is for people who have a general understanding of the Java / Spring Boot ecosystem. As test frameworks of choice, we will be using JUnit, Mockito and assertJ. For all automated tests, we use an in-memory database (H2).

## Introduction
All code samples below are based on our experience at OnlinePajak. Please note that we do not have so-called `monster applications` nor `monster databsases`. Our back-end has been well-divided into mid-size applications and each of them has their own database.
If we had had to work with a gigantic legacy database, our approach might have been different.

## The Repository and Service layers
* Using Spring Data, the repository layer is super-thin. In many cases, it is just an interface. 
We therefore do not test it directly and prefer to test the service layer.

Our approach:
Test -> Service -> Repository -> H2 

Note: Spring Data is flexible enough that you can use it for SQL databases (MySql, Postgres, H2...) as well as NoSql databases such as MongoDB. Not all NoSql database allow you to run your database in-memory for testing purpose. This can (be done with MongoDB)[https://dzone.com/articles/spring-boot-with-embedded-mongodb]. 

## The Controller layer

## Conclusion



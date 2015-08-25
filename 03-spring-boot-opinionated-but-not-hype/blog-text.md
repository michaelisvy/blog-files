# Spring Boot: opinionated but not hype

It is summertime and we are taking it as an opportunity to add more Spring Boot coverage as part of our Core-Spring training course.
I took this opportunity to have lots of conversations about Spring Boot with our community.

## Embedded container... or not

Lots of people tend to think that you have to go containerless if you'd like to use Spring Boot.
You can actually work in 2 ways as shown in the diagram below:


1) The application is packaged as a war file so it can be run inside an external application server just like any Java/Spring application

```Maven config


2) The application embeds the web server (Tomcat, Jetty or UnderTow) and can be run in the command line

```xml
Maven config
```

```
how to deploy
```

There are pros and cons for each solution [as explained here](https://www.reddit.com/r/java/comments/36nt73/war_vs_containerless_spring_boot_etc/).



## XML or Java Config
Once upon a time (somewhere around 2005), the world was simple: Spring only proposed XML and beginners didn't have to choose between 3 ways of doing dependency injection (XML, annotations, Java Config).

10 years later, we have lots of options. Spring Boot is typically used in 2 ways:

```Diagram for annotations + Java Config or annotations



# Component scanning: all you can eat!



# properties or YAML
Completely your choice
show one way and the other
one thing you only can do with YAML files (profiles)

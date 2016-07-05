---
layout: post
title: Contributing to Apache Camel with new components
---

Writing a new Camel component is simple, but sometimes it's not so easy for a beginner to understand how it can be integrated in current Camel codebase.

With this post I'd like to show you how you can do this step-by-step.

### A simple component

In this guide I will add a simple component: camel-calculator. This is just a simple example, just to show the steps needed for integrating and contributing your component to Apache Camel codebase.

### Using the Archetype

The latest Camel release is 2.17.2 at the moment of writing, so we will use the archetype from this version.

Be sure you've cloned the (Apache Camel repository)[https://github.com/apache/camel] from github and the codebase is aligned with the upstream codebase.

Open a terminal and enter into the components folder

```shell
~/workspace/apache-camel/camel/components$ 
```

Generate the skeleton for the camel-calculator component

```shell
~/workspace/apache-camel/camel/components/camel-calculator$ mvn archetype:generate -DarchetypeGroupId=org.apache.camel.archetypes -DarchetypeArtifactId=camel-archetype-component -DarchetypeVersion=2.17.2  -DgroupId=org.apache.camel -DartifactId=camel-calculator -Dname=Calculator -Dscheme=calculator
```

You will be asked for the version of the component and other things in interactive way. Since the camel version currently in development is 2.18, use 2.18-SNAPSHOT as version of your component.

### Cleaning up the component

First thing to do is cleaning the pom of the newly generated component a bit. So:

- Change the packaging from bundle to jar.
- Remove all the version tags from dependencies
- In the properties section of POM remove everything and add the following properties:

```xml
   <properties>
      <camel.osgi.export.pkg>org.apache.camel.component.calculator.*</camel.osgi.export.pkg>
      <camel.osgi.export.service>org.apache.camel.spi.ComponentResolver;component=calculator</camel.osgi.export.service>
   </properties>
```

- Add the tag 

```xml
  <name>Camel :: Calculator</name>
```

- Since the GroupId is inherited from the parent POM of components folder, you can remove the groupId tag from your component POM.
- Remove camel-apt from the dependencies

Finally your POM should be like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

  <modelVersion>4.0.0</modelVersion>

  <parent>
    <artifactId>components</artifactId>
    <groupId>org.apache.camel</groupId>
    <version>2.18-SNAPSHOT</version>
  </parent>

  <artifactId>camel-calculator</artifactId>
  <packaging>jar</packaging>
  <name>Camel :: Calculator</name>

  <properties>
      <camel.osgi.export.pkg>org.apache.camel.component.calculator.*</camel.osgi.export.pkg>
      <camel.osgi.export.service>org.apache.camel.spi.ComponentResolver;component=calculator</camel.osgi.export.service>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.apache.camel</groupId>
      <artifactId>camel-core</artifactId>
    </dependency>

    <!-- logging -->
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <scope>test</scope>
    </dependency>

    <!-- testing -->
    <dependency>
      <groupId>org.apache.camel</groupId>
      <artifactId>camel-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

</project>
```

You now need to create the right packaging. So move the generated classes from src/main/java/org/apache/camel to src/main/java/org/apache/camel/component/calculator 
and do the same for the test package.




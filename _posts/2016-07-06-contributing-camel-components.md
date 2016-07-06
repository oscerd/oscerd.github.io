---
layout: post
title: Contributing new components to Apache Camel project
---

Writing a new Camel component is simple, but sometimes it's not so easy for a beginner to understand how it can be integrated in current Camel codebase.

With this post I'd like to show you how you can do this step-by-step.

### A simple component

In this guide I will add a simple component: camel-square. This is just a simple example, just to show the steps needed for integrating and contributing your component to Apache Camel codebase.

### Using the Archetype

The latest Camel release is 2.17.2 at the moment of writing, so we will use the archetype from this version.

Be sure you've cloned the [Apache Camel repository](https://github.com/apache/camel) from github and the codebase is aligned with the upstream codebase.

Open a terminal and enter into the components folder

```
~/workspace/apache-camel/camel/components$ 
```

Generate the skeleton for the camel-square component

```
~/workspace/apache-camel/camel/components$ mvn archetype:generate -DarchetypeGroupId=org.apache.camel.archetypes -DarchetypeArtifactId=camel-archetype-component -DarchetypeVersion=2.17.2  -DgroupId=org.apache.camel -DartifactId=camel-square -Dname=Square -Dscheme=square
```

You will be asked for the version of the component and other things in interactive way. Since the camel version currently in development is 2.18, use 2.18-SNAPSHOT as version of your component.

### Cleaning up the component

First thing to do is cleaning the pom of the newly generated component a bit. So:

- Change the packaging from bundle to jar.
- Remove all the version tags from dependencies
- In the properties section of POM remove everything and add the following properties:

```xml
   <properties>
      <camel.osgi.export.pkg>org.apache.camel.component.square.*</camel.osgi.export.pkg>
      <camel.osgi.export.service>org.apache.camel.spi.ComponentResolver;component=square</camel.osgi.export.service>
   </properties>
```

- Add the tag 

```xml
  <name>Camel :: Square</name>
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

  <artifactId>camel-square</artifactId>
  <packaging>jar</packaging>
  <name>Camel :: Square</name>

   <properties>
      <camel.osgi.export.pkg>org.apache.camel.component.square.*</camel.osgi.export.pkg>
      <camel.osgi.export.service>org.apache.camel.spi.ComponentResolver;component=square</camel.osgi.export.service>
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

You now need to create the right packaging. So move the generated classes from src/main/java/org/apache/camel to src/main/java/org/apache/camel/component/square 
and do the same for the test package. Don't forget to align also the file `src/main/resources/META-INF/services/org/apache/camel/component/square` to point to the component class 
`class=org.apache.camel.component.square.SquareComponent`

We are now ready to write our component.

### Writing the component

This is a simple example: the component won't have a lot of features and we will use only a Producer Endpoint. You can remove the class SquareConsumer then.

Your SquareComponent may look like this:

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.camel.component.square;

import java.util.Map;

import org.apache.camel.CamelContext;
import org.apache.camel.Endpoint;

import org.apache.camel.impl.UriEndpointComponent;

/**
 * Represents the component that manages {@link SquareEndpoint}.
 */
public class SquareComponent extends UriEndpointComponent {
    
    public SquareComponent() {
        super(SquareEndpoint.class);
    }

    public SquareComponent(CamelContext context) {
        super(context, SquareEndpoint.class);
    }

    protected Endpoint createEndpoint(String uri, String remaining, Map<String, Object> parameters) throws Exception {
        Endpoint endpoint = new SquareEndpoint(uri, this);
        setProperties(endpoint, parameters);
        return endpoint;
    }
}
```

and your SquareEndpoint class like this:

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.camel.component.square;

import org.apache.camel.Consumer;
import org.apache.camel.Processor;
import org.apache.camel.Producer;
import org.apache.camel.impl.DefaultEndpoint;
import org.apache.camel.spi.Metadata;
import org.apache.camel.spi.UriEndpoint;
import org.apache.camel.spi.UriPath;

/**
 * Represents a Square endpoint.
 */
@UriEndpoint(scheme = "square", title = "Square", syntax="square:name", label = "Square")
public class SquareEndpoint extends DefaultEndpoint {
    @UriPath @Metadata(required = "true")
    private String name;

    public SquareEndpoint() {
    }

    public SquareEndpoint(String uri, SquareComponent component) {
        super(uri, component);
    }

    public Producer createProducer() throws Exception {
        return new SquareProducer(this);
    }

    public Consumer createConsumer(Processor processor) throws Exception {
    	throw new UnsupportedOperationException("The Square endpoint doesn't support consumers.");
    }

    public boolean isSingleton() {
        return true;
    }

    /**
     * Some description of this option, and what it does
     */
    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

}
```

Since the component won't support Consumer Endpoints, we throw an UnsupportedOperationException in that particular case. 
Let's write our SquareProducer. The Producer Endpoint will get the body of the message and it will calculate the square of the body.
At the beginning we will have a producer of this form:

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.camel.component.square;

import org.apache.camel.Exchange;
import org.apache.camel.impl.DefaultProducer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * The Square producer.
 */
public class SquareProducer extends DefaultProducer {
    private static final Logger LOG = LoggerFactory.getLogger(SquareProducer.class);
    private SquareEndpoint endpoint;

    public SquareProducer(SquareEndpoint endpoint) {
        super(endpoint);
        this.endpoint = endpoint;
    }

    public void process(Exchange exchange) throws Exception {
        System.out.println(exchange.getIn().getBody());    
    }

}
```

The Math.pow() method is slow. For this example we will simply multiply the number by itself.
Once again: this is just an example to show how you can contribute your component to the Apache Camel community, so there won't be validation on the Body type and so on.
We may think about a Producer like this:

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.camel.component.square;

import org.apache.camel.Exchange;
import org.apache.camel.Message;
import org.apache.camel.impl.DefaultProducer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * The Square producer.
 */
public class SquareProducer extends DefaultProducer {
    private static final Logger LOG = LoggerFactory.getLogger(SquareProducer.class);
    private SquareEndpoint endpoint;

    public SquareProducer(SquareEndpoint endpoint) {
        super(endpoint);
        this.endpoint = endpoint;
    }

    public void process(Exchange exchange) throws Exception {
    	LOG.debug("Getting value from exchange");
    	Integer value = exchange.getIn().getBody(Integer.class);
    	LOG.debug("Computing square");
    	Integer square = value * value;
    	LOG.info("The square is " + square);
        if (exchange.getPattern().isOutCapable()) {
            Message out = exchange.getOut();
            out.copyFrom(exchange.getIn());
            out.setBody(square);
        } else {
            Message in = exchange.getIn();
            in.setBody(square);
        }
        
    }

}
```

It's very simple. It just compute the square and, based on the Exchange Pattern, put the computation result in the in/out message.
Let's take a look at the testing part. You should have a test class SquareComponentTest: let's modify it a bit. The result can be:

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.camel.component.square;

import org.apache.camel.builder.RouteBuilder;
import org.apache.camel.component.mock.MockEndpoint;
import org.apache.camel.test.junit4.CamelTestSupport;
import org.junit.Test;

public class SquareComponentTest extends CamelTestSupport {

    @Test
    public void testSquare() throws Exception {
        MockEndpoint mock = getMockEndpoint("mock:result");
        mock.expectedMinimumMessageCount(1);
        mock.expectedBodiesReceived(9);
        
        template.sendBody("direct:square", 3);
        
        assertMockEndpointsSatisfied();
    }

    @Override
    protected RouteBuilder createRouteBuilder() throws Exception {
        return new RouteBuilder() {
            public void configure() {
                from("direct:square")
                  .to("square://bar")
                  .to("mock:result");
            }
        };
    }
}
```

Now the component is ready for the last little things to do. 

Try to install it:

```
~/workspace/apache-camel/camel/components/camel-square$ mvn clean install
```

Check for code-style errors and eventually fix it

```
~/workspace/apache-camel/camel/components/camel-square$ mvn -Psourcecheck
```

### Integrating the component in the Apache Camel codebase

The component is now ready to be integrated in the current Camel codebase.

We need to add a dependency in `apache-camel/pom.xml`

```xml
     <dependency>
        <groupId>org.apache.camel</groupId>
        <artifactId>camel-square</artifactId>
     </dependency>
```

Include the component in `apache-camel/src/main/descriptors/common-bin.xml`

```xml
     <include>org.apache.camel:camel-square</include>
```

Include the component in `parent/pom.xml`

```xml
       <dependency>
          <groupId>org.apache.camel</groupId>
          <artifactId>camel-square</artifactId>
          <version>${project.version}</version>
       </dependency>
```

Now you're able to build your new Camel 2.18-SNAPSHOT with camel-square included:

```
~/workspace/apache-camel/camel/$ mvn clean install -DskipTests
```

At the end of the build you should see the camel-square component listed.

### Component documentation

From Camel 2.18-SNAPSHOT we will have documentation generated from code inside our codebase.

The default directory inside the new component folder is `src/main/docs` and the documentation file in is an .adoc file with the same name of the component, in this case it will be `square.adoc`

Create the `src/main/docs/square.adoc` file with content and the following placeholders

```
// component options: START
// component options: END

// endpoint options: START
// endpoint options: END
```

The placeholders are just for the automatic documentation generation.

Run a clean install on the component once again

```
~/workspace/apache-camel/camel/components/camel-square$ mvn clean install 
```

Your .adoc file will contain the documentation updated for component and endpoint.

Now add a link to your component in the file `docs/user-manual/en/SUMMARY.md`. This way the component will be added to the Gitbook generated from the Asciidoc component files.

### Integrating with Apache Karaf

Apache Karaf is an important ally for Apache Camel. 
Usually a Camel component should work also in an OSGi environment. For each component in Camel there is a Karaf feature definition. Let's add the one for camel-square. 
In `platform/karaf/features/src/main/resources/features.xml` add the following code

```xml
  <feature name='camel-square' version='${project.version}' resolver='(obr)' start-level='50'>
    <feature version='${project.version}'>camel-core</feature>
    <bundle>mvn:org.apache.camel/camel-square/${project.version}</bundle>
  </feature>
```

Since the component is very simple we don't need external bundle in this case. To test if our feature work in an OSGi enviroment we have to add an integration test.
Let's create it in `tests/camel-itest-karaf` with the name CamelSquareTest. The content of the test will be:

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.camel.itest.karaf;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.ops4j.pax.exam.junit.PaxExam;

@RunWith(PaxExam.class)
public class CamelSquareTest extends BaseKarafTest {

    public static final String COMPONENT = extractName(CamelSquareTest.class);

    @Test
    public void test() throws Exception {
        testComponent(COMPONENT);
    }
}
```

To run the test we have to follow two steps. First, we need to build our Karaf features

```
~/workspace/apache-camel/camel/platform/karaf/features$ mvn clean install 
```

and second run the integration test

```
~/workspace/apache-camel/camel/tests/camel-itest-karaf$ mvn clean test -Dtest=CamelSquareTest
```

or 

```
~/workspace/apache-camel/camel/tests/camel-itest-karaf$ run-tests CamelSquareTest
```

If everything is fine and test passes the component will install in Karaf.

### Contributions

The camel-square component is now part of Camel. What is still missing? The Pull Request off course :-) 

This surely is the most satisfactory part :-)

You should have everything committed locally and maybe you need to align to the current Apache Camel codebase.

```
~/workspace/apache-camel/camel$ git pull --rebase <remote_name> master
```

If you have multiple commit for your component, squash them in a single one it's a good idea.

At this point you just need to open the Pull Request and the Apache Camel team will review it for you. You'll receive feedback about improvements you can do and, off course, thanks from the community :-)

### Conclusions

In this post I've shown how a Camel component can be added to the Apache Camel codebase step-by-step.
I think writing components is one of the best things to do to understand Camel architecture and features and to improve your Camel skill.
Camel community love [contributions](http://camel.apache.org/contributing.html), you just need to start :-)

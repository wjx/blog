---
title: Another reason for mybatis "Invalid bound statement (not found)"
date: 2021-07-11 22:19:09
description: Use classpath* in mapperLocations to make sure mybatis mapper xml files in submodules be picked up by Spring.
---

Mybatis users often encounter "Invalid bound statement (not found)" error reporting when invoking a mapped method. The most common reasons being:

* The namespace specified in mapper xml file doesn't match the corresponding java mapper interface name.
* The method id specified in mapper xml file doesn't match the method name in java mapper interface.
* The mapper xml file is not packaged into the targe jar package, which can be solved by adding the following config into pom.xml:

```xml
    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>*.yml</include>
                    <include>*.xml</include>
                    <include>sqlmap/*.xml</include>
                    <include>META-INF/spring.factories</include>
                </includes>
            </resource>
        </resources>
    </build>
```

This week I encountered this error reporting again, and after some digging I found it's caused by the misconfiguration of `mapperLocations`.  

My project consists of some maven modules each compiled into a jar file. All those compiled jar files would be assembled into an executable jar file. Each submodule has it's own resource path and mybatis xml files. And my config was:

```yaml
mybatis:
  mapperLocations: classpath:sqlmap/*.xml
```

It turns out with is configuration, those mybatis mapper xml files in submodule jars will not be picked up by Spring, hence mybatis warning mapper method not bound. The solution is to change the `classpath` to `classpath*`:

```yaml
mybatis:
  mapperLocations: classpath*:sqlmap/*.xml
```

Here's [Spring document](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#resources-app-ctx-wildcards-in-resource-paths):
> One use for this mechanism is when you need to do component-style application assembly. All components can publish context definition fragments to a well-known location path, and, when the final application context is created using the same path prefixed with classpath*:, all component fragments are automatically picked up.

> This special prefix specifies that all classpath resources that match the given name must be obtained (internally, this essentially happens through a call to ClassLoader.getResources(…​)) and then merged to form the final application context definition.



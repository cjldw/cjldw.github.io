---
title: maven打包jar
date: 2016-11-26 01:28:06
desc:
tags:
---

maven项目如何将编写好的java文件打包成可执行的jar包呢?

<!-- more -->

### 使用 maven-jar-plugin 和 maven-jar-plugin 插件

0. 创建maven项目, 可以使用eclipse或idea或mvn命令

    ```bash
        mvn archetype:generate -DgroupId=com.luowen.app -DartifactId=mvn-app -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
    ```

1. 项目目录结构, 可以随便写点java文件

    ```java
        |   +---main
        |   +---java
        |   |   \---com
        |   |       \---luowen
        |   |           \---app
        |   |               |   App.java
        |   |               |
        |   |               +---http
        |   |               |       HttpGetDemo.java
        |   |               |
        |   |               +---io
        |   |               |       FileReaderDemo.java
        |   |               |
        |   |               \---log4jtest
        |   |                       Log4jConfigureDemo.java
        |   |
        |   \---resources
        |           log4j.properties
        |
        \---test
            \---java
                \---com
                    \---luowen
                        \---app
                            |   AppTest.java
                            |
                            \---io
                                    FileReaderDemoTest.java
    ```

2. pom.xml配置文件

    ```html
        <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
            <modelVersion>4.0.0</modelVersion>
            <groupId>com.luowen.app</groupId>
            <artifactId>mvn-app</artifactId>
            <packaging>jar</packaging>
            <version>1.0-SNAPSHOT</version>
            <name>mvn-app</name>
            <url>http://maven.apache.org</url>

            <properties>
                <jdk.version>1.8</jdk.version>
                <junit.version>4.12</junit.version>
            </properties>

            <dependencies>
                <!-- https://mvnrepository.com/artifact/junit/junit -->
                <dependency>
                    <groupId>junit</groupId>
                    <artifactId>junit</artifactId>
                    <version>4.12</version>
                    <scope>test</scope>
                </dependency>

                <!-- https://mvnrepository.com/artifact/log4j/log4j -->
                <dependency>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                    <version>1.2.17</version>
                    <scope>compile</scope>
                </dependency>

                <!-- https://mvnrepository.com/artifact/org.apache.httpcomponents/httpclient -->
                <dependency>
                    <groupId>org.apache.httpcomponents</groupId>
                    <artifactId>httpclient</artifactId>
                    <version>4.5.2</version>
                    <scope>compile</scope>
                </dependency>
            </dependencies>
            <build>
                <resources>
                    <resource>
                        <directory>src/main/resources</directory>
                        <filtering>true</filtering>
                    </resource>
                </resources>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-compiler-plugin</artifactId>
                        <version>2.3.2</version>
                        <configuration>
                            <source>${jdk.version}</source>
                            <target>${jdk.version}</target>
                        </configuration>
                    </plugin>

                    <plugin>
                        <!-- maven jar plugin 配置部分
                            主要将编写的java文件编译打包 -->
                        <artifactId>maven-jar-plugin</artifactId>
                        <configuration>
                            <archive>
                                <manifest>
                                    <addClasspath>true</addClasspath>
                                    <mainClass>com.luowen.app.App</mainClass>
                                    <!-- 制定依赖包的存放目录, 后期应用打包好的jar包, 该目录下的所有*.jar都需要拷贝使用 -->
                                    <classpathPrefix>dependencies-jars</classpathPrefix>
                                </manifest>
                            </archive>
                        </configuration>
                    </plugin>
                    <plugin>
                        <!-- 复制依赖包到制定输入目录 -->
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-dependency-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>copy-dependencies</id>
                                <phase>package</phase>
                                <goals>
                                    <goal>copy-dependencies</goal>
                                </goals>
                                <configuration>
                                    <includeScope>runtime</includeScope>
                                    <outputDirectory>${project.build.directory}/dependencies-jars/</outputDirectory>
                                </configuration>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>

        </project>
    ```

### 使用 maven-shade-plugin 一步到位

1. pom.xml配置文件

    ```html
        <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
            <modelVersion>4.0.0</modelVersion>
            <groupId>com.luowen.app</groupId>
            <artifactId>mvn-app</artifactId>
            <packaging>jar</packaging>
            <version>1.0-SNAPSHOT</version>
            <name>mvn-app</name>
            <url>http://maven.apache.org</url>

            <properties>
                <jdk.version>1.8</jdk.version>
                <junit.version>4.12</junit.version>
            </properties>

            <dependencies>
                <!-- https://mvnrepository.com/artifact/junit/junit -->
                <dependency>
                    <groupId>junit</groupId>
                    <artifactId>junit</artifactId>
                    <version>4.12</version>
                    <scope>test</scope>
                </dependency>

                <!-- https://mvnrepository.com/artifact/log4j/log4j -->
                <dependency>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                    <version>1.2.17</version>
                    <scope>compile</scope>
                </dependency>

                <!-- https://mvnrepository.com/artifact/org.apache.httpcomponents/httpclient -->
                <dependency>
                    <groupId>org.apache.httpcomponents</groupId>
                    <artifactId>httpclient</artifactId>
                    <version>4.5.2</version>
                    <scope>compile</scope>
                </dependency>

            </dependencies>



            <build>
                <resources>
                    <resource>
                        <directory>src/main/resources</directory>
                        <filtering>true</filtering>
                    </resource>
                </resources>
                <plugins>

                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-compiler-plugin</artifactId>
                        <version>2.3.2</version>
                        <configuration>
                            <source>${jdk.version}</source>
                            <target>${jdk.version}</target>
                        </configuration>
                    </plugin>
                    <plugin>
                        <!-- 使用shade包一步到位, 他会将所有依赖的class文件都打包在一起, 推荐使用-->
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-shade-plugin</artifactId>
                        <version>2.1</version>
                        <executions>
                            <execution>
                                <phase>package</phase>
                                <goals>
                                    <goal>shade</goal>
                                </goals>
                                <configuration>
                                    <transformers>
                                        <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                            <mainClass>com.luowen.app.App</mainClass>
                                        </transformer>
                                    </transformers>
                                </configuration>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </project>
    ```

2. 执行命令打包

    ```bash
        mvn package
    ```

###  推荐使用第二种方式, 简单高效, 不依赖其他jar

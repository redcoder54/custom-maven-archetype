# 自定义Archetype

Archetype可以看作是maven项目的模版，通过archetype可以快速生成一个项目骨架，例如常用的maven-archetype-quickstart和maven-archetype-webapp是maven官方提供的模版，使用archetype创建项目时，只需要提供一些基本的信息（比如groupId、artifactId、version等），它就能生成项目的基本结构和pom文件。

下面我们来介绍一下如何自定义创建一个archetype。

## 编写Archetype

一个典型的archetype maven项目主要包括以下几个部分：

- pom.xml: Archetype自身的pom
- src/main/resources/archetype-resources: 基于该Archtype生成的项目的pom原型

- src/main/resources/META-INF/maven/archetype-metadata.xml: Archetype的描述符文件
- src/main/resources/archetype-resources/**: 其它需要包含在archetype中的内容

编写Archetype项目时，首先要定义好其要包含的目录和文件，比如下面是本archetype项目的目录结构。

![myarchetype.png](https://i.loli.net/2019/09/01/t9zEXUwnQMhZPLR.png)

src/main/resources/archetype-resources/pom.xml是该archetype项目生成的pom原型，在这里定义好项目的基本配置，项目生成时，这些配置就是现成的了。注意groupId、artifactId、version和name等属性并没有直接声明，而是采用了属性声明的方式，如下所示

```java
<groupId>${groupId}</groupId>
<artifactId>${artifactId}</artifactId>
<version>${version}</version>
<name>${artifactId}</name>
```

使用archetype生成项目时，用户一般会提供这些信息，项目生成后，这些属性会被填充。

一个archetype项目的核心部分是archetype-metadata.xml描述符文件，它位于archetype项目的资源目录下的META-INF/maven目录下。它主要做两件事：

> 1. 声明哪些目录和文件应包含在archetype中
> 2. 这个archetype使用哪些属性参数

下面是本archetype项目的archetype-metadata.xml的内容

```java
<?xml version="1.0" encoding="UTF-8"?>
<arche-type-descriptor name="myarchetype">
    <fileSets>
        <fileSet filtered="true" packaged="true">
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.java</include>
            </includes>
        </fileSet>
        <fileSet filtered="true" packaged="true">
            <directory>src/test/java</directory>
            <includes>
                <include>**/*.java</include>
            </includes>
        </fileSet>
        <fileSet filtered="true" packaged="false">
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.properties</include>
            </includes>
        </fileSet>
    </fileSets>
    <requiredProperties>
        <requiredProperty key="port"/>
        <requiredProperty key="groupId">
            <defaultValue>wyn</defaultValue>
        </requiredProperty>
    </requiredProperties>
</arche-type-descriptor>
```

它主要包含fileSets和requiredPropeties两个部分，其中fileSets包含多个fileSet子元素，每个fileSet定义一个目录以及该目录的包含和排除规则。

上述代码片段中第一个fileSet指向的目录是src/main/java，该目录对应于archetype项目的资源目录的archetype-resources/src/main/java子目录。fileSet元素有两个属性，其中filtered表示是否对该文件集合应用属性替换。例如像${x}这样的值是否替换为命令输入的x参数的值；packaged表示是否在该目录下创建包路径。

fileSet包含了includes元素，其中声明了一个值为  __\*\*/\*.java__  的规则，表示包含 src/main/java 中任意路径下的java文件。处理includes,你也可以使用excludes元素声明排除规则。

默认情况下，maven-archetype-plugin插件要求用户在使用archetype生成项目时，必须提供 groupId、artifactId、version和package。除此之外，用户在编写自定义archetype时，可以要求提供额外的参数。例如上述代码片段中，使用 requiredProperties 配置要求提供**port**参数，并且为groupId提供了一个默认值**wyn**。这样，archetype中所有开启filtered的文件中就可以使用 **${port}** 属性声明，然后项目生成时使用命令行输入的值填充。

假设使用该archetype项目生成项目，用户输入的groupId、artifactId和version分别为：cn.wxy、demo、1.0.0-SNAPSHOT，下表表示了该archetype中的文件与生成项目的文件对应关系

| Archetype资源目录下                    | 生成的项目根目录下               |
| -------------------------------------- | -------------------------------- |
| archetype-resources/src/main/java      | /src/main/java                   |
| App.java                               | cn.wxy.demo.App.java             |
| dao/Dao.java                           | cn.wxy.demo.dao.Dao.java         |
| service/Service.java                   | cn.wxy.demo.service.Service.java |
| archetype-resources/src/main/resources | /src/main/resources              |
| app.properties                         | app.properties                   |
| arche-resources/src/test               | /src/test                        |

项目生成后，打开App.java、Dao.java、Service.java，可以看到 **${package}** 被替换为cn.wxy.demo, **${port}** 被替换为用户输入的值。

最后，不要忘了将我们自定义的archetype项目安装到本地仓库中，执行`mvn clean install`将其安装到本地仓库。

如果你有自己的远程仓库，可以执行`mvn depoly`将自定义的archetype部署到远程仓库。

## 使用自定义的archetype创建项目

将自定义的archetype项目安装到本地仓库后，我们便可以使用该archetype创建项目了。

在命令行下执行下述命令，

```shell
mvn archetype:generate -DarchetypeGroupId=wyn -DarchetypeArtifactId=myarchetype -DarchetypeVersion=1.0
```

然后输入artifactId、version、port，以交互的方式创建项目。

![create.png](https://i.loli.net/2019/09/01/1oF7hHMtLAiUZmX.png)

## 生成Archetype Catalog

使用archetype创建项目时，一般不需要精确指定archetype的坐标信息，maven-archtype-plugin会提供一个archetype列表供用户选择。该列表的信息来源一个叫archetype-catalog.xml的文件。

那么能否把我们自定义的archetype加入到这个列表中呢？答案是肯定的。

maven-archetype-plugin提供了一个crawl的goal，用户可以使用它来遍历本地maven仓库，生成一个archetype-catalog.xml文件（该文件位于本地仓库的根目录下，默认是~/.m2/repository目录，其中**~**表示用户目录）。

进入当前archetype项目的根目录下，执行`mvn archetype:crawl`命令，在本地仓库的根目录下生成archetype-catalog.xml文件。

然后，我们在执行`mvn archetype:generator`命令生成的新项目时，就可以看到我们自定义的archetype。

![archetype_generate.png](https://i.loli.net/2019/09/01/NTtSCZlJyj3b4rE.png)
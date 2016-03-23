# Maven
参考资料：
- http://ifeve.com/maven-1/
- http://ifeve.com/maven-2/
- 基本标签内容说明: http://www.cnblogs.com/zz0412/p/Maven_pom.html 

### Maven概览-核心概念
Maven的中心思想是POM文件（项目对象模型）。POM文件是以XML文件的形式表述项目的资源，如源码、测试代码、依赖（用到的外部Jar包）等。POM文件应该位于项目的根目录下。
下图说明了Maven是如何使用POM文件的，以及POM文件的主要组成部分：
![maven概览](http://ifeve.com/wp-content/uploads/2014/06/maven-overview-1.png)

- POM文件

当你执行一条Maven命令的时候，你会传入一个pom文件。Maven会在该pom文件描述的资源上执行该命令。

- 构建生命周期、阶段和目标

Maven的构建过程被分解为构建生命周期、阶段和目标。一个构建周期由一系列的构建阶段组成，每一个构建阶段由一系列的目标组成。当你运行Maven的时候，你会传入一条命令。这条命令就是构建生命周期、阶段或目标的名字。如果执行一个生命周期，该生命周期内的所有构建阶段都会被执行。如果执行一个构建阶段，在预定义的构建阶段中，所有处于当前构建阶段之前的阶段也都会被执行。

- 依赖和仓库

Maven执行时，其中一个首要目标就是检查项目的依赖。依赖是你的项目用到的jar文件（java库）。如果在本地仓库中不存在该依赖，则Maven会从中央仓库下载并放到本地仓库。本地仓库只是你电脑硬盘上的一个目录。你可以根据需要制定本地仓库的位置。你也可以指定下载依赖的远程仓库的地址。这些将会在后续的小节中详细介绍。

- 插件

构建插件可以向构建阶段中增加额外的构建目标。如果Maven标准的构建阶段和目标无法满足项目构建的需求，你可以在POM文件里增加插件。Maven有一些标准的插件供选用，如果需要你可以自己实现插件。

- 配置文件

配置文件用于以不同的方式构建项目。比如，你可能需要在本地环境构建，用于开发和测试，你也可能需要构建后用于开发环境。这两个构建过程是不同的。在POM文件中增加不同的构建配置，可以启用不同的构建过程。当运行Maven时，可以指定要使用的配置。

- 父pom

所有的Maven pom文件都继承自一个父pom。如果没有指定父pom，则该pom文件继承自根pom。pom文件的继承关系如下图所示：
![enter image description here](http://ifeve.com/wp-content/uploads/2014/06/super-pom.png)

可以让一个pom文件显式地继承另一个pom文件。这样，可以通过修改公共父pom文件的设置来修改所有子pom文件的设置。在pom文件的起始处指定父pom，例如：
``` scala
<project xmlns=”http://maven.apache.org/POM/4.0.0″
xmlns:xsi=”http://www.w3.org/2001/XMLSchema-instance”
xsi:schemaLocation=”http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd”>
<modelVersion>4.0.0</modelVersion>

<parent>
<groupId>org.codehaus.mojo</groupId>
<artifactId>my-parent</artifactId>
<version>2.0</version>
<relativePath>../my-parent</relativePath>
</parent>

<artifactId>my-project</artifactId>
…
</project>
```
子pom文件的设置可以覆盖父pom文件的设置，只需要在子pom文件里指定新的设置即可。
关于pom文件继承更详细的内容可以参考Maven POM文档。

- 有效pom

考虑到pom文件的继承关系，当Maven执行的时候可能很难确定最终的pom文件的内容。总的pom文件（所有继承关系生效后）被称为有效pom（effective pom）。可以使用以下的命令让Maven打印出当前的有效pom：

> mvn help:effective-pom

执行以上命令，Maven会将有效pom输出到命令行。

### Maven目录结构
Maven有一个标准的目录结构。如果你在项目中遵循Maven的目录结构，就无需在pom文件中指定源代码、测试代码等目录。
以下为最重要的目录：
``` java
- src
  - main
    - java
    - resources
    - webapp
  - test
    - java
    - resources

- target
```
src目录是源代码和测试代码的根目录。main目录是应用的源代码目录。test目录是测试代码的目录。main和test下的java目录，分别表示应用的java源代码和测试代码。
resources目录包含项目的资源文件，比如应用的国际化配置的属性文件等。
如果是一个web项目，则webapp目录为web项目的根目录，其中包含如WEB-INF等子目录。
target目录是由Maven创建的，其中包含编译后的类文件、jar文件等。当执行maven的clean目标后，target目录会被清空。

- 项目依赖
Maven内嵌有依赖管理的功能。你只需要在pom文件里指定依赖jar包的名称、版本号，Maven会自动下载并放到你的Maven本地仓库中。如果这些外部jar包依赖了其它的库，它们也会被下载到你的Maven本地仓库。
- 外部依赖
Maven的外部依赖指的是不在Maven的仓库（包括本地仓库、中央仓库和远程仓库）中的依赖（jar包）。它可能位于你本地硬盘的某个地方，比如web应用的lib目录下。这里的“外部”是对Maven仓库系统而言的，不仅仅是对项目而言的。大部分的外部依赖都是针对项目的，很少的外部依赖是针对仓库系统的（即不在仓库中）。
配置外部依赖的示例如下：

``` xml
<dependency>
  <groupId>mydependency</groupId>
  <artifactId>mydependency</artifactId>
  <scope>system</scope>
  <version>1.0</version>
  <systemPath>${basedir}\war\WEB-INF\lib\mydependency.jar</systemPath>
</dependency> 
```
- 快照依赖
快照依赖指的是那些还在开发中的依赖（jar包）。与其经常地更新版本号来获取最新版本，不如你直接依赖项目的快照版本。快照版本的每一个build版本都会被下载到本地仓库，即使该快照版本已经在本地仓库了。总是下载快照依赖可以确保本地仓库中的每一个build版本都是最新的。
在pom文件的最开头（设置groupId和artifactId的地方），在版本号后追加-SNAPSHOT，则告诉Maven你的项目是一个快照版本。如：
```
<version>1.0-SNAPSHOT</version>
```
可以看到加到版本号后的-SNAPSHOT。

### Maven仓库
Maven仓库就是存储jar包和一些元数据信息的目录。其中的元数据即pom文件，描述了该jar包属于哪个项目，以及jar包所需的外部依赖。该元数据信息使得Maven可以递归地下载所有的依赖，直到整个依赖树都下载完毕并放到你的本地仓库中。
Maven有三种类型的仓库：
- 本地仓库---本地仓库就是开发者电脑上的一个目录。该仓库包含了Maven下载的所有依赖。一般来讲，一个本地仓库为多个不同的项目服务。因此，Maven只需下载一次，即使有多个项目都依赖它.通过mvn install命令可以将你自己的项目构建并安装到本地仓库中。这样，你的其它项目就可以通过在pom文件将该jar包作为外部依赖来使用。
- 中央仓库---Maven的中央仓库由Maven社区提供。默认情况下，所有不在本地仓库中的依赖都会去这个中央仓库查找。然后Maven会将这些依赖下载到你的本地仓库。访问中央仓库不需要做额外的配置。
- 远程仓库---远程仓库是位于web服务器上的一个仓库，Maven可以从该仓库下载依赖，就像从中央仓库下载依赖一样。远程仓库可以位于Internet上的任何地方，也可以是位于本地网络中。远程仓库一般用于放置组织内部的项目，该项目由多个项目共享。比如，由多个内部项目共用的安全项目。该安全项目不能被外部访问，因此不能放在公开的中央仓库下，而应该放到内部的远程仓库中。远程仓库中的依赖也会被Maven下载到本地仓库中。可以在pom文件里配置远程仓库。将以下的xml片段放到属性之后：

``` xml
<repositories>
   <repository>
       <id>jenkov.code</id>
       <url>http://maven.jenkov.com/maven2/lib</url>
   </repository>
</repositories>
```
Maven根据以上的顺序去仓库中搜索依赖。首先是本地仓库，然后是中央仓库，最后，如果pom文件中配置了远程仓库，则会去远程仓库中查找。
### Maven的构建生命周期、阶段和目标
当使用Maven构建项目时，会遵循一个构建生命周期。该生命周期分为多个构建阶段，而构建阶段又分为多个构建目标。

- 构建生命周期
Maven有三个内嵌的构建生命周期：
- default
- clean
- site

每一个构建生命期关注项目构建的不同方面。因此，它们是独立地执行的。Maven可以执行多个生命期，但是它们是串行执行的，相互独立，就像你执行了多条独立的Maven命令。
default生命期关注的是项目的编译和打包。clean生命期关注的是从输出目录中删掉临时文件，包括自动生成的源文件、编译后的类文件，之前版本的jar文件等。site生命期关注的是为项目生成文档。实际上，site可以使用文档为项目生成一个完整的网站。
- 构建阶段

每一个构建生命期被分为一系列的构建阶段，构建阶段又被分为构建目标。因此，整个构建过程由一系列的构建生命期、构建阶段和构建目标组成。
你可以执行一个构建生命期，如clean或site，一个构建阶段，如default生命期的install，或者一个构建目标，如dependency:copy-dependencies。注意：你不能直接执行default生命期，你需要指定default生命期中的一个构建阶段或者构建目标。
当你执行一个构建阶段时，所有在该构建阶段之前的构建阶段（根据标准构建顺序）都会被执行。因此，执行install阶段，意味着所有位于install阶段前的构建阶段都会被执行，然后才执行install阶段。
default生命期更多的关注于构建代码。由于你不能直接执行default生命期，你需要执行其中一个构建阶段或者构建目标。default生命期包含了相当多的构建阶段和目标，这里不会所有都介绍。最常用的构建阶段有：

``` xml
构建阶段           描述
validate           验证项目的正确性，以及所有必需的信息都是否都存在。同时也会确认项目的依赖是否都下载完毕。
compile            编译项目的源代码
test               选择合适的单元测试框架，对编译后的源码执行测试；这些测试不需要代码被打包或者部署。
package            将编译后的代码以可分配的形式打包，如Jar包。
install            将项目打包后安装到本地仓库，可以作为其它项目的本地依赖。
deploy             将最终的包复制到远程仓库，与其它开发者和项目共享。    
```
- 构建目标

构建目标是Maven构建过程中最细化的步骤。一个目标可以与一个或多个构建阶段绑定，也可以不绑定。如果一个目标没有与任何构建阶段绑定，你只能将该目标的名称作为参数传递给mvn命令来执行它。如果一个目标绑定到多个构建阶段，该目标在绑定的构建阶段执行的同时被执行。

### Maven构建配置

Maven构建配置使你能使用不同的配置来构建项目。不用创建两个独立的pom文件。你只需使用不同的构建配置指定不同的配置文件，然后使用该配置文件构建项目即可。

关于构建配置的详细信息可以参考Maven POM参考的Profile部分。这里是一个快速的介绍。

Maven的构建配置在pom文件的profiles属性中指定，例如：
``` xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
   http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.jenkov.crawler</groupId>
  <artifactId>java-web-crawler</artifactId>
  <version>1.0.0</version>

  <profiles>
      <profile>
          <id>test</id>
          <activation>...</activation>
          <build>...</build>
          <modules>...</modules>
          <repositories>...</repositories>
          <pluginRepositories>...</pluginRepositories>
          <dependencies>...</dependencies>
          <reporting>...</reporting>
          <dependencyManagement>...</dependencyManagement>
          <distributionManagement>...</distributionManagement>
      </profile>
  </profiles>

</project>
```
构建配置描述的是当使用该配置构建项目时，对pom文件所做的修改，比如修改应用使用的配置文件等。profile属性中的值将会覆盖其上层的、位于pom文件中的配置。

在profile属性中，有一个activation子属性。该属性指定了启用该构建配置的条件。选择构建配置的一种方式是在settings.xml文件中指定；另一种方式是在Maven命令行使用-P profile-name指定。更多细节参考构建配置的文档。

- build配置
可以使用build配置修改相关的plugins配置
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.6</source>
                <target>1.6</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
  </plugins>
</build>
```
- Maven插件

使用Maven插件，可以向构建过程添加自定义的动作。创建一个简单的Java类，该类继承一个特殊的Maven类，然后为项目创建一个pom文件。该插件应该位于其项目下。
Maven本质上是一个插件框架，它的核心并不执行任何具体的构建任务，所有这些任务都交给插件来完成，例如编译源代码是由maven-compiler-plugin完成的。进一步说，每个任务对应了一个插件目标（goal），每个插件会有一个或者多个目标，例如maven-compiler-plugin的compile目标用来编译位于src/main/java/目录下的主源码，testCompile目标用来编译位于src/test/java/目录下的测试源码。
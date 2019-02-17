---
title: maven的聚合和继承
tags:
  - maven
abbrlink: dcf98c31
date: 2017-11-11 15:26:25
---



### 一、为什么要聚合？

随着技术的飞速发展和各类用户对软件的要求越来越高，软件本身也变得越来越复杂，然后软件设计人员开始采用各种方式进行开发，于是就有了我们的分层架构、分模块开发，来提高代码的清晰和重用。针对于这一特性，maven也给予了相应的配置。

<br/>

所谓聚合，顾名思义，就是把多个模块或项目聚合到一起，我们可以建立一个专门负责聚合工作的Maven project ---  aggregator。

<br/>

建立该project的时候，我们要注意以下几点：

1. 该aggregator本身也做为一个Maven项目，它必须有自己的POM
2. 它的打包方式必须为： pom
3. 引入了新的元素：modules---module
4. 版本：聚合模块的版本和被聚合模块版本一致
5. relative path：每个module的值都是一个当前POM的相对目录
6. 目录名称：为了方便的快速定位内容，模块所处的目录应当与其artifactId一致(Maven约定而不是硬性要求)，总之，模块所处的目录必须和<module>模块所处的目录</module>相一致。
7. 习惯约定：为了方便构建，通常将聚合模块放在项目目录层的最顶层，其它聚合模块作为子目录存在。这样当我们打开项目的时候，第一个看到的就是聚合模块的POM
8. 聚合模块减少的内容：聚合模块的内容仅仅是一个pom.xml文件，它不包含src/main/java、src/test/java等目录，因为它只是用来帮助其它模块构建的工具，本身并没有实质的内容。
9. 聚合模块和子模块的目录：他们可以是父子类，也可以是平行结构，当然如果使用平行结构，那么聚合模块的POM也需要做出相应的更改。

<br/>

### 二、为什么要继承？

做面向对象编程的人都会觉得这是一个没意义的问题，是的，继承就是避免重复，maven的继承也是这样，它还有一个好处就是让项目更加安全

<br/>

情景分析二：我们在项目开发的过程中，可能多个模块独立开发，但是多个模块可能依赖相同的元素，比如说每个模块都需要Junit，使用spring的时候，其核心jar也必须都被引入，在编译的时候，maven-compiler-plugin插件也要被引入

<br/>

如何配置继承：

1. 说到继承肯定是一个父子结构，那么我们在aggregator中来创建一个parent project
2. <packaging>: 作为父模块的POM，其打包类型也必须为POM
3. 结构：父模块只是为了帮助我们消除重复，所以它也不需要src/main/java、src/test/java等目录
4. 新的元素：<parent> ， 它是被用在子模块中的
5. .<parent>元素的属性：<relativePath>: 表示父模块POM的相对路径，在构建的时候，Maven会先根据relativePath检查父POM，如果找不到，再从本地仓库查找
6. relativePath的默认值： ../pom.xml
7. 子模块省略groupId和version： 使用了继承的子模块中可以不声明groupId和version, 子模块将隐式的继承父模块的这两个元素

<br/>

### 三、dependencyManagement

在dependencyManagement中配置的元素既不会给parent引入依赖，也不会给它的子模块引入依赖，**仅仅是它的配置是可继承的 **

<br/>

它提供了一种**管理依赖版本号 **的方式

例如在父项目里：

~~~xml
<dependencyManagement>  
	<dependencies>  
		<dependency>  
			<groupId>mysql</groupId>  
			<artifactId>mysql-connector-java</artifactId>  
			<version>5.1.2</version>  
		</dependency>  
...  
	<dependencies>  
</dependencyManagement>  
~~~

然后在子项目里就可以添加mysql-connector时可以不指定版本号，例如：

~~~xml
<dependencies>  
	<dependency>  
		<groupId>mysql</groupId>  
		<artifactId>mysql-connector-java</artifactId>  
	</dependency>  
</dependencies> 
~~~

这样做的好处就是：如果有多个子项目都引用同一样依赖，则可以避免在每个使用的子项目里都声明一个版本号，这样当想升级或切换到另一个版本时，只需要在顶层父容器里更新，而不需要一个一个子项目的修改 ；另外如果某个子项目需要另外的一个版本，只需要声明version就可。

<br/>

**dependencyManagement里只是声明依赖，并不实现引入，因此子项目需要显式的声明需要用的依赖**。

<br/>

### 四、dependencies

- 相对于dependencyManagement，所有声明在dependencies里的依赖都会自动引入，并默认被所有的子项目继承。
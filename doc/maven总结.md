# maven总结

### 1、概述

> Maven是一个项目管理工具，他包含了一个项目对象模型，一组标准集合，一个项目生命周期，一个依赖管理系统和用来运行定义生命周期阶段中插件目标的逻辑。

### 2、maven项目默认目录组织

> 默认源码：${basedir}/src/main/java
>
> 资源文件:${basedir}/src/main/resources
>
> 测试代码：${basedir}/src/main/test
>
> JAR文件存放在:${basedir}/target/classes

### 3、配置文件

#### 配置本地仓库地址

~~~xml
<localRepository>/path/to/local/repo</localRepository>
~~~

#### 配置镜像仓库URL

~~~xml
  <mirrors>
    <mirror> 
	  <id>alimaven</id> 
	  <name>aliyun maven</name>   
	  <url>http://maven.aliyun.com/nexus/content/groups/public/</url> 
	  <mirrorOf>central</mirrorOf> 
    </mirror>
  </mirrors>
~~~

+ mirror

  > mirror相当于一个**拦截器**，它会拦截maven对remote repository的相关请求，把请求里的remote repository地址，重定向到mirror里配置的地址。
  >
  > mirror表示的是两个Repository之间的关系

+ mirrorOf

  > 标签里面放置的是要被镜像的Repository ID。

  ~~~xml
  <!--  匹配所有远程仓库。  -->
  <mirrorOf>*</mirrorOf>
  <!--  匹配仓库repo1和repo2，使用逗号分隔多个远程仓库。 -->
  <mirrorOf>repo1,repo2</mirrorOf> 
  <!--  匹配所有远程仓库，repo1除外，使用感叹号将仓库从匹配中排除。 -->
  <mirrorOf>*,!repo1</miiroOf> 
  ~~~

#### 2个镜像解决方案

在**setting.xml**中配置一个代替远程仓库的镜像，其他需要单独配置的镜像在项目的**pom.xml**中配置

~~~xml
<repositories>
    <repository>
        <id>nexus-163</id>
        <name>Nexus 163</name>
        <url>http://mirrors.163.com/maven/repository/maven-public/</url>
        <layout>default</layout>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <releases>
            <enabled>true</enabled>
        </releases>
    </repository>
</repositories>
~~~

### 4、仓库

#### 本地仓库

> 默认情况下，本地仓库在用户目录下的**.m2/repository/**。我们也可以在 settings.xml 文件配置本地仓库的地址

#### 远程仓库

> 对于Maven来说，每个用户只有一个本地仓库，但是可以配置多个远程仓库。

1. 中央仓库

   > 中央仓库可以通过修改setting.xml中的mirrorOf标签，代替默认的[中央仓库](http://repo1.maven.org/maven2/ )

2. 私服

   > 内网自建的maven repository，其URL是一个内部网址 

#### 仓库优先级

> local_repo > settings_profile_repo > pom_profile_repo > pom_repositories > settings_mirror > central

#### 配置多个仓库

+ setting.xml中的profiles节点配置多个repository

  ~~~xml
  <profiles>
  	<profile> 
  	  <id>boundlessgeo</id>  
  	  <repositories> 
  	    <repository> 
  	      <id>boundlessgeo</id>  
  	      <url>https://repo.boundlessgeo.com/main/</url>  
  	      <releases> 
  	        <enabled>true</enabled> 
  	      </releases>  
  	      <snapshots> 
  	        <enabled>true</enabled>  
  	        <updatePolicy>always</updatePolicy> 
  	      </snapshots> 
  	    </repository> 
  	  </repositories> 
  	</profile> 
  	<profile> 
  	  <id>aliyun</id>  
  	  <repositories> 
  	    <repository> 
  	      <id>aliyun</id>  
  	      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>  
  	      <releases> 
  	        <enabled>true</enabled> 
  	      </releases>  
  	      <snapshots> 
  	        <enabled>true</enabled>  
  	        <updatePolicy>always</updatePolicy> 
  	      </snapshots> 
  	    </repository> 
  	  </repositories> 
  	</profile>  
  	<profile> 
  	  <id>maven-central</id>  
  	  <repositories> 
  	    <repository> 
  	      <id>maven-central</id>  
  	      <url>http://central.maven.org/maven2/</url>  
  	      <releases> 
  	        <enabled>true</enabled> 
  	      </releases>  
  	      <snapshots> 
  	        <enabled>true</enabled>  
  	        <updatePolicy>always</updatePolicy> 
  	      </snapshots> 
  	    </repository> 
  	  </repositories> 
  	</profile> 
  <profiles>
  ~~~

+ 通过配置 activeProfiles 子节点激活

  ~~~xml
  <activeProfiles>
  	<activeProfile>boundlessgeo</activeProfile>
  	<activeProfile>aliyun</activeProfile>
  	<activeProfile>maven-central</activeProfile>
  </activeProfiles>
  ~~~

  
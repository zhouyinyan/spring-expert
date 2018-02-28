# Spring Boot 之 Developer tools 

Spring Boot 包含了一组工具，让应用开发的体验更舒适一点。spring-boot-devtools模块能够在任何项目中提供额外的"开发时"特性，简单的在项目中加入模块依赖：  
Maven.  
```java
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```
Gradle.  
```java
dependencies {
    compile("org.springframework.boot:spring-boot-devtools")
}
```

## 默认属性配置

Spring Boot支持的一些库使用缓存来提高性能。比如，模板引擎会缓存编译好的模板来避免重复编译，同样，在为静态资源提供服务时，Spring MVC会在response上添加HTTP caching header。  

缓存在生产时非常有用，但在开发时就不一样了，它会阻止你看见你在应用程序中修改后带来的变化。因此，spring-boot-devtools 会默认禁用这些缓存。   

缓存选项通常配置在application.properties文件中。比如，thymeleaf 提供 spring.thymeleaf.cache 属性。 spring-boot-devtools 会自动应用开发时配置，而不需要手动配置。

全部应用的属性配置 查看[DevToolsPropertyDefaultsPostProcessor](https://github.com/spring-projects/spring-boot/tree/v1.5.10.RELEASE/spring-boot-devtools/src/main/java/org/springframework/boot/devtools/env/DevToolsPropertyDefaultsPostProcessor.java).

## 自动重启

使用spring-boot-devtools的应用程序能够在classpath下面的文件发生变化时自动重启。这个特性在使用IDE开发时非常有用，它让你可以快速的得到到代码变化后的反馈。默认，类路径的整个目录都会受到监控。注意某些资源比如静态资产文件(css，js)和视图模板不需要重启应用。  

As DevTools monitors classpath resources, the only way to trigger a restart is to update the classpath. The way in which you cause the classpath to be updated depends on the IDE that you are using. In Eclipse, saving a modified file will cause the classpath to be updated and trigger a restart. In IntelliJ IDEA, building the project (Build -> Make Project) will have the same effect. （意思是修改源代码后，要经过编译，并输出到target目录）

> spring boot的重启技术使用两个classloaders。不更改的class(比如三方jars)被base classloader加载，而正在开发中的代码通过restart classloader加载。当应用重启时，restart classloader就会被丢弃，然后重新创建一个新的。 这么做的目的就是让应用重启速度比”冷启动“开得多，因为base classloader是已经存在且可用的。 


  1. 排除resource      

     某些资源修改时，没必要触发重启。比如，thymeleaf模板。默认修改`/META-INF/maven`, `/META-INF/resources`, `/resources`, `/static`, `/public` or `/templates`这些目录的资源文件时不会触发重启，但是会触发LiveReload。如果需要自定义排除目录，使用spring.devtools.restart.exclude属性。比如仅排除`/static` 和 `/public` ：`spring.devtools.restart.exclude=static/**,public/**`

  2. 添加watching path    

     You may want your application to be restarted or reloaded when you make changes to files that are not on the classpath. 

     你可能想当没在classpath中的文件发生变化时也重启应用。可以使用使用

      `spring.devtools.restart.additional-paths` 属性来添加要监控的目录。

  3. 禁用restart  

     不想使用restart特性时，通过 `spring.devtools.restart.enabled` 属性。大部分情况下，可以配置该属性在你的`application.properties` 中。（依然会初始化restart classloader，但不会监控文件变化）。   

     比如因为重启特性与特定的库不兼容，此时想要完全禁用restart，你需要在调用`SpringApplication.run(…)`之前，设置一个 `System` 属性 .比如

     ```java
     public static void main(String[] args) {
         System.setProperty("spring.devtools.restart.enabled", "false");
         SpringApplication.run(MyApp.class, args);
     }
     ```

  4. 使用tigger file   

     If you work with an IDE that continuously compiles changed files, you might prefer to trigger restarts only at specific times.

     如果你使用的IDE会持续的自动编译修改的文件，你可能更希望在想要的时间触发重启。此时就要用到tigger file，即指定一个特定的文件，只有该文件被修改后才会触发重启。tigger file可以手动更新，也可以使用IDE插件。

     使用 `spring.devtools.restart.trigger-file` 属性来指定tigger file。

  5. 定制化重启classloader  

     As described in the [Restart vs Reload](https://docs.spring.io/spring-boot/docs/1.5.10.RELEASE/reference/htmlsingle/#using-spring-boot-restart-vs-reload) section above, restart functionality is implemented by using two classloaders. For most applications this approach works well, however, sometimes it can cause classloading issues.

     By default, any open project in your IDE will be loaded using the “restart” classloader, and any regular `.jar` file will be loaded using the “base” classloader. If you work on a multi-module project, and not each module is imported into your IDE, you may need to customize things. To do this you can create a `META-INF/spring-devtools.properties` file.

     The `spring-devtools.properties` file can contain `restart.exclude.` and `restart.include.` prefixed properties. The `include` elements are items that should be pulled up into the “restart” classloader, and the `exclude` elements are items that should be pushed down into the “base” classloader. The value of the property is a regex pattern that will be applied to the classpath.

     For example:

     ```java
     restart.exclude.companycommonlibs=/mycorp-common-[\\w-]+\.jar
     restart.include.projectcommon=/mycorp-myproj-[\\w-]+\.jar
     ```

     （笔者认为，这种问题应该通过约定来规避）。

  6. 限制  
使用standard ObjectInputStream反序列化时，重启功能不能很好的工作，如果需要反序列化数据，需要Spring’s ConfigurableObjectInputStream 和 Thread.currentThread().getContextClassLoader() 组合使用。  

不幸的是，许多三方库反序列化时未考虑到context classloader。如果发现存在问题时，需要给源作者提交fix request。


## live reload

spring-boot-devtools 包含一个内置的LiveReload server，用于当一个资源修改时触发浏览器刷新。LiveReload 浏览器扩展可以从[livereload.com](http://livereload.com/extensions/)获取，支持Chrome，firefox和safari.  
如果不想启动LiveReload server，设置spring.devtools.livereload.enabled 为false。 

## 全局设置

可以通过一个放置$HOME目录中，名为.spring-boot-devtool.properties文件来配置全局的devtools(注意文件名以"."开头)。所有添加在该文件中的配置会应用到机器上所有开发的spring boot应用。比如配置使用同一个tigger file来触发自动重启：
~/.spring-boot-devtools.properties. 
```java
spring.devtools.reload.trigger-file=.reloadtrigger
```

## 远程

笔者目前还未使用，不做说明，感兴趣者参考spring-boot文档。 


参考：

https://docs.spring.io/spring-boot/docs/1.5.10.RELEASE/reference/htmlsingle/#using-boot-devtools



## 实际例子

### 在Intellij IDEA中实际应用 

1. 添加依赖
```java
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <optional>true</optional>
</dependency>
```

2. chrome 浏览器安装live Reload扩展插件

3. 通过mvn spring-boot run 启动spring boot应用。（注：默认在IDEA中，右键XXXApplication启动类，在启动完成后会自动停止）

4. 修改java源文件，保存（commond+s），编译项目（commond+f9），此时应用重启。也可以重新编译单个源文件（shift+commond+f9）。

   IDEA中build project 和rebuild project 的区别是什么？

5. 修改thymeleaf模板文件，保存（commond+s），编译项目（commond+f9），应用不重启，触发LiveReload。浏览器开启live reload插件，则可以立即查看到变化。 


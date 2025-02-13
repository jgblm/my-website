# 廖雪峰手写Spring

原文链接：https://liaoxuefeng.com/books/summerframework/introduction/index.html

## 1. 实现IOC容器

IOC容器的功能：通过@ComponentScan扫描package下的所有@Configuration和@Component，根据Component的依赖关系，实例化和注入Component，放入ApplicationContext容器中。

1. 扫描获取所有的类 **ResourceResolver**
2. 配置属性获取与解析  **PropertyResolver**
3. 为了介入Bean的整个创建过程，并不直接创建Bean实例，而是先根据配置创建**BeanDefination**.
4. 根据BeanDefination创建**Bean**。对于存在强依赖的Bean，需要递归创建依赖的bean。对于弱依赖的bean，创建完成后再进行set。
5. 初始化Bean，调用initMethod。
6. **BeanPostProcessor**用来替换bean，比如代理类。为了保证被注入时使用的是代理类，注入时使用的是原始类，BeanPostProcessor需要提供一个方法获取原始的bean。

## 2. 实现AOP

AOP本质上是proxy模式。在运行期进行代理有两种实现方式：

1. Java标准库的动态代理机制，不过只能代理接口；
2. CGLIB，可以对具体实现类进行代理。cglib会生成对应类的子类。

Spring中如何创建代理类：在BeanPostProcessor中对需要进行AOP的bean进行代理，返回代理类。

> Spring AOP统一存在的问题：内部方法相互调用，不会走aop。

## 3. 实现JDBC和事务

**JdbcTemplate**

JdbcTemplate基于template模式。由JdbcTemplate处理获取连接、释放连接、捕获SQLException，回调方法专注于使用connection。(实际实现JdbcTemplate中更进一步创建Statement, 回调方法只需专注于Statement执行sql并解析ResultSet)

```java
// template模式
public <T> T execute(StatementCallback<T> action) {
    try (Connection con = DataSourceUtils.getConnection()) {
      Statement stmt = con.createStatement();
      T result = action.doInStatement(stmt);
      return result;
    } catch (SQLException ex) {
      throw new DataAccessException(ex);
    }
}

// 回调方法
@FunctionalInterface
public interface StatementCallback<T> {
    @Nullable
    T doInStatement(Statement stmt) throws SQLException;
}

// 实现查询的回调方法
public class QueryStatementCallback implements StatementCallback<T> {
    @Override
    public T doInStatement(Statement stmt) throws SQLException {
      try (ResultSet rs = stmt.executeQuery(sql)) {
        return extractData(rs);
      }
    }
 }

```

**声明式事务**

对@Transactional注解的类进行代理。

进行方法前，开启事务；

方法执行后，执行成功commit，异常rollback。

## 4. 实现Web MVC

#### 1 启动IOC容器

Servlet规范定义的组件有3类：

1. Servlet：处理HTTP请求，然后输出响应；
2. Filter：对HTTP请求进行过滤，可以有多个Filter形成过滤器链，实现权限检查、限流、缓存等逻辑；
3. Listener：用来监听Web应用程序产生的事件，包括启动、停止、Session有修改等。

Tomcat服务器启动一个基于Spring开发的Web应用程序的执行顺序：

1. 创建Servlet容器
2. 初始化一个指定的Listener(`ContextLoaderListener`)：这个Listener在初始化的时候会启动IOC容器，并将DispatcherServlet注册到Servlet容器。

#### 2. 实现MVC

根据SpringMVC的各种注解拿到Controller的各种配置，在DispatcherServlet中根据这些配置去匹配对应的Controller执行逻辑，返回结果会按照类型和配置进行渲染。

## 5. 实现Spring Boot

SpringMVC开发完成后，还需要打包成war，放到tomcat的webapp目录下，重新启动tomcat才行。Spring Boot省略了这些麻烦的部署流程，直接打包成jar包就可以运行。

#### 1. 启动嵌入式tomcat

1. main方法中启动嵌入式tomcat
2. tomcat启动的时候会注册一个listener `ServletContainerInitializer` 。这个Listener在初始化的时候会启动IOC容器，并将DispatcherServlet注册到Servlet容器，注册Filter。

#### 2. 打包jar

1. 提供maven插件按照如下格式打包
2. 提供ClassLoader去加载BOOT-INF/下的class和jar

```
xyz.jar
├── BOOT-INF
│   ├── classes
│   │   ├── application.yml
│   │   ├── logback-spring.xml
│   │   ├── static
│   │   │   └── ... static files ...
│   │   └── templates
│   │       └── ... templates ...
│   └── lib
│       ├── spring-boot-3.0.0.jar
│       └── ... other jars ...
├── META-INF
│   └── MANIFEST.MF   // 指定了Main-Class（JarLauncher）
└── org
    └── springframework
        └── boot
            └── loader
                ├── JarLauncher.class // ClassLoader去BOOT-INF/下加载class和jar包
                └── ... other classes ...
```


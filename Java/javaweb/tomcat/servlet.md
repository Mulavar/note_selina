## Servlet

+ [Servlet](#Servlet)
	+ [Servlet生命周期](#servlet简介)  
    + [Servlet生命周期](#servlet生命周期)  
    + [Servlet运行过程](#servlet运行过程)  
    + [Servlet创建方式](#servlet创建方式)  
    + [Servlet配置](#servlet配置)  


### Servlet简介
Servlet是sun公司开发动态web的一门技术，核心是一个Servlet接口，里面的核心方法是`serve`，`HttpServlet`实现了该接口，并将该方法的逻辑拆分为了`doGet`和`doPost`。因此平时写javaweb时，我们都会创建一个servlet继承HttpServlet类，并重写这两个方法。



### Servlet生命周期

1. `init(ServletConfig config)`

    servlet对象创建的时候执行，**默认**第一次访问servlet时创建该对象，即servlet对应的虚拟路径被第一次访问的时候创建(**而非服务器刚启动的时候**)

    **config**：获得servlet的name——`<servlet-name>${name}$</servlet-name>`

2. `service(ServletRequest req, ServletResponse res)`

    每次请求都会执行

    **req**：**ServletRequest**内部封装的是http请求的信息

    **res**：代表响应认为要封装的响应的信息

3. `destory()`

    servlet销毁的时候执行，服务器关闭的时候销毁

所以servlet的运行过程可以表示如下。



### Servlet运行过程

1. 查看是否已有该servlet的实例对象，如果有，跳到第3步；
2. 创建该servlet对应的实例对象，并调用init()；
3. 创建对应的request、response(空)作为方法参数调用service()；
4. 销毁servlet对象，调用destory()。



### Servlet创建方式

两种创建方式：

1. 实现`Servlet`接口

    + 覆盖尚未实现的方法(核心是**service**方法)

    + **web.xml**配置（为什么需要配置？）

      servlet只是一个类，想要通过浏览器访问，需要映射到一个路径。

    ```xml
    <servlet>
    	<servlet-name>${name}</servlet-name>
    	<servlet-class>${全限定类名}</servlet-class>
    </servlet>
    <servlet-mapping>
    	<servlet-name>${name}</servlet-name>
    	<url-pattern>${虚拟映射路径}</url-pattern>
    </servlet-mapping>
    ```

2. 继承`HttpServlet`类

    覆盖doGet()、doPost()方法
    
    

### Servlet配置

1. `url-pattern`配置

    + 完全匹配

        `<url-pattern>/abc</url-pattern>`

    + 目录匹配

        `<url-pattern>/abc/*</url-pattern>`

    + 扩展名匹配

        `<url-pattern>*.abc</url-pattern>`

        **Note**: 扩展名匹配和目录匹配不能一起用

2. **启动配置**

    ```xml
    <servlet>
    	<load-on-startup>3</load-on-startup>
    </servlet>
    ```

3. **默认servlet**

    `<url-pattern>/</url-pattern>`

    当访问资源地址所有的servlet都不匹配时，默认的servlet负责处理。

    **Note**：web应用中所有的资源的响应都是servlet负责，包括静态资源。

4. ` <welcome-file-list>`配置

    当访问该web应用又不指定资源路径的时候，tomcat会根据配置文件的`<welcome-file-list>`从上往下依次寻找文件名对应的页面，并显示。

    ```xml
    <welcome-file-list>
        <welcome-file>index.html</welcome-file>
        <welcome-file>index.htm</welcome-file>
        <welcome-file>index.jsp</welcome-file>
        <welcome-file>default.html</welcome-file>
        <welcome-file>default.htm</welcome-file>
        <welcome-file>default.jsp</welcome-file>
      </welcome-file-list>
    ```

    

    
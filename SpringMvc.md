# SpringMvc简单介绍

spring开发的springMvc框架

多了一个前端控制器

# HelloWorld细节

客户端访问当前项目下的/hello请求

SpringMvc的前端控制器接受到所有请求

然后来看请求地址和RequestMapping标注的哪个匹配,来找到到底使用哪个类的哪个方法来执行

方法执行完成以后会有一个返回值,SpringMvc认为这个返回值就是要去的页面

拿到返回值以后用视图解析器进行拼串得到完整的页面地址

拿到页面地址以后,前端控制器帮我们转发页面

如果不指定配置文件

就在

/WEB-INF下创建一个前端控制器名-servlet.xml文件

会在/WEB-INF/xxx name-servlet.xml 

# URL配置和RequsetMap方法上的参数

/*和/都是拦截所有请求

/**范围更大      /* *还能拦截JSP页面

/是把tomcat服务器下的DefaultServlet的url / 重写了

DefaultServlet的/ 默认是拦截所有静态资源

前端控制器相当于重写了DefaultServlet 

为什么jsp又可以访问,因为我们没有覆盖服务器的JSPServlet

写/也是为了迎合后来的Rest风格

4开头都是客户端错误	

规定RequestMapping请求方式和请求参数

params(必须带这个参数)	!params(必须不携带这个参数)

还可以赋值 params=123

eg: params={"!username"} 客户端请求的时候必须不带username该参数,否则404

eg:params={"username=123","pwd","!age"}

 请求参数必须有username=123 并且必须存在pwd 并且不能存在age参数

headers 可以规定只有火狐可以访问,谷歌不可以访问

consums 规定请求头中的Content-type,只接受这种类型

produces 给浏览器加上Content-type 给响应头加上Content-type  

## RequstMapping模糊匹配

ant风格 来源于ant项目构建工具

URL地址可以写模糊的通配符

? 能替代任意一个字符

*能替代任意多个字符和一层路径

**能替代多层路径

模糊和精确多个匹配情况下 精确优先

模糊优先级 ?>*>**

RequestParam RequestHeader CoookieValue 

这三个参数用户一致 不带参数会报错

required false 可以不带



视图解析器作用是得到视图view对象

view对象给request域中封装数据并且转发到页面 如InternalResourceView 渲染视图



![image-20200819154624957](D:\学习笔记\image-20200819154624957.png)

 	

需要配置 如果都不配置 静态资源没法访问

如果只配置第一个 动态资源没法访问 需要开启王者开挂模式

foreach 

遍历的是list Index表示索引 item表示值

map  index表示key  item 表示value

otherwise 否则 navigation 导航 graph 图标

OGnl

![image-20200825214219283](D:\学习笔记\image-20200825214219283.png)

_parameter 代表传入的参数

传入单个参数 代表该参数

传入多个参数 封装map 代表该map VENDOR 卖主

provider 供应商

<databaseIdProvide> 该标签指定特定的数据库

type=DB_VENDOR

execution 执行

expression=execution(* com.feng.*. *(..))

  第一个* 代表任意返回值 然后空格 具体包名 再*就是所有类再 * 就是所有方法 

(..)代表所有参数

<tx:method> name=* 代表所有方法都是事务方法

roolback-for=异常全类名

切入点表达式只是说事务管理器要切入这些方法 并没有说这些方法是事务方法，要不要控制事务 但是就是奥里给！	
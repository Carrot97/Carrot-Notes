#  Spring

### IOC的理解

​		IOC（Inversion Of Controll，控制反转）是一种设计思想，就是将原本在程序中手动创建对象的控制权，交由给Spring框架来管理。IOC在其他语言中也有应用，并非Spring特有。IOC容器是Spring用来实现IOC的载体，IOC容器实际上就是一个Map(key, value)，Map中存放的是各种对象。

​		将对象之间的相互依赖关系交给IOC容器来管理，并由IOC容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。IOC容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

### AOP的理解

​		AOP（Aspect-Oriented Programming，面向切面编程）能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可扩展性和可维护性。

​		Spring AOP是基于动态代理的，如果要代理的对象实现了某个接口，那么Spring AOP就会使用JDK动态代理去创建代理对象；而对于没有实现接口的对象，就无法使用JDK动态代理，转而使用CGlib动态代理生成一个被代理对象的子类来作为代理。

### Bean的作用域

1. singleton：单例，默认bean是单例的；
2. prototype：每次请求都会创建一个实例；
3. request：每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效；
4. session：每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP session内有效。

### 单例bean的线程安全问题

​		单例bean存在线程问题，主要是因为当多个线程操作同一个对象的时候，对这个对象的非静态成员变量的写操作会存在线程安全问题。

有两种常见的解决方案：

1. 在bean对象中尽量避免定义可变的成员变量（不太现实）；

2. 在类中定义一个ThreadLocal成员变量，将需要的可变成员变量保存在ThreadLocal中（推荐的一种方式）。

### Bean的生命周期

1. 在配置文件中找到Bean的定义；
2. 利用反射API创建Bean实例；
3. 设置对象属性；
4. 检查Aware相关的接口，并设置相关的依赖；
5. 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行前置处理方法；
6. 如果Bean实现了InitializingBean接口，执行afterPropertiesSet()方法；
7. 如果Bean在配置文件中的定义包含init-method属性，执行指定的方法；
8. 如果有和加载这个Bean的Spring容器相关的BeanPostProcess对象，执行后置处理方法；
9. 使用中；
10. 当要销毁Bean的时候，如果Bean实现了DisposableBean接口，执行destroy()方法。
11. 要销毁Bean的时候，如果Bean在配置文件中的定义包含destroy-method属性，执行指定的方法。

### MVC的理解

​		MVC是一种设计模式，Spring MVC是一款很优秀的MVC框架。Spring MVC可以帮助我们进行更简洁的Web层的开发，并且它天生与Spring框架集成。Spring MVC下我们一般把后端项目分为Service层（处理业务）、Dao层（数据库操作）、Entity层（实体类）、Controller层（控制层，返回数据给前台页面）。

​		Model是系统中涉及的数据，也就是dao和bean；View是用来展示模型中的数据，只是用来展示；Controller是将用户请求都发送给Servlet做处理，返回数据给JSP并展示给用户。

### 事务的传播行为

支持当前事务的情况：

1. PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务；
2. PROPAGATION_SUPPORTS： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行；
3. PROPAGATION_MANDATORY： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）。

不支持当前事务的情况：

1. PROPAGATION_REQUIRES_NEW： 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
2. PROPAGATION_NOT_SUPPORTED： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
3. PROPAGATION_NEVER： 以非事务方式运行，如果当前存在事务，则抛出异常。
4. PROPAGATION_NESTED： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于PROPAGATION_REQUIRED。


## 前言

在实际开发中，如果对象的创建和依赖关系都在代码中写死，

比如在Impl层new一个UserDaoImpl对象，并对该对象进行一系列逻辑操作：

```typescript
public class UserServiceImpl implements UserService {

    private UserDao userDao = new UserDaoImpl();

    @Override
    public List<User> findAll() {
    //对userDao进行操作的逻辑
    }
}
```

随着项目变大，各个模块之间会形成紧密耦合，后期无论是功能扩展还是修改实现，都需要频繁改动业务代码。IoC 和 DI 通过把依赖关系交给容器统一管理，让各个组件只关注自己的业务逻辑，从而降低系统耦合度，提高代码的可维护性和扩展性。

另外，IoC 和 DI 还能显著提升代码的可测试性。没有它们时，依赖对象通常在类内部直接创建，单元测试时很难进行模拟；而通过依赖注入，可以方便地替换为 Mock 对象，更好地对业务逻辑进行测试。



# 一.控制反转（IoC）

**定义：**&#x49;nversion Of Control，简称**IOC**。对象的创建控制权由程序自身转移到外部（容器），这种思想称为控制反转。

* 对象的创建权由程序员主动创建转移到容器(由容器创建、管理对象)。这个容器称为：IOC容器或Spring容器。



## 1.为什么需要控制反转

在之前的学习中，我们需要什么对象，直接new一个对象就可以了

比如在Controller层：

```java
@RestController
public class UserController {

    private UserService userService = new UserServiceImpl();

    @RequestMapping("/list")
    public List<User> list2() {
        // 1. 调用 Service，查询用户信息
        List<User> userList = userService.list();
        // 2. 响应数据
        return userList;
    }
}

```

实现类层：

```typescript
public class UserServiceImpl implements UserService {

    private UserDao userDao = new UserDaoImpl();

    @Override
    public List<User> list() {

        // 1. 调用 dao 获取数据
        List<String> lines = userDao.list();

        // 2. 解析数据，封装成对象 -> 集合
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];

            return new User(id, username, password, name);
        }).collect(Collectors.toList());

        return userList;
    }
}
```

如果说我们需要更换实现类，比如由于业务的变更，UserServiceImpl 不能满足现有的业务需求，我们需要切换为 UserServiceImpl2 这套实现，就需要修改Contorller的代码，需要创建 UserServiceImpl2 的实现`new UserServiceImpl2()` 。

```javascript
public class UserServiceImpl2 implements UserService {

    private final UserDao userDao = new UserDaoImpl();

    @Override
    public List<User> list() {

        // 1. 调用 dao 获取数据
        List<String> lines = userDao.list();

        // 2. 解析数据，封装成对象 -> 集合
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];
            Integer age = Integer.parseInt(parts[4]);

            return new User(id, username, password, name, age);
        }).collect(Collectors.toList());

        return userList;
    }
}
```

Service中调用Dao，也是类似的问题。这种呢，我们就称之为层与层之间 **耦合** 了。 那什么是耦合呢 ？

首先需要了解软件开发涉及到的两个概念：内聚和耦合。

* **内聚：**&#x8F6F;件中各个功能模块内部的功能联系。

* **耦合：**&#x8861;量软件中各个层/模块之间的依赖、关联的程度。



**软件设计原则：高内聚低耦合。**

> **高内聚：**&#x6307;的是一个模块中各个元素之间的联系的紧密程度，如果各个元素(语句、程序段)之间的联系程度越高，则内聚性越高，即 "高内聚"。
>
> **低耦合：**&#x6307;的是软件中各个层、模块之间的依赖关联程序越低越好。



目前层与层之间是存在耦合的，Controller耦合了Service、Service耦合了Dao。而 高内聚、低耦合的目的是使程序模块的可重用性、移植性大大增强。

那最终我们的目标呢，就是做到层与层之间，尽可能的降低耦合，甚至解除耦合。

![](images/image-5.png)

## 2.解耦思路

之前我们在编写代码时，需要什么对象，就直接new一个就可以了。 这种做法呢，层与层之间代码就耦合了，当service层的实现变了之后， 我们还需要修改controller层的代码。

那应该怎么解耦呢？



**1). 首先不能在EmpController中使用new对象。代码如下：**

```javascript
@RestController
public class UserController {

    private UserService userService;

    @RequestMapping("/list")
    public List<User> list() {
        // 1. 调用 Service
        List<User> userList = userService.findAll();
        // 2. 响应数据
        return userList;
    }

}
```

此时，就存在另一个问题了，不能new，就意味着没有业务层对象（程序运行就报错），怎么办呢?&#x20;

我们的解决思路是：

* 提供一个容器，容器中存储一些对象(例：UserService对象)

* Controller程序从容器中获取UserService类型的对象



**2). 将要用到的对象交给一个容器管理。**

![](images/image-3.png)



**3). 应用程序中用到这个对象，就直接从容器中获取**

![](images/image-4.png)

那问题来了，我们如何将对象交给容器管理呢？ 程序运行时，容器如何为程序提供依赖的对象呢？&#x20;

我们想要实现上述解耦操作，就涉及到Spring中的两个核心概念：**控制反转和依赖注入。**



## 3. IoC 的核心思想与实现机制

### 3.1 IoC 的核心思想

IoC（Inversion of Control，控制反转）的核心思想是：**将对象的创建和依赖关系的控制权，从应用程序本身交给外部容器来管理**。



通过这种方式，应用程序不再主动控制对象之间的依赖关系，而是由容器统一协调，从而实现组件之间的解耦。

***

### 3.2 IoC的配置驱动

在 Spring 中，IoC 主要通过“配置”的方式来实现，对象的创建和依赖关系都由配置来描述。常见的配置方式有以下几种。

***

#### 3.2.1 XML配置

模版：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

</beans>
```

* `<beans>` 是 Spring 容器配置根标签

* `xmlns:context` 用来开启组件扫描、读取属性文件等

**XML方式声明一个Bean:**

```python
<bean id="userDao" class="com.example.dao.UserDao"/>
<bean id="userService" class="com.example.service.UserService"/>
```

* `id`：Bean 名称（容器里的唯一标识）

* `class`：要创建的类（反射 new）

此时对象确实被容器创建了，但还没建立依赖关系。

***

#### 3.2.2 Java 配置类方式

除了 XML 配置，Spring 也支持使用 Java 类进行配置，这种方式更加直观、类型安全。

```java
@Configuration
public class AppConfig {

    @Bean
    public ProductRepository productRepository() {
        return new ProductRepositoryImpl();
    }

    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl();
    }

    @Bean
    public OrderRepository orderRepository() {
        return new OrderRepositoryImpl();
    }

    @Bean
    public OrderService orderService(ProductRepository productRepository,
                                     UserRepository userRepository,
                                     OrderRepository orderRepository) {
        return new OrderService(productRepository, userRepository, orderRepository);
    }
}
```

在这种方式下，`@Configuration` 用于标识配置类，`@Bean` 方法返回的对象会被 Spring 容器管理。方法参数即为依赖对象，Spring 会自动完成注入。

***

#### 3.2.3 基于注解的自动扫描

在实际开发中，最常见的方式是通过注解配合组件扫描来完成 IoC 配置。

修改之前耦合的代码

**1). 将Service及Dao层的实现类，交给IOC容器管理**

在实现类加上 `@Component` 注解，就代表把当前类产生的对象交给IOC容器管理。

**A. UserDaoImpl**

```typescript
@Component
public class UserDaoImpl implements UserDao {
    @Override
    public List<String> findAll() {
        InputStream in = this.getClass().getClassLoader().getResourceAsStream("user.txt");
        ArrayList<String> lines = IoUtil.readLines(in, StandardCharsets.UTF_8, new ArrayList<>());
        return lines;
    }
}
```



**B. UserServiceImpl**

```java
@Component
public class UserServiceImpl implements UserService {

    private UserDao userDao;

    @Override
    public List<User> findAll() {
        List<String> lines = userDao.findAll();
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];
            Integer age = Integer.parseInt(parts[4]);
            LocalDateTime updateTime = LocalDateTime.parse(parts[5], DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return new User(id, username, password, name, age, updateTime);
        }).collect(Collectors.toList());
        return userList;
    }
}
```



**2). 为Controller 及 Service注入运行时所依赖的对象**

**A. UserServiceImpl**

```java
@Component
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao userDao;
    
    @Override
    public List<User> findAll() {
        List<String> lines = userDao.findAll();
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];
            Integer age = Integer.parseInt(parts[4]);
            LocalDateTime updateTime = LocalDateTime.parse(parts[5], DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return new User(id, username, password, name, age, updateTime);
        }).collect(Collectors.toList());
        return userList;
    }
}
```

**B. UserController**

```java
@RestController
public class UserController {
    
    @Autowired
    private UserService userService;

    @RequestMapping("/list")
    public List<User> list(){
        //1.调用Service
        List<User> userList = userService.findAll();
        //2.响应数据
        return userList;
    }

}
```

***

### 3.3. IoC到底怎么将控制反转的？

**控制反转（IoC）**，反转的不是“代码逻辑”，而是反转“对象创建与依赖关系维护”的**控制权**，之前的业务代码自己new对象，自己决定用什么方式实现，自己管理生命周期，这样不仅代码逻辑会冗余，而且编写难度也大，层与层之间的耦合也高。

**IoC提出了将对象的控制权交给Spring的IoC容器，由IoC容器创建及管理对象，当你需要用到某个对象时，容器会直接把该依赖给送过来。实现了“控制权”从业务代码 -> 转移到容器。**



## 4.Bean的声明

**被 Spring IOC 容器创建、管理的对象称为bean对象。**

在之前的案例中，要把某个对象交给IOC容器管理，需要在类上添加一个注解：**`@Component`**

而Spring框架为了更好的标识web应用程序开发当中，bean对象到底归属于哪一层，又提供了@Component的衍生注解：

那么此时，我们就可以使用 `@Service` 注解声明Service层的bean。 使用 `@Repository` 注解声明Dao层的bean。 代码实现如下：

**Service层:**

```javascript
@Service
public class UserServiceImpl implements UserService {

    private UserDao userDao;

    @Override
    public List<User> findAll() {
        List<String> lines = userDao.findAll();
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];
            Integer age = Integer.parseInt(parts[4]);
            LocalDateTime updateTime = LocalDateTime.parse(parts[5], DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return new User(id, username, password, name, age, updateTime);
        }).collect(Collectors.toList());
        return userList;
    }
}
```

**Dao层:**

```typescript
@Repository
public class UserDaoImpl implements UserDao {
    @Override
    public List<String> findAll() {
        InputStream in = this.getClass().getClassLoader().getResourceAsStream("user.txt");
        ArrayList<String> lines = IoUtil.readLines(in, StandardCharsets.UTF_8, new ArrayList<>());
        return lines;
    }
}
```

> **注意1**：声明bean的时候，可以通过注解的value属性指定bean的名字，如果没有指定，默认为类名首字母小写。
>
> **注意2**：使用以上四个注解都可以声明bean，但是在springboot集成web开发中，声明控制器bean只能用@Controller。



## 5.组件扫描

> 问题：使用前面学习的四个注解声明的bean，一定会生效吗？
>
> 答案：不一定。（原因：bean想要生效，还需要被组件扫描）

* 前面声明bean的四大注解，要想生效，还需要被组件扫描注解 `@ComponentScan` 扫描。

* 该注解虽然没有显式配置，但是实际上已经包含在了启动类声明注解 `@SpringBootApplication` 中，默认扫描的范围是启动类所在包及其子包。

![](images/image-2.png)

所以，我们在项目开发中，只需要按照如上项目结构，将项目中的所有的业务类，都放在启动类所在包的子包中，就无需考虑组件扫描问题。

**组件扫描的简化流程：**

1. 确定扫描包路径

2. 遍历 class 文件

3. 判断是否有组件注解

4. 解析类信息（不创建对象）

5. 注册为 `BeanDefinition`

6. IOC 容器后续统一实例化

**⚠️ 注意：扫描阶段 ≠ 创建对象阶段**



# 二.依赖注入（DI）

依赖注入（Dependency Injection，简称**DI**）是实现控制反转（IoC）的一种具体手段。它强调的是如何将对象所依赖的其他对象传递给这个对象。可以把 IoC 看作是一种设计理念，而 DI 是实现这种理念的具体技术。



容器为应用程序提供运行时，所依赖的资源，称之为依赖注入。



依赖注入的核心思想是将这些被依赖者对象的创建和获取从依赖者对象中分离出来，由外部（通常是一个容器，如 Spring 的 IoC 容器）负责将被依赖者对象 “注入” 到依赖者对象中。这样，依赖者对象不需要知道被依赖者对象是如何创建的，只需要使用它提供的功能即可。



在之前的程序案例中，我们使用了@Autowired这个注解，完成了依赖注入的操作，而这个Autowired翻译过来叫：**自动装配。**

`@Autowired`注解，默认是按照**类型**进行自动装配的（去IOC容器中找某个类型的对象，然后完成注入操作）

> 入门程序举例：在EmpController运行的时候，就要到IOC容器当中去查找EmpService这个类型的对象，而我们的IOC容器中刚好有一个EmpService这个类型的对象，所以就找到了这个类型的对象完成注入操作。



## 1.@Autowired用法

@Autowired 进行依赖注入，常见的方式，有如下三种：

1\). 属性注入

```java
@RestController
public class UserController {

    //方式一: 属性注入
    @Autowired
    private UserService userService;
    
  }
```

* 优点：代码简洁、方便快速开发。

* 缺点：隐藏了类之间的依赖关系、可能会破坏类的封装性。不利于单测、反射注入、依赖不透明。



2\). 构造函数注入

```java
@RestController
public class UserController {

    //方式二: 构造器注入
    private final UserService userService;
    
    @Autowired //如果当前类中只存在一个构造函数, @Autowired可以省略
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
 }   
```

* 优点：能清晰地看到类的依赖关系、提高了代码的安全性。依赖不可变、强制注入、易测试

* 缺点：代码繁琐、如果构造参数过多，可能会导致构造函数臃肿。

* **注意：如果只有一个构造函数，@Autowired注解可以省略。（通常来说，也只有一个构造函数）**



3\). setter注入

```java
/**
 * 用户信息Controller
 */
@RestController
public class UserController {
    
    //方式三: setter注入
    private UserService userService;
    
    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
    
}    
```

* 优点：保持了类的封装性，依赖关系更清晰。

* 缺点：需要额外编写setter方法，增加了代码量。

> 在项目开发中，基于@Autowired进行依赖注入时，基本都是第一种和第二种方式。（官方推荐第二种方式，因为会更加规范）但是在企业项目开发中，很多的项目中，也会选择第一种方式因为更加简洁、高效（在规范性方面进行了妥协）。



## 2.注意事项

那如果在IOC容器中，存在多个相同类型的bean对象，会出现什么情况呢？

在下面的例子中，我们准备了两个UserService的实现类，并且都交给了IOC容器管理。 代码如下：

![](images/image-1.png)

此时，我们启动项目会发现，控制台报错了：

![](images/image.png)

出现错误的原因呢，是因为在Spring的容器中，UserService这个类型的bean存在两个，框架不知道具体要注入哪个bean使用，所以就报错了。



如何解决上述问题呢？Spring提供了以下几种解决方案：

* @Primary

* @Qualifier

* @Resource



**方案一：使用@Primary注解**

当存在多个相同类型的Bean注入时，加上@Primary注解，来确定默认的实现。

```java
@Primary
@Service
public class UserServiceImpl implements UserService {
}
```



**方案二：使用@Qualifier注解**

指定当前要注入的bean对象。 在@Qualifier的value属性中，指定注入的bean的名称。 @Qualifier注解不能单独使用，必须配合@Autowired使用。

```java
@RestController
public class UserController {

    @Qualifier("userServiceImpl")
    @Autowired
    private UserService userService;
```



**方案三：使用@Resource注解**

是按照bean的名称进行注入。通过name属性指定要注入的bean的名称。

```java
@RestController
public class UserController {
        
    @Resource(name = "userServiceImpl")
    private UserService userService;
```



> 面试题：@Autowird 与 @Resource的区别
>
> * @Autowired 是spring框架提供的注解，而@Resource是JDK提供的注解
>
> * @Autowired 默认是按照类型注入，而@Resource是按照名称注入


































































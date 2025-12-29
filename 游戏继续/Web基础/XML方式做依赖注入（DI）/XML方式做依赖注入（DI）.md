#### 方式 A：Setter 注入（property）

前提：你的类有 setter

```typescript
public class UserService {private UserDao userDao;public void setUserDao(UserDao userDao) { this.userDao = userDao; }
}
```

XML：

```python
<bean id="userDao" class="com.example.dao.UserDao"/>

<bean id="userService" class="com.example.service.UserService"><property name="userDao" ref="userDao"/>
</bean>
```

解释：

* `name="userDao"` 对应 `setUserDao`

* `ref="userDao"` 表示引用容器里的另一个 Bean

这就是 DI：把 `userDao` 注入到 `userService`。

***

#### 方式 B：构造器注入（constructor-arg）

前提：有构造器

```java
public class UserService {private final UserDao userDao;public UserService(UserDao userDao) { this.userDao = userDao; }
}
```

XML：

```python
<bean id="userDao" class="com.example.dao.UserDao"/>

<bean id="userService" class="com.example.service.UserService"><constructor-arg ref="userDao"/>
</bean>
```

如果构造器参数多，可以用 `index` 或 `name` 指定：

```javascript
<constructor-arg index="0" ref="userDao"/>
```

***

### XML 开启组件扫描（把注解方式也纳入 XML 管理）

`<context:component-scan base-package="com.example"/>`

只要写了这一句：

* `@Component/@Service/...` 就会生效

* 你不需要再手写 `<bean>`（除非某些特殊对象）

***

### scope、生命周期等常用 XML 配置

#### 单例/多例

```python
<bean id="userService" class="com.example.service.UserService" scope="singleton"/>
<bean id="tempObj" class="com.example.Temp" scope="prototype"/>
```

#### 初始化与销毁方法

```javascript
<bean id="dataSource"
      class="com.example.DataSource"
      init-method="init"
      destroy-method="close"/>
```






















































































































































































































































































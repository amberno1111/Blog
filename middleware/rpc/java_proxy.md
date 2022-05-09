# [RPC预备知识]Java中的代理

代理技术是指，在不改变原始类(或者叫被代理类)代码的情况下，通过引入代理类来给增强原始类的功能。代理技术常用于日志、计数器等等场景当中，作用是用于实现框架代码跟业务代码解耦。

在Java中，代理技术有静态代理和动态代理之分：

- 静态代理是指代理类在程序运行前就已经存在，往往是在代码编译的时候就存在的。静态代理类可以通过程序员手写，或者编译时自动生成等方式实现。当我们在实现设计模式中的代理模式时，我们就是在手写静态代理。

- 动态代理是指代理类在程序运行时被创建，比如Spring的AOP、cglib动态代理等。动态代理的好处是足够灵活。

## 一、设计模式：Proxy Pattern

代理模式有两种实现方法：

- 基于接口的实现：原始类和代理类都实现了同一个接口，依据基于接口编程的思想，在使用时创建代理类对象来替代原始类对象

- 基于继承的实现：代理类通过继承原始类并override原始类方法的方式，来实现对原始类对象的代理

### 1.1 基于接口实现的代理模式

我们来看一个日志打印的需求，要求是在查询数据库的方法前后，打印出入参和出参的信息。

假设现在有一个查询用户信息的Repository接口及其实现：

```java
public interface UserRepository {
    // 这是一个通过name查询用户地址的方法
    String getAddress(String name);
}
public class UserRepositoryImpl implements UserRepository {
    // 这个就是原始类
    @Override
    public String getAddress(String name) {
        // 这里做一些查询数据库的逻辑
        String address = doGetAddressByNameFromDB(name);
        return address;
    }
}
public class App {
    public static void main() {
        // 基于接口编程的思想
        UserRepository userRepository = new UserRepositoryImpl();
        String userName = "caicai";
        String address = userRepository.getAddress(userName);
        System.out.println(address);
    }
}
```

当我们没有使用代理模式时，为了实现打印出入参信息的需求，我们一般会这么写：

```java
public class UserRepositoryImpl implements UserRepository {
    // 这个就是原始类
    @Override
    public String getAddress(String name) {
        // 打印入参
        System.out.println("UserRepository#getAddress, args: " + name);
        // 这里做一些查询数据库的逻辑
        String address = doGetAddressByNameFromDB(name);
        // 打印出参
        System.out.println(
            "UserRepository#getAddress, return value: " + address);
        return address;
    }
}
```

这个时候你会发现，日志这个功能跟业务逻辑是完全耦合在一起的，当你想要修改日志的时候，就必须去业务代码里修改，就比较麻烦。代理模式就可以解决这个问题。

```java
// 我们创建一个代理类，跟原始类实现同一个接口
public UserRepositoryImplProxy implements UserRepository {
    // 原始类的实现被代理，作为代理类的一个字段
    private UserRepository userRepository = new UserRepository();

    @Override
    public String getAddress(String name) {
        // 打印入参
        System.out.println("UserRepository#getAddress, args: " + name);
        // 调用被代理的对象的方法，这时候就实现了业务代码跟这个打日志的代码的解耦
        String address = userRepository.getAddress(userName);
        // 打印出参
        System.out.println(
            "UserRepository#getAddress, return value: " + address);
    }
}
// 使用的时候可以做一些修改，直接使用代理类
public class App {
    public static void main() {
        // 基于接口编程的思想，使用代理类对象替换原始类对象
        UserRepository userRepository = new UserRepositoryImplProxy();
        String userName = "caicai";
        String address = userRepository.getAddress(userName);
        System.out.println(address);
    }
}
```

基于接口的代理，其前提是必须存在一个接口。但在现实的应用场景中，可能并不存在接口，这时候就可以考虑使用继承的方式来实现代理模式。

### 1.2 基于继承实现的代理模式

同样还是上面那个例子，只不过这一次没有了接口：

```java
public class UserRepository {
    // 这个就是原始类，没有实现任何接口
    @Override
    public String getAddress(String name) {
        // 这里做一些查询数据库的逻辑
        String address = doGetAddressByNameFromDB(name);
        return address;
    }
}
public class App {
    public static void main() {
        UserRepository userRepository = new UserRepository();
        String userName = "caicai";
        String address = userRepository.getAddress(userName);
        System.out.println(address);
    }
}
```

我们通过继承来实现代理模式：

```java
public class UserRepositoryProxy extends UserRepository {
    // 代理类直接继承了原始类，通过覆写原始类方法的方式实现代理
    @Override
    public String getAddress(String name) {
        // 打印入参
        System.out.println("UserRepository#getAddress, args: " + name);
        // 调用被代理的对象的方法，实际上就是父类的方法
        // 这时候就实现了业务代码跟这个打日志的代码的解耦
        String address = super.getAddress(userName);
        // 打印出参
        System.out.println(
            "UserRepository#getAddress, return value: " + address);
    }
}
// 使用的时候可以做一些修改，直接使用代理类
public class App {
    public static void main() {
        // 基于接口编程的思想，使用代理类对象替换原始类对象
        UserRepository userRepository = new UserRepositoryProxy();
        String userName = "caicai";
        String address = userRepository.getAddress(userName);
        System.out.println(address);
    }
}
```

### 1.3 静态代理的缺点

上文中所述的代理模式都是静态代理，虽然比较容易理解，但我们会发现静态代理其实不够灵活。

一方面，我们需要在代理类中，将原始类的所有方法都重新实现一遍，并且为每个方法增加相似的代码逻辑。比如如果上文例子中的`UserRepository`有多个方法都需要打印日志的话，就是要在代理类里把每个方法都实现一遍然后加上日志的逻辑的。

另一方面，如果要添加的附加功能的类不止一个，我们就需要针对每个类都创建一个代理类。比如如果上文的例子中，又出现了一个`DepartmentRepository`需要加日志的功能，我们又得针对他再创建一个代理类，这在现实的项目中，会导致类成倍的增加，且每个代理类中的代码都是类似的，这就增加了很多不必要的开发成本。

动态代理就可以解决这些静态代理多带来的缺点。

## 二、动态代理

所谓的动态代理，就是我们不必事先为每个原始类编写代理类，而是在运行时，动态的创建原始类所对应的代理类，然后在系统运行时替换掉原始类。

比较常见的动态代理有两种：

- JDK自带的动态代理：基于Java的反射机制实现

- cglib、javassist等开源工具：基于修改字节码的机制实现动态代理

### 2.1 JDK动态代理

还是用上面的那个打日志的例子，不过这一次增加一个`DepartmentRepository`:

```java
public interface UserRepository {
    String getAddress(String name);
    // 增加了一个方法
    Integer getAge(String name);
}
public class UserRepositoryImpl implements UserRepository {
    @Override
    public String getAddress(String name) {
        return "杭州市";
    }
    @Override
    public Integer getAge(String name) {
        return 10;
    }
}
// 增加了一个接口DepartmentRepository
public interface DepartmentRepository {
    Integer getEmployeeCount(String name);
}

public class DepartmentRepositoryImpl implements DepartmentRepository {
    @Override
    public Integer getEmployeeCount(String name) {
        return 100;
    }
}
```

如果按照的静态代理的方式，我们需要做如下事情：

- 创建`UserRepositoryImpl`的代理类，然后实现两个方法，并且在两个方法中加上打印日志的逻辑

- 创建`DepartmentRepositoryImpl`的代理类，然后实现方法，并且在方法中加上打印日志的逻辑

这就非常的繁琐，并且打印日志的逻辑实际上是通用的，但是在静态代理里面无法复用。我们使用JDK动态代理的方式来解决这个问题：

```java
// 这个是一个为了使用JDK代理的工具类
public class JdkProxyHandler implements InvocationHandler {
    // 被代理的对象
    private final Object target;

    public JdkProxyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 只需要写这一个通用的方法即可，不需要给每一个原始类都创建代理类了，也不需要实现所有的方法
        System.out.printf("ClassName: %s, methodName: %s, args: %s%n",
                target.getClass().getSimpleName(), method.getName(), Arrays.toString(args));
        Object result = method.invoke(target, args);
        System.out.printf("ClassName: %s, methodName: %s, result: %s%n",
                target.getClass().getSimpleName(), method.getName(), String.valueOf(result));
        return result;
    }
}
// 使用的时候就非常的简单了
public class DynamicProxyApp {
    public static void main(String[] args) {

        UserRepository userRepository = (UserRepository) Proxy.newProxyInstance(
                DynamicProxyApp.class.getClassLoader(),
                // 这里就是被代理的接口，所以说JDK代理一定要有接口才能做
                new Class[]{UserRepository.class},
                // 这个就是被代理的对象
                new JdkProxyHandler(new UserRepositoryImpl()));

        userRepository.getAddress("caicai");
        userRepository.getAge("caicai");

        DepartmentRepository departmentRepository = (DepartmentRepository) Proxy.newProxyInstance(
                DynamicProxyApp.class.getClassLoader(),
                new Class[]{DepartmentRepository.class},
                new JdkProxyHandler(new DepartmentRepositoryImpl()));

        departmentRepository.getEmployeeCount("caicai");
    }
}
```

执行结果如下：

```
ClassName: UserRepositoryImpl, methodName: getAddress, args: [caicai]
ClassName: UserRepositoryImpl, methodName: getAddress, result: 杭州市

ClassName: UserRepositoryImpl, methodName: getAge, args: [caicai]
ClassName: UserRepositoryImpl, methodName: getAge, result: 10

ClassName: DepartmentRepositoryImpl, methodName: getEmployeeCount, args: [caicai]
ClassName: DepartmentRepositoryImpl, methodName: getEmployeeCount, result: 100
```

#### 2.1.1 为什么JDK的代理一定是基于接口实现的

在JDK动态代理的过程中，会生成对应的代理类(形如`$Proxy0.class`这种名字的类)，这些匿名类是继承了`java.lang.reflect.Proxy`类的，并且实现了我们所传入的接口。由于Java不支持多继承，当前的代理类为了能够替换掉原始类，所以就只能通过实现目标接口的方式来对原始类做扩展。

我们只需要看一下所生成的代理类就明白了：

```java
public class DynamicProxyApp {
    public static void main(String[] args) {
        // 加这一行，就能够把JDK生成的代理类保存在本地，在本工程的 jdk目录下
        // 使用Java17是jdk.proxy.ProxyGenerator.saveGeneratedFiles这个参数，
        // 如果使用其他Java版本，可以参考下 java.lang.reflect.ProxyGenerator这个类中的saveGeneratedFiles字段的值
        // 用它的值替换jdk.proxy.ProxyGenerator.saveGeneratedFiles就行
        System.getProperties().put("jdk.proxy.ProxyGenerator.saveGeneratedFiles","true");

        DepartmentRepository departmentRepository = (DepartmentRepository) Proxy.newProxyInstance(
                DynamicProxyApp.class.getClassLoader(),
                new Class[]{DepartmentRepository.class},
                new JdkProxyHandler(new DepartmentRepositoryImpl()));
        departmentRepository.getEmployeeCount("caicai");
    }
}
```

上面这一段代码执行后，我们就可以在工程目录下的`jdk`文件夹下找到JDK所生成的代理类：

```java
// 1. 从这边可以看到，这个代理类，继承了Proxy，实现了DepartmentRepository
// 由于Java只支持单继承，为了能够代理DepartmentRepositoryImpl，我们就只能让
// 这个代理类实现DepartmentRepository接口，也就是上面所提到的基于接口实现代理的方式
public final class $Proxy0 extends Proxy implements DepartmentRepository {
    private static final Method m0;
    private static final Method m1;
    private static final Method m2;
    private static final Method m3;

    public $Proxy0(InvocationHandler invocationHandler) {
        super(invocationHandler);
    }
    // 2. 这个就是我们所要被代理的方法
    public final Integer getEmployeeCount(String var1) {
        try {
            // 2.1 这个 super.h 就是我们使用时候传入的 invocationHandler 对象
            // 就是上文中 new JdkProxyHandler(new DepartmentRepositoryImpl())
            // 这个对象
            // 这里我们可以看到，基本上就是通过反射来调用我们在JdkProxyHandler中写的方法
            // 执行我们封装好的逻辑
            return (Integer)super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            // 3.静态代码块，用于通过反射来初始化四个方法属性
            // 我们关注的其实就是 m3 这个方法，其他的几个方法是Object自带
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.zju.caicai.common.proxy.DepartmentRepository").getMethod("getEmployeeCount", Class.forName("java.lang.String"));
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

### 2.2 基于字节码修改技术的动态代理

来看一下如何使用cglib实现动态代理，先把cglib的依赖加进来：

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.3.0</version>
</dependency>
```

然后用cglib做一下动态代理：

```java
public class OrderRepository {
    // 这个是要被代理的类，它并没有实现接口
    public Long getPrice(String orderId) {
        return 100L;
    }
}
public class LogInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object target, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        // 这里其实跟JdkProxyHandler的代理写法差不多，也是个工具类
        System.out.printf("ClassName: %s, methodName: %s, args: %s%n",
                target.getClass().getSimpleName(), method.getName(), Arrays.toString(args));
        // 这里是invokeSuper，即cglib是通过继承的方式来实现动态代理的
        Object result = methodProxy.invokeSuper(target, args);
        System.out.printf("ClassName: %s, methodName: %s, result: %s%n",
                target.getClass().getSimpleName(), method.getName(), String.valueOf(result));
        return result;
    }
}
// 调用的地方需要修改下
public class DynamicProxyApp {
    public static void main(String[] args) {
        // 加这一行，是为了把cglib生成的类保存下来，后面的参数是路径
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/caicai/Documents/Workspace/Codes/common/jdk/proxy1");
        Enhancer enhancer = new Enhancer();
        // superClass是原始类，这里也能看出cglib是用继承实现动态代理的
        enhancer.setSuperclass(OrderRepository.class);
        // 这个是代理类
        enhancer.setCallback(new LogInterceptor());
        // 创建来代理类对象以后，直接调用即可
        OrderRepository orderRepository = (OrderRepository) enhancer.create();
        orderRepository.getPrice("xxxx");
    }
}
```

由于cglib是基于继承实现的代理，并且java是不允许继承由final修饰的方法和类的，所以这种方式无法代理被final修饰的方法和类。

#### 2.2.1 cglib生成了3个代理类？

当我们运行完上面的代码之后，会发现，cglib居然一下子就给我们生成了3个类：

- `OrderRepository$$EnhancerByCGLIB$$b537f8d3`

- `OrderRepository$$EnhancerByCGLIB$$b537f8d3$$FastClassByCGLIB$$a878acf1`

- `OrderRepository$$FastClassByCGLIB$$ef66e34d` 



先来看第一个`OrderRepository$$EnhancerByCGLIB$$b537f8d3`：

```java
// 这个就是代理类，可以看到是继承了OrderRepository
// 本文中的是简化了之后的，真实的代理类比这个更复杂一些，多一些hash、equals等方法
// 而且变量名也更不可读，所以这里做了些优化，但逻辑是完全是一样
public class OrderRepository$$EnhancerByCGLIB$$b537f8d3 extends OrderRepository implements Factory {
    // 拦截器，就是所注册的LogInterceptory
    private MethodInterceptor logInterceptor;
    // 被代理的方法
    private static final Method getPrice;
    // 代理的方法
    private static final MethodProxy getPriceProxy;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        // 这个就是生成的代理类
        Class var0 = Class.forName("com.zju.caicai.common.proxy.OrderRepository$$EnhancerByCGLIB$$b537f8d3");
        // 这个是被代理类
        Class var1;
        getPrice = ReflectUtils.findMethods(new String[]{"getPrice", "(Ljava/lang/String;)Ljava/lang/Long;"}, (var1 = Class.forName("com.zju.caicai.common.proxy.OrderRepository")).getDeclaredMethods())[0];
        getPriceProxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)Ljava/lang/Long;", "getPrice", "CGLIB$getPrice$0");
    }

    final Long CGLIB$getPrice$0(String orderId) {
        // 这个是调用原来的没有被代理的方法
        return super.getPrice(orderId);
    }

    public final Long getPrice(String orderId) {
        这里就是调用被代理的方法
        MethodInterceptor interceptor = this.logInterceptor;
        return interceptor != null ? (Long)interceptor.intercept(this, getPrice, new Object[]{orderId}, getPriceProxy) : super.getPrice(orderId);
    }
}

```

**FastClass机制**

cglib多生成的几个类，实际上是它为了优化执行效率所设计的FastClass机制，FastClass机制就是对一个类的方法建立索引，通过索引来直接调用相应的方法。

看一个FastClass的例子，其实原理很简单：

```java
public class InventoryRepository {
    // 被代理类，有两个方法可以被代理，在JDK代理里面，一般都是通过反射执行的
    public Long getInventoryById(String id) {
        return 100L;
    }

    public Long getInventoryByName(String name) {
        return 120L;
    }
}
public class InventoryFastClass {
    // 这个就是被代理类的FaceClass
    public Object invoke(int index, Object obj, Object[] args) {
        InventoryRepository inventoryRepository = (InventoryRepository) obj;
        // 其实就是这里给InventoryRepository的每个方法都做了索引，
        // 这样就可以直接调用，而不需要用反射，这样执行效率就会更高
        switch (index) {
            case 1:
                return inventoryRepository.getInventoryById((String) args[0]);
            case 2:
                return inventoryRepository.getInventoryByName((String) args[0]);
        }
        return null;
    }

    public int getIndex(String name) {
        switch (name) {
            case "getInventoryById":
                return 1;
            case "getInventoryByName":
                return 2;
        }
        return -1;
    }
}
public class FastClassApplication {
    // 使用的时候就很简单了
    public static void main(String[] args) {
        InventoryRepository inventoryRepository = new InventoryRepository();
        InventoryFastClass inventoryFastClass = new InventoryFastClass();
        // 本质上就是对原始类的所有方法，都做了一个索引，以此来绕过反射，而是直接找到对应的方法执行
        int index = inventoryFastClass.getIndex("getInventoryById");
        inventoryFastClass.invoke(index, inventoryRepository, new Object[]{"caicai"});
    }

}

```







## 2.3 动态代理总结

最后我们总结一下JDK动态代理和Gglib动态代理的区别：

- JDK动态代理是实现了被代理对象的接口，Cglib是继承了被代理对象

- JDK和Cglib都是在运行期生成字节码，JDK是直接写Class字节码，Cglib使用ASM框架写Class字节码，Cglib代理实现更复杂，生成代理类比JDK效率低。

- JDK调用代理方法，是通过反射机制调用，Cglib是通过FastClass机制直接调用方法，Cglib执行效率更高。

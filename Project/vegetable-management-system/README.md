# Spring MVC

## 1. Controller层同时处理了页面跳转和API接口，这种混合模式有什么优缺点，如何把职责划分得更清晰？

首先需要分清页面跳转和API接口的代码。

```java
// 页面跳转，返回register.html页面
@GetMapping("/register")
public String registerPage() { return "register"; }
```

```java
// 返回JSON数据
@PostMapping("/register")
@ResponseBody
public Map<String, Object> apiRegister(@RequestBody Map<String, String> params) { ... }
```

采用混合模式开发比较简单。但职责不清晰，违背了单一职责原则。且业务庞大的话，控制器会很大，难以维护。

最好的优化方法是将两种方法分离，一个controller类包含页面跳转的方法，另一个类包含返回JSON数据的方法。

## 2. 分析一下Controller-Service-Repository在DDD设计中的角色定位，如何避免贫血模型的问题？

Controller负责与外部进行通信，只负责协议转换、参数校验和响应格式化，不负责业务逻辑。

Service有两种类型，一个是应用服务，用来写业务流程，不包含业务规则。另一个是领域服务，放公用的业务逻辑。

Repository用于数据的持久化和查询。

贫血模型是指对象只有数据，没有行为。如User实体类只有属性和getter、setter方法，没有行为，所有的业务逻辑都写在了service中。这样的话，对象的行为分散地装在service中，违反了面向对象思想，对象的作用也没发挥出来。

为了避免贫血模型，需要将service的业务逻辑放回对象中。

```java
// 贫血模型的service
@Transactional
public void addBalance(User user, BigDecimal amount) {
    if (amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new RuntimeException("充值金额必须大于0");
    }

    User currentUser = userRepository.findById(user.getId())
            .orElseThrow(() -> new RuntimeException("用户不存在"));

    currentUser.setBalance(currentUser.getBalance().add(amount));
    userRepository.save(currentUser);
}
```

```java
// 改进后的Service
class UserService {
    public User addBalance(User user, BigDecimal amount) {
        user.recharge(user.amount);
    }
}
```

```java
// 改进后的对象
@Entity
public class User {
    public void recharge(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("充值金额必须大于0");
        }
        this.balance = this.balance.add(amount);
    }
    
    public boolean canPurchase(BigDecimal totalAmount) {
        return this.balance.compareTo(totalAmount) >= 0;
    }
}
```

将业务逻辑移到实体类，实现充血模型，这样就将数据和行为集中到对象中，符合面向对象的思想。Service也不负责业务逻辑，只负责业务流程。

## 3. 模板引擎是什么，说一下服务端渲染和前后端分离的区别以及适用场景

模板引擎是将模板和数据转换成html的工具。

```html
<h1>
    Hello, ${name}
</h1>
```

```java
name = "Alice";
```

模板做的事情是解析`${name}`并替换为真实的数据。

使用的Thymeleaf就是当用户请求页面时，服务端获取数据Model，Thymeleaf将数据填入HTML，生成完整的html文件返回给浏览器。

使用服务端渲染，那么浏览器收到的就是渲染完毕的html文档，如果用户提交表单，会导致html刷新。如果使用前后端分离渲染，那么服务端值传递数据，浏览器获取的是html空壳和js代码，js在客户端请求数据并渲染界面。

## 4. Controller设计模式问题，Controller同时注入了多个Service，变成了胖控制器，可能会导致什么问题，如何通过门面模式来优化？

胖控制器如果要编写单元测试，需要Mock大量的Service，导致测试代码冗长。而且业务逻辑应该属于Service层，注入Service也属于业务逻辑的一部分，如果Controller调用多个Service，就导致Service变为简单的CRUD工具。而且一旦Service变更，可能会导致Controller注入的Service不适用，导致报错。

于是最好的方法是使用门面模式，在Service中创建领域服务，负责调用其他的Service。这样就将业务流程重新交给Service，Controller只负责接受参数、调用Service、返回响应。

## 5. 介绍DTO在MVC框架中的作用，如何设计通用的DTO转换来避免重复的代码？

实体类对象包含的是数据库数据的所有字段，但业务中有一些字段是不需要返回的，如例如密码。DTO可以对这些敏感字段进行屏蔽。

而且如果前端只需要获得用户名，和头像，但实体类包含大量字段，将这个用户名和头像属性封装起来制成DTO对象，就能只传输必要的字段，提高性能。

DTO也可以进行数据聚合。例如订单的DTO可能需要包含用户的名称，或者商品的id等。

设计通用的DTO转换可以使用Record。

```java
public record UserDTO(Long id, String name) {
    public static UserDTO from Entity(User user) {
        return new UserDTO(user.getId(), user.getName());
    }
}
```

## 6. 通过try-catch捕获RuntimeException并返回错误页面，这种方法有什么局限性，如何设计全局异常处理机制提高代码复用性？

项目中使用了手动try/catch来捕获逻辑。

```java
@PostMapping("/api/register")
@ResponseBody
public Map<String, Object> apiRegister(@RequestBody Map<String, String> params) {
    Map<String, Object> result = new HashMap<>();
    try {
        User user = new User();
        user.setUsername(params.get("username"));
        user.setPassword(params.get("password"));
        user.setEmail(params.get("email"));
        userService.registerUser(user);
        result.put("success", true);
        result.put("message", "注册成功");
    } catch (RuntimeException e) {
        result.put("success", false);
        result.put("message", e.getMessage());
    }
    return result;
}
```

这样每个Controller方法都要写相同的try/catch，如果接口比较多，写起来很繁琐。

其次，Controller的职责是路由和参数转换，不应该包含异常判断和错误跳转的业务逻辑。

如果对错误页面进行修改，需要修改每一个try/catch的地方。

因此，可以设计全局异常处理机制。首先定义全局的异常类。

```java
public class BusinessException extends RuntimeException {
    private final int errorCode;
    public BusinessException(int errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    // getters...
}
```

然后使用ControllerAdvice注解添加切片，能够捕获所有Controller抛出的异常。

```java
@ControllerAdvice // 如果是前后端分离，使用 @RestControllerAdvice
public class GlobalExceptionHandler {

    // 专门处理业务逻辑异常
    @ExceptionHandler(BusinessException.class)
    public String handleBusinessException(BusinessException e, Model model) {
        model.addAttribute("errorMessage", e.getMessage());
        return "error/business"; // 返回 Thymeleaf 模板路径
    }

    // 兜底：处理所有未知的运行异常
    @ExceptionHandler(RuntimeException.class)
    public String handleRuntimeException(RuntimeException e, Model model) {
        log.error("系统运行异常: ", e);
        model.addAttribute("errorMessage", "服务器开小差了，请稍后再试");
        return "error/500";
    }
}
```

这样，如果想要捕获新的异常，只需要在这个GlobalExceptionHandler中添加ExceptionHandler注解的异常类即可。

## 7. 高并发场景下，这个项目的MVC架构存在什么性能瓶颈，如何通过异步处理或者请求缓存来提高系统吞吐量？

当前系统的请求都是同步处理。

```java
@PostMapping("/api/chat")
public String chat(@RequestBody String message) {
    // 同步调用AI接口，阻塞线程
    return chat.chat(message);
}
```

这样，高并发场景下，请求只能一次性处理一个，导致性能低下。因此，可以使用异步处理来提高效率。

```java
@PostMapping("/api/chat")
@Async
public CompletableFuture<String> chatAsync(@RequestBody String message) {
    String response = chat.chat(message);
    return CompletableFuture.completedFuture(response);
}

// 配置异步线程池
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.initialize();
        return executor;
    }
}
```

配置了线程池，并发发送请求。

或者采用消息队列，Controller接收请求后写入MQ，直接向前端返回处理中，不需要阻塞等待。

如果是查询请求，可以设置缓存，像浏览器缓存、本地缓存或者使用Redis都可以保存热点数据。
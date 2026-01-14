# 第四章：Spring生态（30题）详解

## 4.1 Spring Core（10题）

### 96. Spring IOC的原理是什么？

```
┌─────────────────────────────────────────────────────────────────┐
│                     Spring IOC 原理                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  传统方式:                     IOC方式:                          │
│  ┌─────────┐                  ┌─────────┐                       │
│  │  对象A   │ ──new──>│对象B │  │  对象A   │                       │
│  └─────────┘                  └────┬────┘                       │
│       │                            │                            │
│       │ 主动创建                    │ 依赖注入                    │
│       ▼                            ▼                            │
│  ┌─────────┐               ┌──────────────┐                    │
│  │  对象B   │               │ IOC Container │                    │
│  └─────────┘               │   ┌───────┐    │                    │
│                            │   │ 对象B  │    │                    │
│  控制权在程序员              │   └───────┘    │                    │
│                            └──────────────┘                    │
│                            控制权在容器(控制反转)                  │
└─────────────────────────────────────────────────────────────────┘
```

```java
/**
 * IOC（控制反转）+ DI（依赖注入）原理
 */
public class IocPrincipleDemo {

    // ==================== 1. IOC核心概念 ====================

    /**
     * 传统方式：对象自己控制依赖的创建
     */
    public class TraditionalWay {
        public class UserService {
            // 紧耦合：自己创建依赖
            private UserDao userDao = new UserDaoImpl();

            public void save(User user) {
                userDao.save(user);
            }
        }
    }

    /**
     * IOC方式：容器控制依赖的创建和注入
     */
    @Service
    public class IocWay {
        // 松耦合：由容器注入依赖
        @Autowired
        private UserDao userDao;

        public void save(User user) {
            userDao.save(user);
        }
    }

    // ==================== 2. IOC容器实现原理 ====================

    /**
     * 简化版IOC容器实现
     */
    public class SimpleIocContainer {
        // 存储Bean定义
        private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();
        // 单例Bean缓存
        private Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

        /**
         * 加载Bean定义（从XML或注解）
         */
        public void loadBeanDefinitions(String configLocation) {
            // 1. 解析配置文件或扫描注解
            // 2. 创建BeanDefinition对象
            // 3. 注册到beanDefinitionMap
        }

        /**
         * 获取Bean实例
         */
        public Object getBean(String beanName) {
            // 1. 先从缓存获取
            Object singleton = singletonObjects.get(beanName);
            if (singleton != null) {
                return singleton;
            }

            // 2. 获取Bean定义
            BeanDefinition bd = beanDefinitionMap.get(beanName);

            // 3. 创建Bean实例
            Object bean = createBean(beanName, bd);

            // 4. 放入缓存
            singletonObjects.put(beanName, bean);

            return bean;
        }

        /**
         * 创建Bean实例
         */
        private Object createBean(String beanName, BeanDefinition bd) {
            try {
                // 1. 实例化
                Class<?> beanClass = bd.getBeanClass();
                Object instance = beanClass.getDeclaredConstructor().newInstance();

                // 2. 属性填充（依赖注入）
                populateBean(instance, bd);

                // 3. 初始化
                initializeBean(beanName, instance, bd);

                return instance;
            } catch (Exception e) {
                throw new RuntimeException("创建Bean失败: " + beanName, e);
            }
        }

        /**
         * 属性填充 - DI核心
         */
        private void populateBean(Object bean, BeanDefinition bd) {
            // 通过反射注入依赖
            Field[] fields = bean.getClass().getDeclaredFields();
            for (Field field : fields) {
                if (field.isAnnotationPresent(Autowired.class)) {
                    field.setAccessible(true);
                    // 递归获取依赖的Bean
                    Object dependency = getBean(field.getType().getName());
                    try {
                        field.set(bean, dependency);
                    } catch (IllegalAccessException e) {
                        throw new RuntimeException(e);
                    }
                }
            }
        }
    }

    // ==================== 3. BeanDefinition结构 ====================

    /**
     * Bean定义信息
     */
    public class BeanDefinition {
        private Class<?> beanClass;          // Bean类型
        private String scope;                 // 作用域
        private boolean lazyInit;             // 是否懒加载
        private String initMethodName;        // 初始化方法
        private String destroyMethodName;     // 销毁方法
        private PropertyValues propertyValues; // 属性值
        private ConstructorArgumentValues constructorArgumentValues; // 构造参数

        // getters and setters...
    }

    // ==================== 4. 依赖注入的三种方式 ====================

    @Service
    public class DependencyInjectionDemo {

        // 方式1：字段注入
        @Autowired
        private UserDao userDao;

        // 方式2：构造器注入（推荐）
        private final OrderDao orderDao;

        @Autowired
        public DependencyInjectionDemo(OrderDao orderDao) {
            this.orderDao = orderDao;
        }

        // 方式3：Setter注入
        private ProductDao productDao;

        @Autowired
        public void setProductDao(ProductDao productDao) {
            this.productDao = productDao;
        }
    }

    // ==================== 5. IOC容器启动流程 ====================

    /**
     * 容器启动流程示意
     */
    public void containerStartupProcess() {
        /*
         * 1. 资源定位
         *    - 定位配置文件或扫描包路径
         *
         * 2. BeanDefinition载入
         *    - 解析XML/注解，创建BeanDefinition
         *
         * 3. BeanDefinition注册
         *    - 注册到BeanDefinitionRegistry
         *
         * 4. Bean实例化
         *    - 根据BeanDefinition创建Bean实例
         *
         * 5. 依赖注入
         *    - 填充Bean属性
         *
         * 6. 初始化
         *    - 调用初始化方法
         */
    }
}
```

---

### 97. Bean的生命周期是怎样的？

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Bean 完整生命周期                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐                                                   │
│  │ 1. 实例化     │ ← 反射创建对象                                     │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ 2. 属性填充   │ ← 依赖注入                                         │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────────────────┐                                       │
│  │ 3. Aware接口回调          │ ← BeanNameAware, BeanFactoryAware...  │
│  └──────┬───────────────────┘                                       │
│         ▼                                                           │
│  ┌──────────────────────────┐                                       │
│  │ 4. BeanPostProcessor前置 │ ← postProcessBeforeInitialization     │
│  └──────┬───────────────────┘                                       │
│         ▼                                                           │
│  ┌──────────────────────────┐                                       │
│  │ 5. 初始化方法             │ ← @PostConstruct, InitializingBean    │
│  └──────┬───────────────────┘                                       │
│         ▼                                                           │
│  ┌──────────────────────────┐                                       │
│  │ 6. BeanPostProcessor后置 │ ← postProcessAfterInitialization      │
│  └──────┬───────────────────┘                                       │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ 7. 使用Bean   │ ← 业务使用                                        │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ 8. 销毁       │ ← @PreDestroy, DisposableBean                    │
│  └──────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

```java
/**
 * Bean生命周期完整示例
 */
@Component
public class BeanLifecycleDemo implements
        BeanNameAware,           // Aware接口
        BeanFactoryAware,
        ApplicationContextAware,
        InitializingBean,        // 初始化接口
        DisposableBean {         // 销毁接口

    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;

    // ==================== 1. 实例化 ====================

    public BeanLifecycleDemo() {
        System.out.println("1. 构造方法执行 - 实例化Bean");
    }

    // ==================== 2. 属性填充 ====================

    @Autowired
    private UserService userService;

    @Value("${app.name:demo}")
    private String appName;

    // 属性填充后
    public void afterPropertiesSet() throws Exception {
        System.out.println("2. 属性填充完成");
    }

    // ==================== 3. Aware接口回调 ====================

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("3.1 BeanNameAware - 设置Bean名称: " + name);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
        System.out.println("3.2 BeanFactoryAware - 设置BeanFactory");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
        System.out.println("3.3 ApplicationContextAware - 设置ApplicationContext");
    }

    // ==================== 5. 初始化方法 ====================

    // 方式1：@PostConstruct注解（最先执行）
    @PostConstruct
    public void postConstruct() {
        System.out.println("5.1 @PostConstruct - 初始化方法");
    }

    // 方式2：InitializingBean接口
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("5.2 InitializingBean.afterPropertiesSet()");
    }

    // 方式3：@Bean(initMethod = "customInit")
    public void customInit() {
        System.out.println("5.3 自定义init-method");
    }

    // ==================== 8. 销毁方法 ====================

    // 方式1：@PreDestroy注解（最先执行）
    @PreDestroy
    public void preDestroy() {
        System.out.println("8.1 @PreDestroy - 销毁前方法");
    }

    // 方式2：DisposableBean接口
    @Override
    public void destroy() throws Exception {
        System.out.println("8.2 DisposableBean.destroy()");
    }

    // 方式3：@Bean(destroyMethod = "customDestroy")
    public void customDestroy() {
        System.out.println("8.3 自定义destroy-method");
    }
}

/**
 * BeanPostProcessor - 对所有Bean生效
 */
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    // ==================== 4. 初始化前置处理 ====================
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof BeanLifecycleDemo) {
            System.out.println("4. BeanPostProcessor.postProcessBeforeInitialization - " + beanName);
        }
        return bean;
    }

    // ==================== 6. 初始化后置处理 ====================
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof BeanLifecycleDemo) {
            System.out.println("6. BeanPostProcessor.postProcessAfterInitialization - " + beanName);
            // 这里可以返回代理对象（AOP的实现点）
        }
        return bean;
    }
}

/**
 * InstantiationAwareBeanPostProcessor - 更细粒度的控制
 */
@Component
public class CustomInstantiationAwareBeanPostProcessor
        implements InstantiationAwareBeanPostProcessor {

    // 实例化前
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        System.out.println("0. 实例化前 - " + beanName);
        return null; // 返回null表示继续正常流程
    }

    // 实例化后
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) {
        System.out.println("1.5 实例化后 - " + beanName);
        return true; // 返回true继续属性填充
    }

    // 属性填充前
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        System.out.println("2. 属性填充 - " + beanName);
        return pvs;
    }
}

/**
 * 完整生命周期输出顺序
 */
/*
0. 实例化前 - beanLifecycleDemo
1. 构造方法执行 - 实例化Bean
1.5 实例化后 - beanLifecycleDemo
2. 属性填充 - beanLifecycleDemo
3.1 BeanNameAware - 设置Bean名称: beanLifecycleDemo
3.2 BeanFactoryAware - 设置BeanFactory
3.3 ApplicationContextAware - 设置ApplicationContext
4. BeanPostProcessor.postProcessBeforeInitialization
5.1 @PostConstruct - 初始化方法
5.2 InitializingBean.afterPropertiesSet()
5.3 自定义init-method
6. BeanPostProcessor.postProcessAfterInitialization
7. Bean使用中...
8.1 @PreDestroy - 销毁前方法
8.2 DisposableBean.destroy()
8.3 自定义destroy-method
*/
```

---

### 98. Bean的作用域有哪些？

```java
/**
 * Spring Bean作用域详解
 */
public class BeanScopeDemo {

    // ==================== 1. singleton（默认） ====================

    /**
     * 单例作用域：整个IoC容器中只有一个实例
     */
    @Component
    @Scope("singleton")  // 默认可省略
    public class SingletonBean {
        public SingletonBean() {
            System.out.println("SingletonBean 创建");
        }
    }

    // 测试
    @Test
    public void testSingleton() {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        SingletonBean bean1 = ctx.getBean(SingletonBean.class);
        SingletonBean bean2 = ctx.getBean(SingletonBean.class);
        System.out.println(bean1 == bean2); // true - 同一个实例
    }

    // ==================== 2. prototype ====================

    /**
     * 原型作用域：每次获取都创建新实例
     */
    @Component
    @Scope("prototype")
    public class PrototypeBean {
        public PrototypeBean() {
            System.out.println("PrototypeBean 创建");
        }
    }

    // 测试
    @Test
    public void testPrototype() {
        PrototypeBean bean1 = ctx.getBean(PrototypeBean.class);
        PrototypeBean bean2 = ctx.getBean(PrototypeBean.class);
        System.out.println(bean1 == bean2); // false - 不同实例
    }

    // ==================== 3. request（Web环境） ====================

    /**
     * 请求作用域：每个HTTP请求创建一个实例
     */
    @Component
    @Scope(value = WebApplicationContext.SCOPE_REQUEST,
           proxyMode = ScopedProxyMode.TARGET_CLASS)
    public class RequestScopedBean {
        private String requestId = UUID.randomUUID().toString();

        public String getRequestId() {
            return requestId;
        }
    }

    // ==================== 4. session（Web环境） ====================

    /**
     * 会话作用域：每个HTTP Session创建一个实例
     */
    @Component
    @Scope(value = WebApplicationContext.SCOPE_SESSION,
           proxyMode = ScopedProxyMode.TARGET_CLASS)
    public class SessionScopedBean {
        private Map<String, Object> sessionData = new HashMap<>();

        public void setAttribute(String key, Object value) {
            sessionData.put(key, value);
        }

        public Object getAttribute(String key) {
            return sessionData.get(key);
        }
    }

    // ==================== 5. application（Web环境） ====================

    /**
     * 应用作用域：整个ServletContext生命周期只有一个实例
     */
    @Component
    @Scope(value = WebApplicationContext.SCOPE_APPLICATION)
    public class ApplicationScopedBean {
        private AtomicInteger visitCount = new AtomicInteger(0);

        public int incrementAndGet() {
            return visitCount.incrementAndGet();
        }
    }

    // ==================== 6. websocket ====================

    /**
     * WebSocket作用域：每个WebSocket会话一个实例
     */
    @Component
    @Scope(value = "websocket", proxyMode = ScopedProxyMode.TARGET_CLASS)
    public class WebSocketScopedBean {
        private String connectionId;
    }

    // ==================== 7. 自定义作用域 ====================

    /**
     * 自定义线程作用域
     */
    public class ThreadScope implements Scope {
        private final ThreadLocal<Map<String, Object>> threadLocal =
            ThreadLocal.withInitial(HashMap::new);

        @Override
        public Object get(String name, ObjectFactory<?> objectFactory) {
            Map<String, Object> scope = threadLocal.get();
            return scope.computeIfAbsent(name, k -> objectFactory.getObject());
        }

        @Override
        public Object remove(String name) {
            return threadLocal.get().remove(name);
        }

        @Override
        public void registerDestructionCallback(String name, Runnable callback) {
            // 注册销毁回调
        }

        @Override
        public Object resolveContextualObject(String key) {
            return null;
        }

        @Override
        public String getConversationId() {
            return String.valueOf(Thread.currentThread().getId());
        }
    }

    // 注册自定义作用域
    @Configuration
    public class ScopeConfig implements BeanFactoryPostProcessor {
        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
            beanFactory.registerScope("thread", new ThreadScope());
        }
    }

    // 使用自定义作用域
    @Component
    @Scope("thread")
    public class ThreadScopedBean {
        private String threadName = Thread.currentThread().getName();
    }

    // ==================== 作用域对比表 ====================
    /*
    ┌────────────┬───────────────────────────────────────────────────────┐
    │  作用域     │  说明                                                 │
    ├────────────┼───────────────────────────────────────────────────────┤
    │ singleton  │ 默认，IoC容器中只有一个实例                             │
    │ prototype  │ 每次getBean()都创建新实例                              │
    │ request    │ 每个HTTP请求一个实例（Web环境）                         │
    │ session    │ 每个HTTP Session一个实例（Web环境）                     │
    │ application│ 每个ServletContext一个实例（Web环境）                   │
    │ websocket  │ 每个WebSocket会话一个实例                              │
    └────────────┴───────────────────────────────────────────────────────┘
    */

    // ==================== Singleton注入Prototype问题 ====================

    /**
     * 问题：Singleton中注入Prototype，Prototype不会每次新建
     */
    @Component
    public class SingletonService {
        @Autowired
        private PrototypeBean prototypeBean; // 只会注入一次！

        public void doSomething() {
            // prototypeBean永远是同一个实例
        }
    }

    /**
     * 解决方案1：使用@Lookup
     */
    @Component
    public abstract class SingletonWithLookup {
        @Lookup
        public abstract PrototypeBean getPrototypeBean();

        public void doSomething() {
            PrototypeBean bean = getPrototypeBean(); // 每次都是新实例
        }
    }

    /**
     * 解决方案2：使用ObjectFactory
     */
    @Component
    public class SingletonWithObjectFactory {
        @Autowired
        private ObjectFactory<PrototypeBean> prototypeBeanFactory;

        public void doSomething() {
            PrototypeBean bean = prototypeBeanFactory.getObject(); // 每次新实例
        }
    }

    /**
     * 解决方案3：使用代理
     */
    @Component
    @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
    public class ProxiedPrototypeBean {
        // 通过代理每次获取新实例
    }
}
```

---

### 99. Spring AOP的实现原理？

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Spring AOP 原理                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐           │
│  │   Client    │ ──> │    Proxy    │ ──> │   Target    │           │
│  └─────────────┘     └──────┬──────┘     └─────────────┘           │
│                             │                                       │
│                      ┌──────┴──────┐                                │
│                      │             │                                │
│                ┌─────▼─────┐ ┌─────▼─────┐                         │
│                │  Before   │ │  After    │                         │
│                │  Advice   │ │  Advice   │                         │
│                └───────────┘ └───────────┘                         │
│                                                                      │
│  代理方式选择:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 目标类实现接口? ──Yes──> JDK动态代理                           │   │
│  │       │                                                      │   │
│  │      No                                                      │   │
│  │       │                                                      │   │
│  │       └──────────────> CGLIB代理                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

```java
/**
 * Spring AOP 实现原理详解
 */
public class SpringAopPrincipleDemo {

    // ==================== 1. AOP核心概念 ====================

    /*
     * 切面(Aspect)：横切关注点的模块化，如日志、事务
     * 连接点(JoinPoint)：程序执行的某个特定位置，如方法调用
     * 切点(Pointcut)：匹配连接点的表达式
     * 通知(Advice)：切面在切点处执行的动作
     * 目标对象(Target)：被代理的原始对象
     * 代理(Proxy)：AOP创建的代理对象
     * 织入(Weaving)：将切面应用到目标对象的过程
     */

    // ==================== 2. AOP注解使用示例 ====================

    @Aspect
    @Component
    public class LogAspect {

        // 定义切点
        @Pointcut("execution(* com.example.service.*.*(..))")
        public void servicePointcut() {}

        // 前置通知
        @Before("servicePointcut()")
        public void before(JoinPoint joinPoint) {
            String methodName = joinPoint.getSignature().getName();
            Object[] args = joinPoint.getArgs();
            System.out.println("方法执行前: " + methodName);
        }

        // 后置通知
        @After("servicePointcut()")
        public void after(JoinPoint joinPoint) {
            System.out.println("方法执行后（无论是否异常）");
        }

        // 返回通知
        @AfterReturning(pointcut = "servicePointcut()", returning = "result")
        public void afterReturning(JoinPoint joinPoint, Object result) {
            System.out.println("方法正常返回: " + result);
        }

        // 异常通知
        @AfterThrowing(pointcut = "servicePointcut()", throwing = "ex")
        public void afterThrowing(JoinPoint joinPoint, Exception ex) {
            System.out.println("方法抛出异常: " + ex.getMessage());
        }

        // 环绕通知（最强大）
        @Around("servicePointcut()")
        public Object around(ProceedingJoinPoint pjp) throws Throwable {
            long start = System.currentTimeMillis();
            try {
                System.out.println("环绕前");
                Object result = pjp.proceed(); // 执行目标方法
                System.out.println("环绕后");
                return result;
            } finally {
                long cost = System.currentTimeMillis() - start;
                System.out.println("方法耗时: " + cost + "ms");
            }
        }
    }

    // ==================== 3. JDK动态代理实现 ====================

    /**
     * JDK动态代理 - 基于接口
     */
    public class JdkDynamicProxyDemo {

        // 接口
        public interface UserService {
            void save(User user);
            User getById(Long id);
        }

        // 实现类
        public class UserServiceImpl implements UserService {
            @Override
            public void save(User user) {
                System.out.println("保存用户: " + user);
            }

            @Override
            public User getById(Long id) {
                return new User(id, "test");
            }
        }

        // JDK代理处理器
        public class JdkProxyHandler implements InvocationHandler {
            private final Object target;

            public JdkProxyHandler(Object target) {
                this.target = target;
            }

            @Override
            public Object invoke(Object proxy, Method method, Object[] args)
                    throws Throwable {
                // 前置增强
                System.out.println("Before: " + method.getName());

                // 调用目标方法
                Object result = method.invoke(target, args);

                // 后置增强
                System.out.println("After: " + method.getName());

                return result;
            }
        }

        // 创建代理
        public UserService createProxy(UserService target) {
            return (UserService) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new JdkProxyHandler(target)
            );
        }
    }

    // ==================== 4. CGLIB代理实现 ====================

    /**
     * CGLIB代理 - 基于继承
     */
    public class CglibProxyDemo {

        // 目标类（无需实现接口）
        public class OrderService {
            public void createOrder(Order order) {
                System.out.println("创建订单: " + order);
            }
        }

        // CGLIB方法拦截器
        public class CglibMethodInterceptor implements MethodInterceptor {
            @Override
            public Object intercept(Object obj, Method method, Object[] args,
                    MethodProxy proxy) throws Throwable {
                // 前置增强
                System.out.println("CGLIB Before: " + method.getName());

                // 调用目标方法（使用MethodProxy更快）
                Object result = proxy.invokeSuper(obj, args);

                // 后置增强
                System.out.println("CGLIB After: " + method.getName());

                return result;
            }
        }

        // 创建CGLIB代理
        public OrderService createProxy() {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OrderService.class);
            enhancer.setCallback(new CglibMethodInterceptor());
            return (OrderService) enhancer.create();
        }
    }

    // ==================== 5. Spring AOP代理创建流程 ====================

    /**
     * Spring AOP自动代理创建器
     * AbstractAutoProxyCreator是核心
     */
    public class AopProxyCreationProcess {

        /*
         * 代理创建时机：
         * BeanPostProcessor.postProcessAfterInitialization()
         */

        // 简化的代理创建逻辑
        public Object createProxy(Object bean, String beanName) {
            // 1. 获取所有Advisor（切面）
            List<Advisor> advisors = findEligibleAdvisors(bean.getClass());

            if (advisors.isEmpty()) {
                return bean; // 无需代理
            }

            // 2. 创建代理工厂
            ProxyFactory proxyFactory = new ProxyFactory();
            proxyFactory.setTarget(bean);
            proxyFactory.addAdvisors(advisors);

            // 3. 决定代理方式
            if (shouldProxyTargetClass(bean.getClass())) {
                proxyFactory.setProxyTargetClass(true); // CGLIB
            } else {
                proxyFactory.setInterfaces(bean.getClass().getInterfaces()); // JDK
            }

            // 4. 创建并返回代理对象
            return proxyFactory.getProxy();
        }

        private boolean shouldProxyTargetClass(Class<?> beanClass) {
            // 没有实现接口，使用CGLIB
            return beanClass.getInterfaces().length == 0;
        }
    }

    // ==================== 6. Pointcut表达式详解 ====================

    @Aspect
    @Component
    public class PointcutExpressionDemo {

        // execution表达式
        @Pointcut("execution(public * com.example.service.*.*(..))")
        // 格式: execution(修饰符? 返回类型 类路径?方法名(参数) 异常?)
        public void executionPointcut() {}

        // within表达式 - 匹配类
        @Pointcut("within(com.example.service.*)")
        public void withinPointcut() {}

        // annotation表达式 - 匹配注解
        @Pointcut("@annotation(com.example.annotation.Loggable)")
        public void annotationPointcut() {}

        // bean表达式 - 匹配Bean名称
        @Pointcut("bean(*Service)")
        public void beanPointcut() {}

        // args表达式 - 匹配参数类型
        @Pointcut("args(java.lang.String, ..)")
        public void argsPointcut() {}

        // 组合表达式
        @Pointcut("executionPointcut() && !withinPointcut()")
        public void combinedPointcut() {}
    }
}
```

---

### 100. JDK动态代理和CGLIB的区别？

```java
/**
 * JDK动态代理 vs CGLIB 对比
 */
public class ProxyComparisonDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────┐
    │                  JDK动态代理 vs CGLIB 对比                           │
    ├──────────────┬────────────────────────┬─────────────────────────────┤
    │     特性     │     JDK动态代理         │         CGLIB              │
    ├──────────────┼────────────────────────┼─────────────────────────────┤
    │ 实现方式     │ 基于接口                │ 基于继承（生成子类）         │
    │ 代理对象     │ 实现相同接口的代理类    │ 目标类的子类                 │
    │ 是否需要接口 │ 必须有接口              │ 不需要接口                   │
    │ final方法   │ 不影响                  │ 无法代理final方法            │
    │ final类     │ 不影响                  │ 无法代理final类              │
    │ 性能        │ 创建快，调用稍慢         │ 创建慢，调用快               │
    │ 依赖        │ JDK自带                 │ 需要cglib库                  │
    │ 底层实现    │ 反射                    │ ASM字节码生成                │
    └──────────────┴────────────────────────┴─────────────────────────────┘
    */

    // ==================== 1. JDK动态代理完整示例 ====================

    // 接口
    public interface PaymentService {
        boolean pay(String orderId, BigDecimal amount);
        void refund(String orderId);
    }

    // 实现类
    public class PaymentServiceImpl implements PaymentService {
        @Override
        public boolean pay(String orderId, BigDecimal amount) {
            System.out.println("支付订单: " + orderId + ", 金额: " + amount);
            return true;
        }

        @Override
        public void refund(String orderId) {
            System.out.println("退款订单: " + orderId);
        }
    }

    // JDK代理处理器
    public class JdkInvocationHandler implements InvocationHandler {
        private final Object target;

        public JdkInvocationHandler(Object target) {
            this.target = target;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            long start = System.currentTimeMillis();
            System.out.println("JDK代理 - 方法开始: " + method.getName());

            try {
                // 调用目标方法
                Object result = method.invoke(target, args);
                System.out.println("JDK代理 - 方法成功");
                return result;
            } catch (Exception e) {
                System.out.println("JDK代理 - 方法异常: " + e.getMessage());
                throw e;
            } finally {
                long cost = System.currentTimeMillis() - start;
                System.out.println("JDK代理 - 方法耗时: " + cost + "ms");
            }
        }
    }

    // JDK代理工厂
    public class JdkProxyFactory {
        @SuppressWarnings("unchecked")
        public static <T> T createProxy(T target) {
            return (T) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new JdkInvocationHandler(target)
            );
        }
    }

    // ==================== 2. CGLIB代理完整示例 ====================

    // 无需接口的目标类
    public class NotificationService {
        public void sendEmail(String to, String content) {
            System.out.println("发送邮件到: " + to);
        }

        public void sendSms(String phone, String content) {
            System.out.println("发送短信到: " + phone);
        }

        // final方法无法被代理
        public final void sendPush(String userId, String content) {
            System.out.println("发送推送到: " + userId);
        }
    }

    // CGLIB方法拦截器
    public class CglibMethodInterceptor implements MethodInterceptor {
        @Override
        public Object intercept(Object obj, Method method, Object[] args,
                MethodProxy methodProxy) throws Throwable {
            long start = System.currentTimeMillis();
            System.out.println("CGLIB代理 - 方法开始: " + method.getName());

            try {
                // 调用父类方法（目标方法）
                Object result = methodProxy.invokeSuper(obj, args);
                System.out.println("CGLIB代理 - 方法成功");
                return result;
            } catch (Exception e) {
                System.out.println("CGLIB代理 - 方法异常: " + e.getMessage());
                throw e;
            } finally {
                long cost = System.currentTimeMillis() - start;
                System.out.println("CGLIB代理 - 方法耗时: " + cost + "ms");
            }
        }
    }

    // CGLIB代理工厂
    public class CglibProxyFactory {
        @SuppressWarnings("unchecked")
        public static <T> T createProxy(Class<T> targetClass) {
            Enhancer enhancer = new Enhancer();
            // 设置父类
            enhancer.setSuperclass(targetClass);
            // 设置回调
            enhancer.setCallback(new CglibMethodInterceptor());
            // 创建代理
            return (T) enhancer.create();
        }

        // 带构造参数的代理
        public static <T> T createProxy(Class<T> targetClass,
                Class<?>[] argTypes, Object[] args) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(targetClass);
            enhancer.setCallback(new CglibMethodInterceptor());
            return (T) enhancer.create(argTypes, args);
        }
    }

    // ==================== 3. 性能测试对比 ====================

    public class ProxyPerformanceTest {

        public static void main(String[] args) {
            int iterations = 1000000;

            // 测试JDK代理
            PaymentService jdkProxy = JdkProxyFactory.createProxy(
                new PaymentServiceImpl());
            long jdkStart = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                jdkProxy.pay("order" + i, BigDecimal.ONE);
            }
            long jdkCost = System.nanoTime() - jdkStart;

            // 测试CGLIB代理
            NotificationService cglibProxy = CglibProxyFactory.createProxy(
                NotificationService.class);
            long cglibStart = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                cglibProxy.sendEmail("test@test.com", "content" + i);
            }
            long cglibCost = System.nanoTime() - cglibStart;

            System.out.println("JDK代理耗时: " + jdkCost / 1000000 + "ms");
            System.out.println("CGLIB代理耗时: " + cglibCost / 1000000 + "ms");
        }
    }

    // ==================== 4. Spring中的代理选择 ====================

    @Configuration
    @EnableAspectJAutoProxy(proxyTargetClass = false) // false=优先JDK代理
    public class AopConfig {
        /*
         * proxyTargetClass = false（默认）:
         *   - 有接口：使用JDK动态代理
         *   - 无接口：使用CGLIB代理
         *
         * proxyTargetClass = true:
         *   - 统一使用CGLIB代理
         */
    }

    // ==================== 5. 代理的局限性 ====================

    @Service
    public class SelfInvocationDemo {

        @Transactional
        public void methodA() {
            // 内部调用，不会经过代理！事务不生效
            methodB();
        }

        @Transactional(propagation = Propagation.REQUIRES_NEW)
        public void methodB() {
            // ...
        }

        // 解决方案：获取代理对象
        @Autowired
        private ApplicationContext context;

        public void methodAFixed() {
            // 通过代理调用
            SelfInvocationDemo proxy = context.getBean(SelfInvocationDemo.class);
            proxy.methodB();
        }

        // 或者使用AopContext
        public void methodAFixed2() {
            // 需要开启exposeProxy
            SelfInvocationDemo proxy = (SelfInvocationDemo) AopContext.currentProxy();
            proxy.methodB();
        }
    }

    // 开启exposeProxy
    @EnableAspectJAutoProxy(exposeProxy = true)
    public class ExposeProxyConfig {}
}
```

---

### 101. Spring事务的传播行为有哪些？

```java
/**
 * Spring事务传播行为详解
 */
public class TransactionPropagationDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                          事务传播行为                                        │
    ├──────────────────┬──────────────────────────────────────────────────────────┤
    │    传播行为       │                     说明                                  │
    ├──────────────────┼──────────────────────────────────────────────────────────┤
    │ REQUIRED         │ 默认。有事务加入，没有则新建                                │
    │ REQUIRES_NEW     │ 总是新建事务，挂起当前事务                                  │
    │ SUPPORTS         │ 有事务加入，没有就以非事务方式执行                           │
    │ NOT_SUPPORTED    │ 以非事务方式执行，挂起当前事务                               │
    │ MANDATORY        │ 必须有事务，没有则抛异常                                    │
    │ NEVER            │ 必须无事务，有则抛异常                                      │
    │ NESTED           │ 有事务则嵌套执行（savepoint），没有则新建                    │
    └──────────────────┴──────────────────────────────────────────────────────────┘
    */

    @Service
    public class OrderService {

        @Autowired
        private OrderRepository orderRepository;
        @Autowired
        private InventoryService inventoryService;
        @Autowired
        private PaymentService paymentService;
        @Autowired
        private LogService logService;

        // ==================== 1. REQUIRED（默认） ====================

        /**
         * 有事务就加入，没有就新建
         */
        @Transactional(propagation = Propagation.REQUIRED)
        public void createOrder(Order order) {
            // 保存订单
            orderRepository.save(order);

            // 调用库存服务（会加入当前事务）
            inventoryService.deductStock(order.getProductId(), order.getQuantity());

            // 如果这里异常，订单和库存都会回滚
        }

        // ==================== 2. REQUIRES_NEW ====================

        /**
         * 总是新建事务，当前事务被挂起
         */
        @Transactional(propagation = Propagation.REQUIRED)
        public void createOrderWithLog(Order order) {
            // 保存订单
            orderRepository.save(order);

            // 记录日志（独立事务，不受订单事务影响）
            logService.log("创建订单: " + order.getId());

            // 即使订单事务回滚，日志也已提交
            if (order.getAmount().compareTo(BigDecimal.ZERO) < 0) {
                throw new RuntimeException("金额不能为负");
            }
        }
    }

    @Service
    public class LogService {
        @Autowired
        private LogRepository logRepository;

        @Transactional(propagation = Propagation.REQUIRES_NEW)
        public void log(String message) {
            // 新事务，与调用方事务独立
            logRepository.save(new Log(message));
        }
    }

    // ==================== 3. SUPPORTS ====================

    @Service
    public class QueryService {

        /**
         * 有事务就加入，没有就非事务执行
         * 适用于查询操作
         */
        @Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
        public User getUser(Long id) {
            return userRepository.findById(id).orElse(null);
        }
    }

    // ==================== 4. NOT_SUPPORTED ====================

    @Service
    public class ReportService {

        /**
         * 以非事务方式执行，挂起当前事务
         * 适用于不需要事务的操作
         */
        @Transactional(propagation = Propagation.NOT_SUPPORTED)
        public void generateReport() {
            // 生成报表，可能耗时较长，不需要事务
        }
    }

    // ==================== 5. MANDATORY ====================

    @Service
    public class InventoryService {

        /**
         * 必须在事务中运行，否则抛异常
         * 强制调用方提供事务
         */
        @Transactional(propagation = Propagation.MANDATORY)
        public void deductStock(Long productId, int quantity) {
            // 必须有事务
            // 如果没有事务调用此方法，抛出IllegalTransactionStateException
        }
    }

    // ==================== 6. NEVER ====================

    @Service
    public class CacheService {

        /**
         * 不能在事务中运行，否则抛异常
         */
        @Transactional(propagation = Propagation.NEVER)
        public void refreshCache() {
            // 不能有事务
            // 如果有事务调用此方法，抛出IllegalTransactionStateException
        }
    }

    // ==================== 7. NESTED ====================

    @Service
    public class PaymentService {

        /**
         * 嵌套事务：有事务则创建savepoint，回滚只到savepoint
         * 没有事务则新建
         */
        @Transactional(propagation = Propagation.NESTED)
        public void processPayment(Payment payment) {
            // 保存支付记录
            paymentRepository.save(payment);

            // 如果异常，只回滚支付操作，不影响外层事务
        }
    }

    // ==================== 综合示例 ====================

    @Service
    public class OrderProcessService {

        @Autowired
        private OrderService orderService;
        @Autowired
        private PaymentService paymentService;
        @Autowired
        private NotificationService notificationService;

        @Transactional
        public void processOrder(Order order, Payment payment) {
            try {
                // 1. 创建订单（REQUIRED - 加入当前事务）
                orderService.createOrder(order);

                // 2. 处理支付（NESTED - 嵌套事务）
                try {
                    paymentService.processPayment(payment);
                } catch (PaymentException e) {
                    // 支付失败，但订单可以保留为待支付状态
                    order.setStatus(OrderStatus.PENDING_PAYMENT);
                }

                // 3. 发送通知（REQUIRES_NEW - 独立事务）
                notificationService.sendOrderNotification(order);

            } catch (Exception e) {
                // 订单创建失败，全部回滚
                throw new OrderProcessException("订单处理失败", e);
            }
        }
    }

    // ==================== 传播行为图解 ====================

    /*
     * REQUIRED:
     * ┌─────────────────────────────────────┐
     * │ Transaction A                       │
     * │  ┌─────────────────────────────┐   │
     * │  │ Method A                    │   │
     * │  │  ┌───────────────────────┐  │   │
     * │  │  │ Method B (加入A的事务) │  │   │
     * │  │  └───────────────────────┘  │   │
     * │  └─────────────────────────────┘   │
     * └─────────────────────────────────────┘
     *
     * REQUIRES_NEW:
     * ┌─────────────────────────────────────┐
     * │ Transaction A (挂起)                │
     * │  ┌─────────────────────────────┐   │
     * │  │ Method A                    │   │
     * └──┼─────────────────────────────┼───┘
     *    │                             │
     * ┌──┼─────────────────────────────┼───┐
     * │  │ Transaction B (新建)        │   │
     * │  │  ┌───────────────────────┐  │   │
     * │  │  │ Method B              │  │   │
     * │  │  └───────────────────────┘  │   │
     * │  └─────────────────────────────┘   │
     * └─────────────────────────────────────┘
     *
     * NESTED:
     * ┌─────────────────────────────────────┐
     * │ Transaction A                       │
     * │  ┌─────────────────────────────┐   │
     * │  │ Method A                    │   │
     * │  │  ┌───────────────────────┐  │   │
     * │  │  │ Savepoint             │  │   │
     * │  │  │ Method B (嵌套事务)   │  │   │
     * │  │  └───────────────────────┘  │   │
     * │  └─────────────────────────────┘   │
     * └─────────────────────────────────────┘
     */
}
```

---

### 102. Spring事务失效的场景有哪些？

```java
/**
 * Spring事务失效的常见场景
 */
@Service
public class TransactionFailureDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────┐
    │                      事务失效场景汇总                                │
    ├─────┬───────────────────────────────────────────────────────────────┤
    │  1  │ 方法不是public的                                              │
    │  2  │ 自调用（this调用）                                            │
    │  3  │ 异常被catch吞掉                                               │
    │  4  │ 抛出的异常类型不对（默认只回滚RuntimeException）               │
    │  5  │ 数据库不支持事务（如MyISAM）                                   │
    │  6  │ Bean没有被Spring管理                                          │
    │  7  │ 传播行为设置不当                                              │
    │  8  │ 多线程调用                                                    │
    └─────┴───────────────────────────────────────────────────────────────┘
    */

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private ApplicationContext applicationContext;

    // ==================== 1. 方法不是public ====================

    /**
     * ❌ 失效：非public方法，Spring AOP无法代理
     */
    @Transactional
    private void nonPublicMethod() {
        userRepository.save(new User("test"));
        throw new RuntimeException("异常");
        // 事务不会回滚！
    }

    /**
     * ❌ 失效：protected方法
     */
    @Transactional
    protected void protectedMethod() {
        // 事务可能失效
    }

    /**
     * ✅ 正确：public方法
     */
    @Transactional
    public void publicMethod() {
        // 事务正常工作
    }

    // ==================== 2. 自调用（最常见） ====================

    /**
     * ❌ 失效：内部方法调用，不经过代理
     */
    public void outerMethod() {
        // this调用，不是代理对象调用
        this.innerMethod(); // 事务失效！
    }

    @Transactional
    public void innerMethod() {
        userRepository.save(new User("test"));
        throw new RuntimeException("异常");
        // 不会回滚！
    }

    /**
     * ✅ 解决方案1：注入自身
     */
    @Autowired
    @Lazy
    private TransactionFailureDemo self;

    public void outerMethodFixed1() {
        self.innerMethod(); // 通过代理调用
    }

    /**
     * ✅ 解决方案2：从ApplicationContext获取
     */
    public void outerMethodFixed2() {
        TransactionFailureDemo proxy = applicationContext.getBean(TransactionFailureDemo.class);
        proxy.innerMethod();
    }

    /**
     * ✅ 解决方案3：使用AopContext
     */
    public void outerMethodFixed3() {
        // 需要配置 @EnableAspectJAutoProxy(exposeProxy = true)
        TransactionFailureDemo proxy = (TransactionFailureDemo) AopContext.currentProxy();
        proxy.innerMethod();
    }

    // ==================== 3. 异常被catch吞掉 ====================

    /**
     * ❌ 失效：异常被吞掉，Spring感知不到
     */
    @Transactional
    public void exceptionSwallowed() {
        try {
            userRepository.save(new User("test"));
            throw new RuntimeException("异常");
        } catch (Exception e) {
            // 异常被吞掉，Spring不知道发生了异常
            log.error("异常", e);
        }
        // 事务不会回滚！
    }

    /**
     * ✅ 解决方案1：重新抛出异常
     */
    @Transactional
    public void exceptionRethrow() {
        try {
            userRepository.save(new User("test"));
            throw new RuntimeException("异常");
        } catch (Exception e) {
            log.error("异常", e);
            throw e; // 重新抛出
        }
    }

    /**
     * ✅ 解决方案2：手动回滚
     */
    @Transactional
    public void manualRollback() {
        try {
            userRepository.save(new User("test"));
            throw new RuntimeException("异常");
        } catch (Exception e) {
            log.error("异常", e);
            // 手动标记回滚
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        }
    }

    // ==================== 4. 异常类型不匹配 ====================

    /**
     * ❌ 失效：默认只回滚RuntimeException和Error
     */
    @Transactional
    public void checkedExceptionNotRollback() throws Exception {
        userRepository.save(new User("test"));
        throw new IOException("IO异常"); // 检查异常，不回滚！
    }

    /**
     * ✅ 解决方案：指定回滚异常类型
     */
    @Transactional(rollbackFor = Exception.class)
    public void rollbackForAllException() throws Exception {
        userRepository.save(new User("test"));
        throw new IOException("IO异常"); // 会回滚
    }

    /**
     * 指定不回滚的异常
     */
    @Transactional(noRollbackFor = BusinessException.class)
    public void noRollbackForBusinessException() {
        // BusinessException不会导致回滚
    }

    // ==================== 5. 数据库引擎不支持事务 ====================

    /**
     * ❌ 失效：MySQL的MyISAM引擎不支持事务
     * 需要使用InnoDB引擎
     */

    // ==================== 6. Bean没有被Spring管理 ====================

    /**
     * ❌ 失效：手动new的对象，不被Spring管理
     */
    public void notManagedBean() {
        TransactionFailureDemo demo = new TransactionFailureDemo();
        demo.publicMethod(); // 事务失效！
    }

    // ==================== 7. 传播行为设置不当 ====================

    /**
     * ❌ 可能失效：NOT_SUPPORTED会挂起事务
     */
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void notSupported() {
        // 以非事务方式执行
        userRepository.save(new User("test"));
        throw new RuntimeException("异常");
        // 不会回滚！因为没有事务
    }

    /**
     * ❌ 可能失效：NEVER会抛异常
     */
    @Transactional(propagation = Propagation.NEVER)
    public void neverPropagation() {
        // 如果有事务环境，直接抛异常
    }

    // ==================== 8. 多线程调用 ====================

    /**
     * ❌ 失效：新线程不在同一事务中
     */
    @Transactional
    public void multiThread() {
        userRepository.save(new User("main"));

        new Thread(() -> {
            // 新线程，不在主线程事务中
            userRepository.save(new User("child"));
            throw new RuntimeException("子线程异常");
            // 只会回滚子线程的操作（如果有事务的话）
        }).start();

        // 主线程继续执行
    }

    /**
     * ❌ 使用@Async也会失效
     */
    @Transactional
    public void asyncMethod() {
        userRepository.save(new User("sync"));
        asyncService.doAsync(); // 异步方法在新线程执行
    }

    // ==================== 检查清单 ====================

    /*
     * 事务失效排查清单：
     *
     * 1. □ 方法是否是public？
     * 2. □ 是否是自调用（this调用）？
     * 3. □ 异常是否被catch吞掉了？
     * 4. □ 异常类型是否正确？是否需要rollbackFor=Exception.class？
     * 5. □ 数据库引擎是否支持事务？（InnoDB）
     * 6. □ Bean是否被Spring管理？（是否有@Service等注解）
     * 7. □ 传播行为是否正确？
     * 8. □ 是否涉及多线程/异步？
     * 9. □ 是否配置了事务管理器？
     * 10. □ 是否开启了事务注解支持？@EnableTransactionManagement
     */
}
```

---

### 103. @Transactional的原理？

```java
/**
 * @Transactional 原理详解
 */
public class TransactionalPrincipleDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                    @Transactional 工作原理                               │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  1. 解析注解                                                             │
    │     ┌─────────────┐                                                     │
    │     │ @Transactional │ ──> TransactionAttributeSource                   │
    │     └─────────────┘                                                     │
    │                                                                          │
    │  2. 创建代理                                                             │
    │     ┌─────────────────┐      ┌──────────────────────┐                  │
    │     │  Target Bean     │ ──> │  Proxy Bean          │                  │
    │     └─────────────────┘      │  (含TransactionInterceptor)             │
    │                              └──────────────────────┘                  │
    │                                                                          │
    │  3. 执行流程                                                             │
    │     ┌────────────────────────────────────────────────────────────┐     │
    │     │ TransactionInterceptor.invoke()                            │     │
    │     │     │                                                      │     │
    │     │     ├── 1. 获取事务属性                                     │     │
    │     │     ├── 2. 获取事务管理器                                   │     │
    │     │     ├── 3. 开启事务                                        │     │
    │     │     ├── 4. 执行目标方法                                     │     │
    │     │     ├── 5. 正常：提交事务                                   │     │
    │     │     └── 6. 异常：回滚事务                                   │     │
    │     └────────────────────────────────────────────────────────────┘     │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. @Transactional 注解属性 ====================

    @Target({ElementType.TYPE, ElementType.METHOD})
    @Retention(RetentionPolicy.RUNTIME)
    @Inherited
    @Documented
    public @interface Transactional {

        // 事务管理器
        @AliasFor("transactionManager")
        String value() default "";

        // 传播行为
        Propagation propagation() default Propagation.REQUIRED;

        // 隔离级别
        Isolation isolation() default Isolation.DEFAULT;

        // 超时时间（秒）
        int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

        // 是否只读
        boolean readOnly() default false;

        // 回滚的异常类型
        Class<? extends Throwable>[] rollbackFor() default {};

        // 不回滚的异常类型
        Class<? extends Throwable>[] noRollbackFor() default {};
    }

    // ==================== 2. 事务拦截器核心逻辑 ====================

    /**
     * TransactionInterceptor 简化版实现
     */
    public class SimpleTransactionInterceptor implements MethodInterceptor {

        private PlatformTransactionManager transactionManager;
        private TransactionAttributeSource transactionAttributeSource;

        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            // 1. 获取目标方法
            Method method = invocation.getMethod();
            Class<?> targetClass = invocation.getThis().getClass();

            // 2. 获取事务属性
            TransactionAttribute txAttr = transactionAttributeSource
                .getTransactionAttribute(method, targetClass);

            if (txAttr == null) {
                // 无事务注解，直接执行
                return invocation.proceed();
            }

            // 3. 获取事务管理器
            PlatformTransactionManager tm = determineTransactionManager(txAttr);

            // 4. 开启事务
            TransactionStatus status = tm.getTransaction(txAttr);

            Object result;
            try {
                // 5. 执行目标方法
                result = invocation.proceed();

                // 6. 提交事务
                tm.commit(status);
            } catch (Throwable ex) {
                // 7. 判断是否需要回滚
                if (txAttr.rollbackOn(ex)) {
                    tm.rollback(status);
                } else {
                    tm.commit(status);
                }
                throw ex;
            }

            return result;
        }
    }

    // ==================== 3. 事务管理器实现 ====================

    /**
     * 事务管理器核心实现
     */
    public class DataSourceTransactionManager implements PlatformTransactionManager {

        private DataSource dataSource;

        @Override
        public TransactionStatus getTransaction(TransactionDefinition definition) {
            // 1. 获取数据库连接
            Connection conn = DataSourceUtils.getConnection(dataSource);

            // 2. 关闭自动提交
            conn.setAutoCommit(false);

            // 3. 设置隔离级别
            if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
                conn.setTransactionIsolation(definition.getIsolationLevel());
            }

            // 4. 绑定到当前线程
            TransactionSynchronizationManager.bindResource(dataSource, conn);

            // 5. 返回事务状态
            return new DefaultTransactionStatus(conn, true, true);
        }

        @Override
        public void commit(TransactionStatus status) {
            Connection conn = (Connection) status.getTransaction();
            try {
                conn.commit();
            } finally {
                cleanup(conn);
            }
        }

        @Override
        public void rollback(TransactionStatus status) {
            Connection conn = (Connection) status.getTransaction();
            try {
                conn.rollback();
            } finally {
                cleanup(conn);
            }
        }

        private void cleanup(Connection conn) {
            // 恢复自动提交
            conn.setAutoCommit(true);
            // 解绑连接
            TransactionSynchronizationManager.unbindResource(dataSource);
            // 释放连接
            DataSourceUtils.releaseConnection(conn, dataSource);
        }
    }

    // ==================== 4. 事务同步管理器 ====================

    /**
     * 使用ThreadLocal保存事务状态
     */
    public class TransactionSynchronizationManager {

        // 资源绑定（如数据库连接）
        private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<>("Transactional resources");

        // 事务同步器
        private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
            new NamedThreadLocal<>("Transaction synchronizations");

        // 当前事务名称
        private static final ThreadLocal<String> currentTransactionName =
            new NamedThreadLocal<>("Current transaction name");

        // 是否只读
        private static final ThreadLocal<Boolean> currentTransactionReadOnly =
            new NamedThreadLocal<>("Current transaction read-only status");

        // 隔离级别
        private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
            new NamedThreadLocal<>("Current transaction isolation level");

        // 是否激活
        private static final ThreadLocal<Boolean> actualTransactionActive =
            new NamedThreadLocal<>("Actual transaction active");

        public static void bindResource(Object key, Object value) {
            Map<Object, Object> map = resources.get();
            if (map == null) {
                map = new HashMap<>();
                resources.set(map);
            }
            map.put(key, value);
        }

        public static Object getResource(Object key) {
            Map<Object, Object> map = resources.get();
            return map != null ? map.get(key) : null;
        }
    }

    // ==================== 5. 编程式事务 ====================

    @Service
    public class ProgrammaticTransactionDemo {

        @Autowired
        private TransactionTemplate transactionTemplate;

        @Autowired
        private PlatformTransactionManager transactionManager;

        /**
         * 使用TransactionTemplate
         */
        public void withTransactionTemplate() {
            transactionTemplate.execute(status -> {
                // 业务逻辑
                userRepository.save(new User("test"));

                // 回滚
                // status.setRollbackOnly();

                return null;
            });
        }

        /**
         * 使用TransactionManager
         */
        public void withTransactionManager() {
            DefaultTransactionDefinition def = new DefaultTransactionDefinition();
            def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

            TransactionStatus status = transactionManager.getTransaction(def);
            try {
                // 业务逻辑
                userRepository.save(new User("test"));

                transactionManager.commit(status);
            } catch (Exception e) {
                transactionManager.rollback(status);
                throw e;
            }
        }
    }

    // ==================== 6. 事务配置 ====================

    @Configuration
    @EnableTransactionManagement // 开启事务注解支持
    public class TransactionConfig {

        @Bean
        public PlatformTransactionManager transactionManager(DataSource dataSource) {
            return new DataSourceTransactionManager(dataSource);
        }

        @Bean
        public TransactionTemplate transactionTemplate(
                PlatformTransactionManager transactionManager) {
            return new TransactionTemplate(transactionManager);
        }
    }
}
```

---

### 104. BeanFactory和ApplicationContext的区别？

```java
/**
 * BeanFactory vs ApplicationContext 对比
 */
public class BeanFactoryVsApplicationContextDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                BeanFactory vs ApplicationContext                         │
    ├──────────────────┬─────────────────────────┬─────────────────────────────┤
    │      特性         │      BeanFactory        │    ApplicationContext       │
    ├──────────────────┼─────────────────────────┼─────────────────────────────┤
    │ 定位              │ IoC基础容器              │ 高级容器（继承BeanFactory）  │
    │ Bean加载          │ 懒加载                  │ 预加载（启动时实例化）        │
    │ 国际化            │ ❌ 不支持               │ ✅ MessageSource            │
    │ 事件发布          │ ❌ 不支持               │ ✅ ApplicationEventPublisher│
    │ 资源访问          │ ❌ 不支持               │ ✅ ResourceLoader           │
    │ AOP支持           │ 手动配置                │ 自动代理                     │
    │ 环境抽象          │ ❌ 不支持               │ ✅ Environment              │
    │ 注解支持          │ 有限                    │ 完整支持                     │
    │ 适用场景          │ 资源受限环境            │ 企业应用                     │
    └──────────────────┴─────────────────────────┴─────────────────────────────┘
    */

    // ==================== 1. 继承关系 ====================

    /*
                            BeanFactory
                                 │
                    ┌────────────┴────────────┐
                    │                         │
            ListableBeanFactory      HierarchicalBeanFactory
                    │                         │
                    └────────────┬────────────┘
                                 │
                    ConfigurableListableBeanFactory
                                 │
                         ApplicationContext
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
    ConfigurableApplicationContext  WebApplicationContext
              │
    ┌─────────┴─────────┐
    │                   │
    AnnotationConfigApplicationContext  ClassPathXmlApplicationContext
    */

    // ==================== 2. BeanFactory 使用示例 ====================

    public void beanFactoryDemo() {
        // 使用DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

        // 注册Bean定义
        BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(UserService.class);
        beanFactory.registerBeanDefinition("userService", builder.getBeanDefinition());

        // 获取Bean（懒加载，此时才创建）
        UserService userService = beanFactory.getBean(UserService.class);

        // BeanFactory基本功能
        boolean contains = beanFactory.containsBean("userService");
        boolean singleton = beanFactory.isSingleton("userService");
        Class<?> type = beanFactory.getType("userService");
    }

    // ==================== 3. ApplicationContext 使用示例 ====================

    public void applicationContextDemo() {
        // 基于注解的ApplicationContext
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

        // 基于XML的ApplicationContext
        // ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

        // 1. 获取Bean
        UserService userService = context.getBean(UserService.class);

        // 2. 国际化
        String message = context.getMessage("greeting", new Object[]{"World"}, Locale.CHINA);

        // 3. 发布事件
        context.publishEvent(new CustomEvent(this, "test"));

        // 4. 获取资源
        Resource resource = context.getResource("classpath:config.properties");

        // 5. 获取环境
        Environment env = context.getEnvironment();
        String property = env.getProperty("app.name");
    }

    // ==================== 4. 懒加载 vs 预加载 ====================

    @Component
    public class LazyLoadingDemo {

        /**
         * BeanFactory：默认懒加载
         * - 优点：启动快，节省资源
         * - 缺点：运行时才发现Bean配置问题
         */
        public void beanFactoryLazyLoading() {
            DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
            // ... 注册Bean定义
            // Bean在getBean时才创建
        }

        /**
         * ApplicationContext：默认预加载单例Bean
         * - 优点：启动时发现问题，运行时快
         * - 缺点：启动慢，占用内存
         */
        public void applicationContextEagerLoading() {
            // 启动时就创建所有单例Bean
            ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        }

        /**
         * ApplicationContext可以配置懒加载
         */
        @Lazy
        @Component
        public class LazyBean {
            // 这个Bean会懒加载
        }
    }

    // ==================== 5. 事件发布功能 ====================

    // 自定义事件
    public class OrderCreatedEvent extends ApplicationEvent {
        private final Order order;

        public OrderCreatedEvent(Object source, Order order) {
            super(source);
            this.order = order;
        }

        public Order getOrder() {
            return order;
        }
    }

    // 事件发布者
    @Service
    public class OrderService {
        @Autowired
        private ApplicationEventPublisher eventPublisher;

        public void createOrder(Order order) {
            // 保存订单
            orderRepository.save(order);
            // 发布事件
            eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        }
    }

    // 事件监听者
    @Component
    public class OrderEventListener {

        @EventListener
        public void handleOrderCreated(OrderCreatedEvent event) {
            // 处理订单创建事件
            sendNotification(event.getOrder());
        }
    }

    // ==================== 6. 国际化功能 ====================

    @Configuration
    public class I18nConfig {

        @Bean
        public MessageSource messageSource() {
            ResourceBundleMessageSource source = new ResourceBundleMessageSource();
            source.setBasename("messages");
            source.setDefaultEncoding("UTF-8");
            return source;
        }
    }

    // messages_zh_CN.properties: greeting=你好, {0}
    // messages_en.properties: greeting=Hello, {0}

    @Service
    public class I18nService {
        @Autowired
        private MessageSource messageSource;

        public String greet(String name, Locale locale) {
            return messageSource.getMessage("greeting", new Object[]{name}, locale);
        }
    }

    // ==================== 7. 资源访问功能 ====================

    @Service
    public class ResourceService {
        @Autowired
        private ResourceLoader resourceLoader;

        public void loadResources() throws IOException {
            // 类路径资源
            Resource classpath = resourceLoader.getResource("classpath:config.yml");

            // 文件系统资源
            Resource file = resourceLoader.getResource("file:/path/to/file.txt");

            // URL资源
            Resource url = resourceLoader.getResource("https://example.com/data.json");

            // 读取资源内容
            try (InputStream is = classpath.getInputStream()) {
                String content = StreamUtils.copyToString(is, StandardCharsets.UTF_8);
            }
        }
    }

    // ==================== 8. 环境抽象功能 ====================

    @Service
    public class EnvironmentService {
        @Autowired
        private Environment environment;

        public void useEnvironment() {
            // 获取属性
            String appName = environment.getProperty("app.name", "default");

            // 获取必需属性
            String dbUrl = environment.getRequiredProperty("database.url");

            // 获取带类型的属性
            Integer port = environment.getProperty("server.port", Integer.class, 8080);

            // 检查Profile
            boolean isProd = environment.acceptsProfiles(Profiles.of("prod"));
        }
    }
}
```

---

### 105. Spring中的设计模式有哪些？

```java
/**
 * Spring中常用的设计模式
 */
public class SpringDesignPatternsDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                     Spring 中的设计模式                                  │
    ├───────────────────┬─────────────────────────────────────────────────────┤
    │    设计模式        │                   应用场景                          │
    ├───────────────────┼─────────────────────────────────────────────────────┤
    │ 单例模式          │ Bean默认作用域、ApplicationContext                   │
    │ 工厂模式          │ BeanFactory、FactoryBean                            │
    │ 代理模式          │ AOP、@Transactional、@Async                         │
    │ 模板方法模式      │ JdbcTemplate、RestTemplate、TransactionTemplate     │
    │ 观察者模式        │ ApplicationEvent、ApplicationListener               │
    │ 策略模式          │ Resource接口、InstantiationStrategy                 │
    │ 适配器模式        │ HandlerAdapter、AdvisorAdapter                      │
    │ 装饰器模式        │ BeanWrapper、HttpServletRequestWrapper              │
    │ 责任链模式        │ Filter链、Interceptor链                             │
    │ 建造者模式        │ BeanDefinitionBuilder、UriComponentsBuilder        │
    └───────────────────┴─────────────────────────────────────────────────────┘
    */

    // ==================== 1. 单例模式 ====================

    /**
     * Spring IoC容器中Bean默认是单例的
     */
    @Component
    @Scope("singleton") // 默认
    public class SingletonBean {
        // 整个容器中只有一个实例
    }

    // Spring单例实现（简化版）
    public class DefaultSingletonBeanRegistry {
        // 一级缓存：存放完全初始化的单例
        private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

        public Object getSingleton(String beanName) {
            return singletonObjects.get(beanName);
        }

        protected void addSingleton(String beanName, Object singletonObject) {
            singletonObjects.put(beanName, singletonObject);
        }
    }

    // ==================== 2. 工厂模式 ====================

    /**
     * BeanFactory - 简单工厂/工厂方法
     */
    public void beanFactoryDemo() {
        BeanFactory factory = new DefaultListableBeanFactory();
        Object bean = factory.getBean("beanName");
    }

    /**
     * FactoryBean - 工厂Bean，创建复杂对象
     */
    @Component
    public class ConnectionFactoryBean implements FactoryBean<Connection> {

        @Override
        public Connection getObject() throws Exception {
            // 创建复杂对象
            return DriverManager.getConnection("jdbc:mysql://localhost:3306/test");
        }

        @Override
        public Class<?> getObjectType() {
            return Connection.class;
        }

        @Override
        public boolean isSingleton() {
            return true;
        }
    }

    // 使用
    // getBean("connectionFactoryBean") 返回Connection对象
    // getBean("&connectionFactoryBean") 返回FactoryBean本身

    // ==================== 3. 代理模式 ====================

    /**
     * AOP使用代理模式
     */
    @Aspect
    @Component
    public class LoggingAspect {

        @Around("execution(* com.example.service.*.*(..))")
        public Object log(ProceedingJoinPoint pjp) throws Throwable {
            // 代理逻辑
            System.out.println("Before");
            Object result = pjp.proceed();
            System.out.println("After");
            return result;
        }
    }

    // JDK动态代理
    public class JdkProxy implements InvocationHandler {
        private Object target;

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 增强逻辑
            return method.invoke(target, args);
        }
    }

    // ==================== 4. 模板方法模式 ====================

    /**
     * JdbcTemplate - 模板方法模式
     */
    @Service
    public class UserDao {
        @Autowired
        private JdbcTemplate jdbcTemplate;

        public User findById(Long id) {
            // 模板方法，只需关注SQL和映射
            return jdbcTemplate.queryForObject(
                "SELECT * FROM user WHERE id = ?",
                new Object[]{id},
                (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name"))
            );
        }
    }

    // 模板方法抽象类
    public abstract class JdbcTemplate {

        // 模板方法
        public <T> T execute(StatementCallback<T> action) {
            // 获取连接
            Connection conn = getConnection();
            try {
                Statement stmt = conn.createStatement();
                return action.doInStatement(stmt); // 回调
            } finally {
                releaseConnection(conn);
            }
        }

        // 抽象方法由子类或回调实现
        protected abstract Connection getConnection();
        protected abstract void releaseConnection(Connection conn);
    }

    // ==================== 5. 观察者模式 ====================

    /**
     * ApplicationEvent - 事件
     */
    public class UserRegisteredEvent extends ApplicationEvent {
        private final User user;

        public UserRegisteredEvent(Object source, User user) {
            super(source);
            this.user = user;
        }
    }

    /**
     * 事件发布者
     */
    @Service
    public class UserRegistrationService {
        @Autowired
        private ApplicationEventPublisher publisher;

        public void register(User user) {
            // 保存用户
            userRepository.save(user);
            // 发布事件
            publisher.publishEvent(new UserRegisteredEvent(this, user));
        }
    }

    /**
     * 事件监听者（观察者）
     */
    @Component
    public class EmailNotificationListener {

        @EventListener
        public void onUserRegistered(UserRegisteredEvent event) {
            // 发送欢迎邮件
            emailService.sendWelcomeEmail(event.getUser());
        }
    }

    @Component
    public class PointsListener {

        @EventListener
        public void onUserRegistered(UserRegisteredEvent event) {
            // 赠送积分
            pointsService.grantPoints(event.getUser(), 100);
        }
    }

    // ==================== 6. 策略模式 ====================

    /**
     * Resource - 策略模式
     * 不同的资源访问策略
     */
    public interface Resource {
        InputStream getInputStream() throws IOException;
        boolean exists();
        String getFilename();
    }

    // 具体策略
    public class ClassPathResource implements Resource { /* ... */ }
    public class FileSystemResource implements Resource { /* ... */ }
    public class UrlResource implements Resource { /* ... */ }

    // 策略选择
    @Service
    public class ResourceLoaderService {

        public Resource getResource(String location) {
            if (location.startsWith("classpath:")) {
                return new ClassPathResource(location);
            } else if (location.startsWith("file:")) {
                return new FileSystemResource(location);
            } else if (location.startsWith("http:")) {
                return new UrlResource(location);
            }
            return new ClassPathResource(location);
        }
    }

    // ==================== 7. 适配器模式 ====================

    /**
     * HandlerAdapter - 适配不同的Controller
     */
    public interface HandlerAdapter {
        boolean supports(Object handler);
        ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                          Object handler) throws Exception;
    }

    // @Controller注解的适配器
    public class RequestMappingHandlerAdapter implements HandlerAdapter {
        @Override
        public boolean supports(Object handler) {
            return handler instanceof HandlerMethod;
        }

        @Override
        public ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                                  Object handler) {
            // 适配@Controller方法调用
            return null;
        }
    }

    // ==================== 8. 装饰器模式 ====================

    /**
     * BeanWrapper - 装饰器模式
     */
    public interface BeanWrapper {
        void setPropertyValue(String propertyName, Object value);
        Object getPropertyValue(String propertyName);
    }

    public class BeanWrapperImpl implements BeanWrapper {
        private Object wrappedObject;

        public BeanWrapperImpl(Object object) {
            this.wrappedObject = object;
        }

        @Override
        public void setPropertyValue(String propertyName, Object value) {
            // 通过反射设置属性
        }
    }

    // ==================== 9. 责任链模式 ====================

    /**
     * Filter链 - 责任链模式
     */
    public interface Filter {
        void doFilter(ServletRequest request, ServletResponse response,
                     FilterChain chain) throws IOException, ServletException;
    }

    @Component
    public class AuthenticationFilter implements Filter {
        @Override
        public void doFilter(ServletRequest request, ServletResponse response,
                           FilterChain chain) throws IOException, ServletException {
            // 认证逻辑
            if (isAuthenticated(request)) {
                chain.doFilter(request, response); // 传递给下一个
            } else {
                throw new AuthenticationException("未认证");
            }
        }
    }

    // ==================== 10. 建造者模式 ====================

    /**
     * BeanDefinitionBuilder - 建造者模式
     */
    public void builderPatternDemo() {
        BeanDefinition bd = BeanDefinitionBuilder
            .genericBeanDefinition(UserService.class)
            .setScope("singleton")
            .setLazyInit(true)
            .addPropertyReference("userDao", "userDao")
            .addPropertyValue("name", "test")
            .getBeanDefinition();
    }

    // UriComponentsBuilder
    public void uriBuilderDemo() {
        URI uri = UriComponentsBuilder
            .fromUriString("https://api.example.com")
            .path("/users/{id}")
            .queryParam("name", "test")
            .buildAndExpand(123)
            .toUri();
    }
}
```

---

## 4.2 Spring Boot（8题）

### 106. Spring Boot自动配置的原理？

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Spring Boot 自动配置原理                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  @SpringBootApplication                                                  │
│         │                                                                │
│         ├── @SpringBootConfiguration                                     │
│         │         └── @Configuration                                     │
│         │                                                                │
│         ├── @ComponentScan                                               │
│         │         └── 扫描当前包及子包                                    │
│         │                                                                │
│         └── @EnableAutoConfiguration  ◄── 自动配置核心                    │
│                   │                                                      │
│                   └── @Import(AutoConfigurationImportSelector.class)     │
│                             │                                            │
│                             ▼                                            │
│                   加载 META-INF/spring.factories                         │
│                   或 META-INF/spring/...AutoConfiguration.imports        │
│                             │                                            │
│                             ▼                                            │
│                   过滤条件注解 @Conditional                               │
│                             │                                            │
│                             ▼                                            │
│                   注册符合条件的Bean                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
/**
 * Spring Boot 自动配置原理详解
 */
public class AutoConfigurationPrincipleDemo {

    // ==================== 1. @EnableAutoConfiguration 核心 ====================

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import(AutoConfigurationImportSelector.class) // 核心
    public @interface EnableAutoConfiguration {
        String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
        Class<?>[] exclude() default {};
        String[] excludeName() default {};
    }

    // ==================== 2. AutoConfigurationImportSelector ====================

    /**
     * 自动配置导入选择器（简化版）
     */
    public class AutoConfigurationImportSelector implements DeferredImportSelector {

        @Override
        public String[] selectImports(AnnotationMetadata metadata) {
            // 1. 检查是否启用自动配置
            if (!isEnabled(metadata)) {
                return new String[0];
            }

            // 2. 加载自动配置类
            List<String> configurations = getCandidateConfigurations(metadata);

            // 3. 去重
            configurations = removeDuplicates(configurations);

            // 4. 排除指定的配置类
            Set<String> exclusions = getExclusions(metadata);
            configurations.removeAll(exclusions);

            // 5. 过滤（根据@Conditional条件）
            configurations = filter(configurations);

            return configurations.toArray(new String[0]);
        }

        /**
         * 从spring.factories或新的imports文件加载配置类
         */
        protected List<String> getCandidateConfigurations(AnnotationMetadata metadata) {
            // Spring Boot 2.7之前：META-INF/spring.factories
            // Spring Boot 2.7+：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

            List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
                EnableAutoConfiguration.class,
                getClass().getClassLoader()
            );
            return configurations;
        }
    }

    // ==================== 3. spring.factories 文件示例 ====================

    /*
     * META-INF/spring.factories 内容：
     *
     * org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
     * org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
     * org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
     * org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
     * org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
     * ...
     */

    // ==================== 4. 条件注解详解 ====================

    /**
     * 常用条件注解
     */
    public class ConditionalAnnotationsDemo {

        // 类存在时生效
        @ConditionalOnClass(DataSource.class)

        // 类不存在时生效
        @ConditionalOnMissingClass("com.example.SomeClass")

        // Bean存在时生效
        @ConditionalOnBean(DataSource.class)

        // Bean不存在时生效
        @ConditionalOnMissingBean(DataSource.class)

        // 配置属性满足条件时生效
        @ConditionalOnProperty(prefix = "spring.datasource", name = "enabled", havingValue = "true")

        // Web环境生效
        @ConditionalOnWebApplication

        // 非Web环境生效
        @ConditionalOnNotWebApplication

        // 表达式满足时生效
        @ConditionalOnExpression("${feature.enabled:true}")

        // 资源存在时生效
        @ConditionalOnResource(resources = "classpath:config.properties")
    }

    // ==================== 5. 自动配置类示例 ====================

    /**
     * DataSource自动配置（简化版）
     */
    @Configuration
    @ConditionalOnClass(DataSource.class) // DataSource类存在
    @ConditionalOnMissingBean(DataSource.class) // 用户没有自定义DataSource
    @EnableConfigurationProperties(DataSourceProperties.class) // 启用配置属性
    @Import({ DataSourcePoolMetadataProvidersConfiguration.class })
    public class DataSourceAutoConfiguration {

        @Configuration
        @ConditionalOnMissingBean(DataSource.class)
        @ConditionalOnProperty(prefix = "spring.datasource", name = "type")
        static class Generic {
            @Bean
            public DataSource dataSource(DataSourceProperties properties) {
                return properties.initializeDataSourceBuilder().build();
            }
        }

        @Configuration
        @ConditionalOnClass(HikariDataSource.class)
        @ConditionalOnMissingBean(DataSource.class)
        @ConditionalOnProperty(name = "spring.datasource.type",
                             havingValue = "com.zaxxer.hikari.HikariDataSource",
                             matchIfMissing = true)
        static class Hikari {
            @Bean
            @ConfigurationProperties(prefix = "spring.datasource.hikari")
            public HikariDataSource dataSource(DataSourceProperties properties) {
                HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
                return dataSource;
            }
        }
    }

    /**
     * 配置属性类
     */
    @ConfigurationProperties(prefix = "spring.datasource")
    public class DataSourceProperties {
        private String driverClassName;
        private String url;
        private String username;
        private String password;
        private String type;

        // getters and setters...
    }

    // ==================== 6. 自定义自动配置 ====================

    /**
     * 自定义自动配置类
     */
    @Configuration
    @ConditionalOnClass(MyService.class)
    @ConditionalOnProperty(prefix = "my.service", name = "enabled", havingValue = "true", matchIfMissing = true)
    @EnableConfigurationProperties(MyServiceProperties.class)
    public class MyServiceAutoConfiguration {

        @Bean
        @ConditionalOnMissingBean
        public MyService myService(MyServiceProperties properties) {
            MyService service = new MyService();
            service.setName(properties.getName());
            service.setTimeout(properties.getTimeout());
            return service;
        }
    }

    @ConfigurationProperties(prefix = "my.service")
    public class MyServiceProperties {
        private String name = "default";
        private int timeout = 30;
        // getters and setters...
    }

    // 注册到spring.factories
    /*
     * META-INF/spring.factories:
     * org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
     * com.example.autoconfigure.MyServiceAutoConfiguration
     */

    // ==================== 7. 自动配置排序 ====================

    @Configuration
    @AutoConfigureAfter(DataSourceAutoConfiguration.class) // 在某配置之后
    @AutoConfigureBefore(JpaRepositoriesAutoConfiguration.class) // 在某配置之前
    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE) // 指定顺序
    public class MyAutoConfiguration {
        // ...
    }

    // ==================== 8. 查看自动配置报告 ====================

    /*
     * 方式1：启动时添加 --debug
     * java -jar app.jar --debug
     *
     * 方式2：配置文件
     * debug=true
     *
     * 输出：
     * ============================
     * CONDITIONS EVALUATION REPORT
     * ============================
     *
     * Positive matches:
     * -----------------
     * DataSourceAutoConfiguration matched:
     *    - @ConditionalOnClass found required class 'javax.sql.DataSource'
     *
     * Negative matches:
     * -----------------
     * ActiveMQAutoConfiguration:
     *    - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory'
     */
}
```

---

### 107. @SpringBootApplication注解的作用？

```java
/**
 * @SpringBootApplication 注解详解
 */
public class SpringBootApplicationAnnotationDemo {

    // ==================== @SpringBootApplication 源码 ====================

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @SpringBootConfiguration  // 1. 配置类
    @EnableAutoConfiguration  // 2. 启用自动配置
    @ComponentScan(           // 3. 组件扫描
        excludeFilters = {
            @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
            @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
        }
    )
    public @interface SpringBootApplication {

        // 排除自动配置类
        @AliasFor(annotation = EnableAutoConfiguration.class)
        Class<?>[] exclude() default {};

        // 排除自动配置类名
        @AliasFor(annotation = EnableAutoConfiguration.class)
        String[] excludeName() default {};

        // 扫描的基础包
        @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
        String[] scanBasePackages() default {};

        // 扫描的基础包（通过类指定）
        @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
        Class<?>[] scanBasePackageClasses() default {};

        // Bean名称生成器
        @AliasFor(annotation = ComponentScan.class, attribute = "nameGenerator")
        Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

        // 是否代理Bean方法
        @AliasFor(annotation = Configuration.class)
        boolean proxyBeanMethods() default true;
    }

    // ==================== 三个核心注解详解 ====================

    /**
     * 1. @SpringBootConfiguration
     * 等价于 @Configuration，标识这是一个配置类
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Configuration
    public @interface SpringBootConfiguration {
        @AliasFor(annotation = Configuration.class)
        boolean proxyBeanMethods() default true;
    }

    /**
     * 2. @EnableAutoConfiguration
     * 启用Spring Boot自动配置
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage // 自动配置包
    @Import(AutoConfigurationImportSelector.class) // 导入自动配置
    public @interface EnableAutoConfiguration {
        String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
        Class<?>[] exclude() default {};
        String[] excludeName() default {};
    }

    /**
     * 3. @ComponentScan
     * 扫描组件（@Component, @Service, @Controller, @Repository等）
     * 默认扫描启动类所在包及其子包
     */

    // ==================== 使用示例 ====================

    /**
     * 基本使用
     */
    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }

    /**
     * 排除特定自动配置
     */
    @SpringBootApplication(exclude = {
        DataSourceAutoConfiguration.class,
        RedisAutoConfiguration.class
    })
    public class ApplicationWithExclusion {
        public static void main(String[] args) {
            SpringApplication.run(ApplicationWithExclusion.class, args);
        }
    }

    /**
     * 指定扫描包
     */
    @SpringBootApplication(scanBasePackages = {
        "com.example.module1",
        "com.example.module2"
    })
    public class ApplicationWithCustomScan {
        public static void main(String[] args) {
            SpringApplication.run(ApplicationWithCustomScan.class, args);
        }
    }

    /**
     * 等价写法（拆分注解）
     */
    @SpringBootConfiguration
    @EnableAutoConfiguration
    @ComponentScan
    public class ApplicationEquivalent {
        public static void main(String[] args) {
            SpringApplication.run(ApplicationEquivalent.class, args);
        }
    }

    // ==================== 注解属性图解 ====================

    /*
    ┌────────────────────────────────────────────────────────────────────┐
    │                    @SpringBootApplication                          │
    ├────────────────────────────────────────────────────────────────────┤
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐  │
    │  │ @SpringBootConfiguration                                    │  │
    │  │    └── @Configuration                                       │  │
    │  │         └── 标识配置类，可以定义@Bean                         │  │
    │  └─────────────────────────────────────────────────────────────┘  │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐  │
    │  │ @EnableAutoConfiguration                                    │  │
    │  │    ├── @AutoConfigurationPackage                            │  │
    │  │    │      └── 注册启动类所在包                                │  │
    │  │    └── @Import(AutoConfigurationImportSelector)             │  │
    │  │           └── 加载spring.factories中的自动配置类              │  │
    │  └─────────────────────────────────────────────────────────────┘  │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐  │
    │  │ @ComponentScan                                              │  │
    │  │    └── 扫描启动类所在包及子包中的组件                          │  │
    │  │         ├── @Component                                      │  │
    │  │         ├── @Service                                        │  │
    │  │         ├── @Controller                                     │  │
    │  │         └── @Repository                                     │  │
    │  └─────────────────────────────────────────────────────────────┘  │
    │                                                                     │
    └────────────────────────────────────────────────────────────────────┘
    */
}
```

---

### 108. Spring Boot启动流程是怎样的？

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Spring Boot 启动流程                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SpringApplication.run()                                             │
│         │                                                                │
│         ▼                                                                │
│  2. 创建SpringApplication实例                                            │
│         ├── 推断应用类型（Servlet/Reactive/None）                        │
│         ├── 加载ApplicationContextInitializer                           │
│         └── 加载ApplicationListener                                     │
│         │                                                                │
│         ▼                                                                │
│  3. 运行run()方法                                                        │
│         ├── 创建StopWatch计时                                           │
│         ├── 获取SpringApplicationRunListeners                           │
│         ├── 发布ApplicationStartingEvent                                │
│         ├── 准备环境（配置文件加载）                                      │
│         ├── 打印Banner                                                  │
│         ├── 创建ApplicationContext                                      │
│         ├── 准备Context（刷新前）                                        │
│         ├── 刷新Context（核心）                                         │
│         ├── 刷新后处理                                                  │
│         ├── 发布ApplicationStartedEvent                                 │
│         └── 执行Runner                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
/**
 * Spring Boot 启动流程详解
 */
public class SpringBootStartupProcessDemo {

    // ==================== 1. 入口：SpringApplication.run() ====================

    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            // 启动入口
            SpringApplication.run(Application.class, args);
        }
    }

    // ==================== 2. SpringApplication 构造过程 ====================

    public class SpringApplication {

        private final Set<Class<?>> primarySources;
        private WebApplicationType webApplicationType;
        private List<ApplicationContextInitializer<?>> initializers;
        private List<ApplicationListener<?>> listeners;

        public SpringApplication(Class<?>... primarySources) {
            this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));

            // 2.1 推断应用类型
            this.webApplicationType = WebApplicationType.deduceFromClasspath();

            // 2.2 加载ApplicationContextInitializer
            this.initializers = loadInitializers();

            // 2.3 加载ApplicationListener
            this.listeners = loadListeners();

            // 2.4 推断主类
            this.mainApplicationClass = deduceMainApplicationClass();
        }

        // 推断应用类型
        private WebApplicationType deduceFromClasspath() {
            // 检查类路径中的类来判断
            if (ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", null)) {
                return WebApplicationType.REACTIVE; // WebFlux应用
            }
            if (ClassUtils.isPresent("javax.servlet.Servlet", null)) {
                return WebApplicationType.SERVLET; // Servlet应用
            }
            return WebApplicationType.NONE; // 非Web应用
        }

        // 从spring.factories加载初始化器
        private List<ApplicationContextInitializer<?>> loadInitializers() {
            return SpringFactoriesLoader.loadFactories(
                ApplicationContextInitializer.class,
                getClass().getClassLoader()
            );
        }
    }

    // ==================== 3. run()方法核心流程 ====================

    public ConfigurableApplicationContext run(String... args) {
        // 3.1 创建计时器
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        ConfigurableApplicationContext context = null;

        // 3.2 获取SpringApplicationRunListeners
        SpringApplicationRunListeners listeners = getRunListeners(args);

        // 3.3 发布ApplicationStartingEvent
        listeners.starting();

        try {
            // 3.4 封装命令行参数
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

            // 3.5 准备环境
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

            // 3.6 打印Banner
            Banner printedBanner = printBanner(environment);

            // 3.7 创建ApplicationContext
            context = createApplicationContext();

            // 3.8 准备Context
            prepareContext(context, environment, listeners, applicationArguments, printedBanner);

            // 3.9 刷新Context（核心步骤）
            refreshContext(context);

            // 3.10 刷新后处理
            afterRefresh(context, applicationArguments);

            // 3.11 停止计时
            stopWatch.stop();

            // 3.12 发布ApplicationStartedEvent
            listeners.started(context);

            // 3.13 执行Runner
            callRunners(context, applicationArguments);

        } catch (Throwable ex) {
            handleRunFailure(context, ex, listeners);
            throw new IllegalStateException(ex);
        }

        // 3.14 发布ApplicationReadyEvent
        listeners.running(context);

        return context;
    }

    // ==================== 4. 环境准备详解 ====================

    private ConfigurableEnvironment prepareEnvironment(
            SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments) {

        // 4.1 创建环境
        ConfigurableEnvironment environment = getOrCreateEnvironment();

        // 4.2 配置环境
        configureEnvironment(environment, applicationArguments.getSourceArgs());

        // 4.3 发布ApplicationEnvironmentPreparedEvent
        // 此时会加载application.yml/application.properties
        listeners.environmentPrepared(environment);

        // 4.4 绑定环境到SpringApplication
        bindToSpringApplication(environment);

        return environment;
    }

    // ==================== 5. Context创建详解 ====================

    protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = AnnotationConfigServletWebServerApplicationContext.class;
                    break;
                case REACTIVE:
                    contextClass = AnnotationConfigReactiveWebServerApplicationContext.class;
                    break;
                default:
                    contextClass = AnnotationConfigApplicationContext.class;
            }
        }
        return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
    }

    // ==================== 6. Context准备详解 ====================

    private void prepareContext(ConfigurableApplicationContext context,
            ConfigurableEnvironment environment,
            SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments,
            Banner printedBanner) {

        // 6.1 设置环境
        context.setEnvironment(environment);

        // 6.2 后处理Context
        postProcessApplicationContext(context);

        // 6.3 执行Initializers
        applyInitializers(context);

        // 6.4 发布ApplicationContextInitializedEvent
        listeners.contextPrepared(context);

        // 6.5 注册特殊Bean
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        if (printedBanner != null) {
            beanFactory.registerSingleton("springBootBanner", printedBanner);
        }

        // 6.6 加载sources（主配置类）
        Set<Object> sources = getAllSources();
        load(context, sources.toArray(new Object[0]));

        // 6.7 发布ApplicationPreparedEvent
        listeners.contextLoaded(context);
    }

    // ==================== 7. Context刷新（最核心） ====================

    private void refreshContext(ConfigurableApplicationContext context) {
        // 调用AbstractApplicationContext.refresh()
        context.refresh();
    }

    /**
     * AbstractApplicationContext.refresh() 详解
     * 这是Spring IoC容器启动的核心方法
     */
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {

            // 1. 准备刷新
            prepareRefresh();

            // 2. 获取BeanFactory
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 3. 准备BeanFactory
            prepareBeanFactory(beanFactory);

            try {
                // 4. BeanFactory后处理（子类扩展点）
                postProcessBeanFactory(beanFactory);

                // 5. 执行BeanFactoryPostProcessor
                invokeBeanFactoryPostProcessors(beanFactory);

                // 6. 注册BeanPostProcessor
                registerBeanPostProcessors(beanFactory);

                // 7. 初始化MessageSource（国际化）
                initMessageSource();

                // 8. 初始化事件广播器
                initApplicationEventMulticaster();

                // 9. 子类扩展点（如创建Web服务器）
                onRefresh();

                // 10. 注册监听器
                registerListeners();

                // 11. 实例化所有非懒加载的单例Bean（核心）
                finishBeanFactoryInitialization(beanFactory);

                // 12. 完成刷新（发布ContextRefreshedEvent）
                finishRefresh();

            } catch (BeansException ex) {
                // 销毁已创建的Bean
                destroyBeans();
                // 取消刷新
                cancelRefresh(ex);
                throw ex;
            } finally {
                // 清理缓存
                resetCommonCaches();
            }
        }
    }

    // ==================== 8. Runner执行 ====================

    /**
     * 执行ApplicationRunner和CommandLineRunner
     */
    private void callRunners(ApplicationContext context, ApplicationArguments args) {
        List<Object> runners = new ArrayList<>();

        // 获取所有ApplicationRunner
        runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());

        // 获取所有CommandLineRunner
        runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());

        // 排序
        AnnotationAwareOrderComparator.sort(runners);

        // 执行
        for (Object runner : runners) {
            if (runner instanceof ApplicationRunner) {
                ((ApplicationRunner) runner).run(args);
            }
            if (runner instanceof CommandLineRunner) {
                ((CommandLineRunner) runner).run(args.getSourceArgs());
            }
        }
    }

    /**
     * 自定义Runner示例
     */
    @Component
    @Order(1)
    public class MyApplicationRunner implements ApplicationRunner {
        @Override
        public void run(ApplicationArguments args) throws Exception {
            System.out.println("ApplicationRunner执行，参数: " + args.getOptionNames());
        }
    }

    @Component
    @Order(2)
    public class MyCommandLineRunner implements CommandLineRunner {
        @Override
        public void run(String... args) throws Exception {
            System.out.println("CommandLineRunner执行，参数: " + Arrays.toString(args));
        }
    }

    // ==================== 9. 启动事件时序图 ====================

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        启动事件时序                                      │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  ApplicationStartingEvent          ──> 应用开始启动                      │
    │         │                                                                │
    │         ▼                                                                │
    │  ApplicationEnvironmentPreparedEvent ──> 环境准备完成                    │
    │         │                                                                │
    │         ▼                                                                │
    │  ApplicationContextInitializedEvent  ──> Context初始化完成               │
    │         │                                                                │
    │         ▼                                                                │
    │  ApplicationPreparedEvent           ──> Context准备完成                  │
    │         │                                                                │
    │         ▼                                                                │
    │  ContextRefreshedEvent              ──> Context刷新完成                  │
    │         │                                                                │
    │         ▼                                                                │
    │  ApplicationStartedEvent            ──> 应用启动完成                     │
    │         │                                                                │
    │         ▼                                                                │
    │  ApplicationReadyEvent              ──> 应用就绪（Runner执行后）          │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 10. 监听启动事件 ====================

    @Component
    public class StartupEventListener {

        @EventListener
        public void onApplicationStarted(ApplicationStartedEvent event) {
            System.out.println("应用启动完成");
        }

        @EventListener
        public void onApplicationReady(ApplicationReadyEvent event) {
            System.out.println("应用就绪，可以接收请求");
        }

        @EventListener
        public void onContextRefreshed(ContextRefreshedEvent event) {
            System.out.println("Context刷新完成");
        }
    }
}
```

---

### 109. 如何自定义Starter？

```java
/**
 * 自定义Spring Boot Starter
 */
public class CustomStarterDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                    自定义Starter项目结构                                 │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  my-spring-boot-starter/                                                │
    │  ├── my-spring-boot-starter-autoconfigure/   # 自动配置模块             │
    │  │   ├── src/main/java/                                                 │
    │  │   │   └── com/example/autoconfigure/                                │
    │  │   │       ├── MyServiceAutoConfiguration.java                       │
    │  │   │       └── MyServiceProperties.java                              │
    │  │   ├── src/main/resources/                                           │
    │  │   │   └── META-INF/                                                 │
    │  │   │       └── spring.factories                                      │
    │  │   └── pom.xml                                                       │
    │  │                                                                       │
    │  └── my-spring-boot-starter/                 # starter模块（空壳）      │
    │      └── pom.xml                             # 只包含依赖               │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 自动配置模块 pom.xml ====================

    /*
    <!-- my-spring-boot-starter-autoconfigure/pom.xml -->
    <project>
        <groupId>com.example</groupId>
        <artifactId>my-spring-boot-starter-autoconfigure</artifactId>
        <version>1.0.0</version>

        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-autoconfigure</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-configuration-processor</artifactId>
                <optional>true</optional>
            </dependency>
        </dependencies>
    </project>
    */

    // ==================== 2. 配置属性类 ====================

    @ConfigurationProperties(prefix = "my.service")
    public class MyServiceProperties {

        /**
         * 是否启用服务
         */
        private boolean enabled = true;

        /**
         * 服务名称
         */
        private String name = "default-service";

        /**
         * 连接超时时间（毫秒）
         */
        private int connectTimeout = 5000;

        /**
         * 读取超时时间（毫秒）
         */
        private int readTimeout = 10000;

        /**
         * 重试次数
         */
        private int retryCount = 3;

        // getters and setters...

        public boolean isEnabled() {
            return enabled;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        // ... 其他getter/setter
    }

    // ==================== 3. 服务类 ====================

    public class MyService {

        private String name;
        private int connectTimeout;
        private int readTimeout;
        private int retryCount;

        public MyService(MyServiceProperties properties) {
            this.name = properties.getName();
            this.connectTimeout = properties.getConnectTimeout();
            this.readTimeout = properties.getReadTimeout();
            this.retryCount = properties.getRetryCount();
        }

        public String doSomething(String input) {
            return String.format("[%s] Processed: %s", name, input);
        }

        public void connect(String url) {
            System.out.println("Connecting to " + url +
                " with timeout: " + connectTimeout + "ms");
        }
    }

    // ==================== 4. 自动配置类 ====================

    @Configuration
    @ConditionalOnClass(MyService.class)  // MyService类存在时生效
    @ConditionalOnProperty(
        prefix = "my.service",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true  // 默认启用
    )
    @EnableConfigurationProperties(MyServiceProperties.class)
    public class MyServiceAutoConfiguration {

        @Bean
        @ConditionalOnMissingBean  // 用户没有自定义时才创建
        public MyService myService(MyServiceProperties properties) {
            return new MyService(properties);
        }

        /**
         * 可选：提供更多条件配置
         */
        @Bean
        @ConditionalOnBean(DataSource.class)  // 存在数据源时才创建
        @ConditionalOnMissingBean
        public MyServiceHealthIndicator myServiceHealthIndicator(MyService myService) {
            return new MyServiceHealthIndicator(myService);
        }
    }

    // ==================== 5. spring.factories 配置 ====================

    /*
     * META-INF/spring.factories (Spring Boot 2.7之前)
     *
     * org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
     * com.example.autoconfigure.MyServiceAutoConfiguration
     */

    /*
     * META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
     * (Spring Boot 2.7+)
     *
     * com.example.autoconfigure.MyServiceAutoConfiguration
     */

    // ==================== 6. Starter模块 pom.xml ====================

    /*
    <!-- my-spring-boot-starter/pom.xml -->
    <project>
        <groupId>com.example</groupId>
        <artifactId>my-spring-boot-starter</artifactId>
        <version>1.0.0</version>

        <dependencies>
            <!-- 自动配置模块 -->
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-spring-boot-starter-autoconfigure</artifactId>
                <version>${project.version}</version>
            </dependency>

            <!-- 其他必要依赖 -->
        </dependencies>
    </project>
    */

    // ==================== 7. 使用自定义Starter ====================

    /*
     * 用户项目pom.xml
     *
     * <dependency>
     *     <groupId>com.example</groupId>
     *     <artifactId>my-spring-boot-starter</artifactId>
     *     <version>1.0.0</version>
     * </dependency>
     */

    // application.yml配置
    /*
    my:
      service:
        enabled: true
        name: my-custom-service
        connect-timeout: 3000
        read-timeout: 5000
        retry-count: 5
    */

    // 使用
    @RestController
    public class DemoController {

        @Autowired
        private MyService myService;  // 自动注入

        @GetMapping("/demo")
        public String demo() {
            return myService.doSomething("Hello");
        }
    }

    // ==================== 8. 配置提示（可选） ====================

    /*
     * META-INF/spring-configuration-metadata.json
     * 或使用 spring-boot-configuration-processor 自动生成
     */

    /*
    {
      "groups": [
        {
          "name": "my.service",
          "type": "com.example.autoconfigure.MyServiceProperties",
          "sourceType": "com.example.autoconfigure.MyServiceProperties"
        }
      ],
      "properties": [
        {
          "name": "my.service.enabled",
          "type": "java.lang.Boolean",
          "description": "是否启用服务",
          "defaultValue": true
        },
        {
          "name": "my.service.name",
          "type": "java.lang.String",
          "description": "服务名称",
          "defaultValue": "default-service"
        }
      ]
    }
    */

    // ==================== 9. 命名规范 ====================

    /*
     * 官方Starter命名：spring-boot-starter-{name}
     * 如：spring-boot-starter-web
     *
     * 自定义Starter命名：{name}-spring-boot-starter
     * 如：my-spring-boot-starter
     */
}
```

---

### 110. Spring Boot配置加载顺序？

```java
/**
 * Spring Boot 配置加载顺序详解
 */
public class ConfigurationLoadOrderDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                   配置加载顺序（优先级从高到低）                          │
    ├─────┬───────────────────────────────────────────────────────────────────┤
    │  1  │ 命令行参数 --server.port=8081                                     │
    │  2  │ SPRING_APPLICATION_JSON 环境变量                                  │
    │  3  │ ServletConfig/ServletContext 初始化参数                           │
    │  4  │ JNDI属性                                                          │
    │  5  │ Java系统属性 System.getProperties()                               │
    │  6  │ 操作系统环境变量                                                   │
    │  7  │ RandomValuePropertySource (random.*)                              │
    │  8  │ jar包外的 application-{profile}.yml/properties                    │
    │  9  │ jar包内的 application-{profile}.yml/properties                    │
    │ 10  │ jar包外的 application.yml/properties                              │
    │ 11  │ jar包内的 application.yml/properties                              │
    │ 12  │ @PropertySource 注解指定的配置                                    │
    │ 13  │ 默认属性 SpringApplication.setDefaultProperties()                 │
    └─────┴───────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 命令行参数（最高优先级） ====================

    // java -jar app.jar --server.port=8081 --spring.profiles.active=prod

    // ==================== 2. 环境变量配置 ====================

    // SPRING_APPLICATION_JSON='{"server":{"port":8081}}'
    // SERVER_PORT=8081  (下划线分隔，全大写)

    // ==================== 3. 配置文件位置优先级 ====================

    /*
     * 同一目录下: .properties > .yml > .yaml
     *
     * 配置文件位置优先级（从高到低）:
     * 1. file:./config/
     * 2. file:./config/子目录/
     * 3. file:./
     * 4. classpath:/config/
     * 5. classpath:/
     */

    // ==================== 4. Profile配置 ====================

    // application.yml
    /*
    spring:
      profiles:
        active: dev  # 激活的profile

    ---
    spring:
      config:
        activate:
          on-profile: dev
    server:
      port: 8080

    ---
    spring:
      config:
        activate:
          on-profile: prod
    server:
      port: 80
    */

    // 或使用独立文件
    // application-dev.yml
    // application-prod.yml

    // ==================== 5. 配置文件示例 ====================

    // application.yml
    /*
    server:
      port: 8080
      servlet:
        context-path: /api

    spring:
      application:
        name: my-application
      datasource:
        url: jdbc:mysql://localhost:3306/test
        username: root
        password: ${DB_PASSWORD:default}  # 支持环境变量和默认值

    # 自定义配置
    my:
      config:
        name: ${MY_NAME:default-name}
        timeout: ${MY_TIMEOUT:5000}
        items:
          - item1
          - item2
    */

    // ==================== 6. 读取配置的方式 ====================

    @Component
    public class ConfigReaderDemo {

        // 方式1：@Value
        @Value("${server.port}")
        private int port;

        @Value("${my.config.name:default}")  // 带默认值
        private String name;

        // 方式2：@ConfigurationProperties
        @Autowired
        private MyConfigProperties myConfig;

        // 方式3：Environment
        @Autowired
        private Environment environment;

        public void readConfig() {
            String value = environment.getProperty("my.config.name");
            Integer timeout = environment.getProperty("my.config.timeout", Integer.class, 5000);
        }
    }

    @ConfigurationProperties(prefix = "my.config")
    @Component
    public class MyConfigProperties {
        private String name;
        private int timeout;
        private List<String> items;
        // getters and setters
    }

    // ==================== 7. 多环境配置最佳实践 ====================

    /*
     * 项目结构：
     * src/main/resources/
     * ├── application.yml          # 通用配置
     * ├── application-dev.yml      # 开发环境
     * ├── application-test.yml     # 测试环境
     * ├── application-prod.yml     # 生产环境
     * └── application-local.yml    # 本地配置（gitignore）
     */

    // application.yml（通用配置）
    /*
    spring:
      profiles:
        active: ${SPRING_PROFILES_ACTIVE:dev}
      application:
        name: my-app

    logging:
      level:
        root: INFO
    */

    // application-dev.yml
    /*
    server:
      port: 8080

    spring:
      datasource:
        url: jdbc:mysql://localhost:3306/dev_db

    logging:
      level:
        com.example: DEBUG
    */

    // application-prod.yml
    /*
    server:
      port: 80

    spring:
      datasource:
        url: jdbc:mysql://prod-db:3306/prod_db

    logging:
      level:
        com.example: WARN
    */

    // ==================== 8. 配置加密 ====================

    // 使用jasypt加密敏感配置
    /*
    spring:
      datasource:
        password: ENC(encrypted_password_here)

    jasypt:
      encryptor:
        password: ${JASYPT_PASSWORD}  # 解密密钥从环境变量获取
    */

    // ==================== 9. 配置刷新 ====================

    @RefreshScope  // 支持配置热刷新（配合Spring Cloud Config）
    @Component
    public class RefreshableConfig {

        @Value("${my.config.name}")
        private String name;

        public String getName() {
            return name;
        }
    }
}
```

---

### 111. Spring Boot的Actuator有什么用？

```java
/**
 * Spring Boot Actuator 详解
 */
public class ActuatorDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                     Actuator 功能概览                                    │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
    │  │  健康检查     │  │  指标监控     │  │  应用信息     │                  │
    │  │  /health     │  │  /metrics    │  │  /info       │                  │
    │  └──────────────┘  └──────────────┘  └──────────────┘                  │
    │                                                                          │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
    │  │  环境配置     │  │  Bean信息     │  │  日志配置     │                  │
    │  │  /env        │  │  /beans      │  │  /loggers    │                  │
    │  └──────────────┘  └──────────────┘  └──────────────┘                  │
    │                                                                          │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
    │  │  线程转储     │  │  堆转储       │  │  HTTP追踪    │                  │
    │  │  /threaddump │  │  /heapdump   │  │  /httptrace  │                  │
    │  └──────────────┘  └──────────────┘  └──────────────┘                  │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 添加依赖 ====================

    /*
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    */

    // ==================== 2. 配置端点 ====================

    // application.yml
    /*
    management:
      endpoints:
        web:
          exposure:
            include: "*"  # 暴露所有端点，生产环境谨慎配置
            # include: health,info,metrics
          base-path: /actuator  # 端点基础路径
      endpoint:
        health:
          show-details: always  # 显示健康检查详情
        shutdown:
          enabled: true  # 启用优雅关闭端点（危险）
      server:
        port: 8081  # 管理端口，与应用端口分离
    */

    // ==================== 3. 常用端点详解 ====================

    /*
    ┌─────────────────┬────────────────────────────────────────────────────────┐
    │     端点         │                    说明                                │
    ├─────────────────┼────────────────────────────────────────────────────────┤
    │ /health         │ 应用健康状态                                           │
    │ /info           │ 应用信息                                               │
    │ /metrics        │ 应用指标                                               │
    │ /env            │ 环境配置                                               │
    │ /beans          │ 所有Bean列表                                           │
    │ /configprops    │ 配置属性                                               │
    │ /mappings       │ URL映射                                                │
    │ /loggers        │ 日志配置（可动态修改）                                   │
    │ /threaddump     │ 线程转储                                               │
    │ /heapdump       │ 堆转储（下载文件）                                      │
    │ /scheduledtasks │ 定时任务                                               │
    │ /httptrace      │ HTTP请求追踪                                           │
    │ /shutdown       │ 优雅关闭应用（POST请求）                                 │
    │ /prometheus     │ Prometheus格式指标（需要micrometer-registry-prometheus）│
    └─────────────────┴────────────────────────────────────────────────────────┘
    */

    // ==================== 4. 自定义健康检查 ====================

    @Component
    public class CustomHealthIndicator implements HealthIndicator {

        @Autowired
        private DataSource dataSource;

        @Override
        public Health health() {
            try {
                // 检查数据库连接
                Connection conn = dataSource.getConnection();
                conn.close();

                return Health.up()
                    .withDetail("database", "connected")
                    .withDetail("timestamp", System.currentTimeMillis())
                    .build();

            } catch (Exception e) {
                return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
            }
        }
    }

    /**
     * 响应式健康检查（WebFlux）
     */
    @Component
    public class ReactiveHealthIndicator implements ReactiveHealthIndicator {

        @Override
        public Mono<Health> health() {
            return checkHealth()
                .map(status -> Health.up().withDetail("status", status).build())
                .onErrorResume(ex -> Mono.just(Health.down(ex).build()));
        }

        private Mono<String> checkHealth() {
            return Mono.just("OK");
        }
    }

    // ==================== 5. 自定义端点 ====================

    @Component
    @Endpoint(id = "custom")
    public class CustomEndpoint {

        private Map<String, Object> data = new ConcurrentHashMap<>();

        @ReadOperation  // GET请求
        public Map<String, Object> getData() {
            return Collections.unmodifiableMap(data);
        }

        @ReadOperation  // GET请求带参数
        public Object getDataByKey(@Selector String key) {
            return data.get(key);
        }

        @WriteOperation  // POST请求
        public void setData(@Selector String key, String value) {
            data.put(key, value);
        }

        @DeleteOperation  // DELETE请求
        public void deleteData(@Selector String key) {
            data.remove(key);
        }
    }

    // 访问：GET /actuator/custom
    // 访问：GET /actuator/custom/myKey
    // 访问：POST /actuator/custom/myKey?value=myValue

    // ==================== 6. 自定义指标 ====================

    @Component
    public class CustomMetrics {

        private final Counter requestCounter;
        private final Timer requestTimer;
        private final AtomicInteger activeUsers;

        public CustomMetrics(MeterRegistry registry) {
            // 计数器
            this.requestCounter = Counter.builder("app.requests.total")
                .description("Total number of requests")
                .tag("type", "api")
                .register(registry);

            // 计时器
            this.requestTimer = Timer.builder("app.requests.duration")
                .description("Request duration")
                .register(registry);

            // Gauge（即时值）
            this.activeUsers = new AtomicInteger(0);
            Gauge.builder("app.users.active", activeUsers, AtomicInteger::get)
                .description("Number of active users")
                .register(registry);
        }

        public void recordRequest(Runnable operation) {
            requestCounter.increment();
            requestTimer.record(operation);
        }

        public void userLogin() {
            activeUsers.incrementAndGet();
        }

        public void userLogout() {
            activeUsers.decrementAndGet();
        }
    }

    // ==================== 7. 与Prometheus集成 ====================

    /*
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    */

    // application.yml
    /*
    management:
      endpoints:
        web:
          exposure:
            include: prometheus,health,info
      metrics:
        export:
          prometheus:
            enabled: true
        tags:
          application: ${spring.application.name}
    */

    // 访问 /actuator/prometheus 获取Prometheus格式的指标

    // ==================== 8. 安全配置 ====================

    @Configuration
    public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .requestMatcher(EndpointRequest.toAnyEndpoint())
                .authorizeRequests()
                    .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                    .anyRequest().hasRole("ACTUATOR_ADMIN")
                .and()
                .httpBasic();
        }
    }
}
```

---

### 112. Spring Boot如何实现热部署？

```java
/**
 * Spring Boot 热部署方案
 */
public class HotDeployDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        热部署方案对比                                    │
    ├──────────────────┬──────────────────────────────────────────────────────┤
    │     方案          │                  特点                                │
    ├──────────────────┼──────────────────────────────────────────────────────┤
    │ Spring DevTools  │ 官方方案，自动重启，适合开发环境                       │
    │ JRebel           │ 商业方案，真正热替换，无需重启                         │
    │ Spring Loaded    │ 已停止维护                                           │
    │ IDEA热交换       │ IDE支持，仅支持方法体修改                              │
    └──────────────────┴──────────────────────────────────────────────────────┘
    */

    // ==================== 1. Spring DevTools 配置 ====================

    /*
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
    */

    // application.yml
    /*
    spring:
      devtools:
        restart:
          enabled: true  # 启用自动重启
          additional-paths: src/main/java  # 额外监控路径
          exclude: static/**,public/**  # 排除不触发重启的路径
          poll-interval: 1s  # 轮询间隔
          quiet-period: 400ms  # 静默期（等待更多变化）
        livereload:
          enabled: true  # 启用LiveReload（浏览器自动刷新）
    */

    // ==================== 2. DevTools 原理 ====================

    /*
     * DevTools 使用两个类加载器：
     *
     * 1. Base ClassLoader
     *    - 加载不变的类（如第三方jar包）
     *    - 应用启动时加载
     *
     * 2. Restart ClassLoader
     *    - 加载项目代码
     *    - 代码修改时丢弃并重建
     *
     * 优点：比完全重启快（只重载项目代码）
     * 缺点：仍然是重启，有状态会丢失
     */

    // ==================== 3. IDEA 配置 ====================

    /*
     * 1. File -> Settings -> Build, Execution, Deployment
     *    -> Compiler -> Build project automatically ✓
     *
     * 2. File -> Settings -> Advanced Settings
     *    -> Allow auto-make to start even if developed application is currently running ✓
     *
     * 或者：
     * 3. 快捷键 Ctrl+Shift+Alt+/ -> Registry
     *    -> compiler.automake.allow.when.app.running ✓
     */

    // ==================== 4. 触发重启的文件 ====================

    /*
     * 默认情况下，以下路径的修改会触发重启：
     * - /META-INF/maven
     * - /META-INF/resources
     * - /resources
     * - /static
     * - /public
     * - /templates
     *
     * 自定义排除：
     * spring.devtools.restart.exclude=static/**,public/**
     *
     * 添加额外监控路径：
     * spring.devtools.restart.additional-paths=src/main/java
     */

    // ==================== 5. 远程热部署（不推荐生产使用） ====================

    // application.yml
    /*
    spring:
      devtools:
        remote:
          secret: my-secret-key  # 设置密钥
    */

    // 启动远程应用时添加参数
    // java -jar app.jar --spring.devtools.remote.secret=my-secret-key

    // 本地运行 RemoteSpringApplication
    // org.springframework.boot.devtools.RemoteSpringApplication

    // ==================== 6. LiveReload 浏览器插件 ====================

    /*
     * 安装浏览器插件 LiveReload
     * 修改静态资源时，浏览器自动刷新
     *
     * 禁用 LiveReload:
     * spring.devtools.livereload.enabled=false
     */

    // ==================== 7. 禁用特定自动配置的缓存 ====================

    // DevTools 自动禁用模板引擎缓存
    /*
    spring:
      thymeleaf:
        cache: false
      freemarker:
        cache: false
      groovy:
        template:
          cache: false
    */

    // ==================== 8. 自定义重启触发器 ====================

    // trigger-file 配置
    /*
    spring:
      devtools:
        restart:
          trigger-file: .reloadtrigger  # 只有修改此文件才触发重启
    */

    // ==================== 9. JRebel 配置（商业方案） ====================

    /*
     * JRebel 是商业热部署工具，真正实现热替换：
     *
     * 1. 安装 JRebel 插件（IDEA Plugins）
     * 2. 激活 JRebel
     * 3. 使用 JRebel 启动应用
     *
     * 优点：
     * - 真正热替换，不丢失状态
     * - 支持修改类结构
     *
     * 缺点：
     * - 商业软件，需要付费
     */

    // ==================== 10. 最佳实践 ====================

    /*
     * 开发环境:
     * - 使用 DevTools + IDEA自动编译
     * - 必要时考虑 JRebel
     *
     * 测试/生产环境:
     * - 禁用 DevTools（打包时自动排除）
     * - 使用滚动部署实现零停机
     *
     * 注意事项:
     * - DevTools 只在 开发环境 使用
     * - 生产环境打包时会自动排除 DevTools
     * - scope 设置为 runtime 和 optional
     */
}
```

---

### 113. Spring Boot如何优化启动速度？

```java
/**
 * Spring Boot 启动速度优化
 */
public class StartupOptimizationDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      启动优化策略                                        │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  1. 减少自动配置          5. 使用懒加载                                  │
    │  2. 减少组件扫描范围      6. 关闭JMX                                     │
    │  3. 使用索引加速扫描      7. AOT编译（GraalVM）                          │
    │  4. 延迟初始化Bean       8. 优化JVM参数                                  │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 减少自动配置 ====================

    @SpringBootApplication(exclude = {
        // 排除不需要的自动配置
        DataSourceAutoConfiguration.class,
        RedisAutoConfiguration.class,
        MongoAutoConfiguration.class,
        KafkaAutoConfiguration.class,
        SecurityAutoConfiguration.class,
        JmxAutoConfiguration.class
    })
    public class OptimizedApplication {
        public static void main(String[] args) {
            SpringApplication.run(OptimizedApplication.class, args);
        }
    }

    // 或在配置文件中排除
    /*
    spring:
      autoconfigure:
        exclude:
          - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
          - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
    */

    // ==================== 2. 减少组件扫描范围 ====================

    @SpringBootApplication
    @ComponentScan(basePackages = "com.example.core")  // 只扫描必要的包
    public class LimitedScanApplication {
        // ...
    }

    // 或使用更精确的扫描
    @ComponentScan(
        basePackages = "com.example",
        excludeFilters = @ComponentScan.Filter(
            type = FilterType.REGEX,
            pattern = "com.example.unused.*"
        )
    )

    // ==================== 3. 使用组件索引 ====================

    /*
    <!-- 编译时生成组件索引，加速启动时的组件扫描 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-indexer</artifactId>
        <optional>true</optional>
    </dependency>
    */

    // 生成 META-INF/spring.components 索引文件
    // 启动时直接读取索引，无需扫描类路径

    // ==================== 4. 延迟初始化Bean ====================

    // 全局延迟初始化
    /*
    spring:
      main:
        lazy-initialization: true
    */

    // 或代码配置
    @SpringBootApplication
    public class LazyApplication {
        public static void main(String[] args) {
            SpringApplication app = new SpringApplication(LazyApplication.class);
            app.setLazyInitialization(true);
            app.run(args);
        }
    }

    // 单个Bean延迟初始化
    @Lazy
    @Service
    public class HeavyService {
        // 只在第一次使用时初始化
    }

    // 排除某些Bean不延迟（需要立即初始化）
    @Lazy(false)
    @Component
    public class EagerBean {
        // 立即初始化
    }

    // ==================== 5. 关闭不需要的功能 ====================

    /*
    spring:
      jmx:
        enabled: false  # 关闭JMX
      main:
        banner-mode: off  # 关闭Banner

    management:
      endpoints:
        enabled-by-default: false  # 关闭Actuator端点

    logging:
      level:
        root: WARN  # 减少日志输出
    */

    // ==================== 6. JVM参数优化 ====================

    /*
    # 基础优化
    java -jar app.jar \
      -Xms512m \                    # 初始堆大小
      -Xmx512m \                    # 最大堆大小（设置相同避免扩展）
      -XX:+UseG1GC \                # 使用G1垃圾收集器
      -XX:MaxGCPauseMillis=100 \    # 最大GC暂停时间
      -XX:+TieredCompilation \       # 分层编译
      -XX:TieredStopAtLevel=1 \      # 只使用C1编译器（启动快）
      -noverify                      # 跳过字节码验证

    # 类数据共享（CDS）
    java -Xshare:dump                # 先生成共享存档
    java -Xshare:on -jar app.jar     # 使用共享存档启动
    */

    // ==================== 7. Spring AOT（Spring 6+） ====================

    /*
    <!-- Spring 6 / Spring Boot 3 支持AOT编译 -->
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <executions>
            <execution>
                <id>process-aot</id>
                <goals>
                    <goal>process-aot</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
    */

    // ==================== 8. GraalVM Native Image ====================

    /*
    <!-- 使用GraalVM编译原生镜像 -->
    <plugin>
        <groupId>org.graalvm.buildtools</groupId>
        <artifactId>native-maven-plugin</artifactId>
    </plugin>

    编译命令：
    mvn -Pnative native:compile

    优点：启动时间毫秒级，内存占用低
    缺点：编译慢，部分反射功能需要配置
    */

    // ==================== 9. 启动耗时分析 ====================

    @SpringBootApplication
    public class StartupAnalysisApplication {

        public static void main(String[] args) {
            SpringApplication app = new SpringApplication(StartupAnalysisApplication.class);
            // 启用启动分析
            app.setApplicationStartup(new BufferingApplicationStartup(2048));
            app.run(args);
        }
    }

    // 访问 /actuator/startup 查看启动分析报告

    // ==================== 10. 异步初始化 ====================

    @Configuration
    public class AsyncInitConfig {

        @Bean
        public ApplicationRunner asyncInit(ApplicationContext context) {
            return args -> {
                // 异步初始化重量级组件
                CompletableFuture.runAsync(() -> {
                    // 预热缓存
                    warmUpCache();
                    // 建立数据库连接池
                    initConnectionPool();
                });
            };
        }
    }

    // ==================== 11. 启动优化检查清单 ====================

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        启动优化检查清单                                  │
    ├─────────────────────────────────────────────────────────────────────────┤
    │ □ 排除不需要的自动配置                                                  │
    │ □ 限制组件扫描范围                                                      │
    │ □ 使用spring-context-indexer生成索引                                   │
    │ □ 开启全局懒加载（如适用）                                              │
    │ □ 关闭JMX和不需要的Actuator端点                                        │
    │ □ 优化JVM参数（堆大小、GC、分层编译）                                   │
    │ □ 使用CDS（类数据共享）                                                │
    │ □ 考虑使用Spring AOT或GraalVM                                         │
    │ □ 分析启动报告，识别慢Bean                                             │
    │ □ 异步初始化非关键组件                                                  │
    └─────────────────────────────────────────────────────────────────────────┘
    */
}
```

---

## 4.3 Spring Cloud（8题）

### 114. Spring Cloud的核心组件有哪些？

```java
/**
 * Spring Cloud 核心组件详解
 */
public class SpringCloudComponentsDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                    Spring Cloud 核心组件                                 │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │   ┌─────────────────────────────────────────────────────────────────┐  │
    │   │                        API Gateway                              │  │
    │   │              (Spring Cloud Gateway / Zuul)                      │  │
    │   └─────────────────────────────────────────────────────────────────┘  │
    │                                  │                                      │
    │                                  ▼                                      │
    │   ┌─────────────────────────────────────────────────────────────────┐  │
    │   │                     服务注册与发现                                │  │
    │   │              (Nacos / Eureka / Consul)                          │  │
    │   └─────────────────────────────────────────────────────────────────┘  │
    │                                  │                                      │
    │          ┌───────────────────────┼───────────────────────┐             │
    │          │                       │                       │             │
    │          ▼                       ▼                       ▼             │
    │   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐     │
    │   │  Service A  │ ──────> │  Service B  │ ──────> │  Service C  │     │
    │   └─────────────┘ Feign   └─────────────┘ Feign   └─────────────┘     │
    │          │                       │                       │             │
    │          └───────────────────────┼───────────────────────┘             │
    │                                  │                                      │
    │                                  ▼                                      │
    │   ┌─────────────────────────────────────────────────────────────────┐  │
    │   │  配置中心(Nacos Config / Config Server)                        │  │
    │   │  熔断限流(Sentinel / Hystrix)                                  │  │
    │   │  链路追踪(Sleuth + Zipkin)                                     │  │
    │   │  消息总线(Bus)                                                  │  │
    │   └─────────────────────────────────────────────────────────────────┘  │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 核心组件对照表 ====================

    /*
    ┌──────────────────┬────────────────────┬────────────────────┐
    │      功能         │  Netflix OSS       │  Alibaba           │
    ├──────────────────┼────────────────────┼────────────────────┤
    │ 服务注册与发现    │ Eureka             │ Nacos              │
    │ 负载均衡         │ Ribbon             │ Ribbon/LoadBalancer│
    │ 服务调用         │ Feign              │ OpenFeign          │
    │ 熔断器           │ Hystrix            │ Sentinel           │
    │ 网关             │ Zuul               │ Gateway            │
    │ 配置中心         │ Config             │ Nacos Config       │
    │ 链路追踪         │ Sleuth + Zipkin    │ Sleuth + Zipkin    │
    │ 消息总线         │ Bus                │ Bus                │
    └──────────────────┴────────────────────┴────────────────────┘
    */

    // ==================== 1. 服务注册与发现 (Nacos) ====================

    /*
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    */

    // application.yml
    /*
    spring:
      cloud:
        nacos:
          discovery:
            server-addr: localhost:8848
            namespace: dev
            group: DEFAULT_GROUP
    */

    @SpringBootApplication
    @EnableDiscoveryClient  // 启用服务发现
    public class ServiceApplication {
        public static void main(String[] args) {
            SpringApplication.run(ServiceApplication.class, args);
        }
    }

    // ==================== 2. 负载均衡 (LoadBalancer) ====================

    @Configuration
    public class LoadBalancerConfig {

        @Bean
        @LoadBalanced  // 启用负载均衡
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }
    }

    @Service
    public class OrderService {
        @Autowired
        private RestTemplate restTemplate;

        public User getUser(Long userId) {
            // 使用服务名调用，自动负载均衡
            return restTemplate.getForObject(
                "http://user-service/users/" + userId,
                User.class
            );
        }
    }

    // ==================== 3. 服务调用 (OpenFeign) ====================

    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    */

    @SpringBootApplication
    @EnableFeignClients  // 启用Feign
    public class OrderApplication {
        // ...
    }

    @FeignClient(name = "user-service", fallback = UserClientFallback.class)
    public interface UserClient {

        @GetMapping("/users/{id}")
        User getUser(@PathVariable("id") Long id);

        @PostMapping("/users")
        User createUser(@RequestBody User user);
    }

    @Component
    public class UserClientFallback implements UserClient {
        @Override
        public User getUser(Long id) {
            return new User(id, "fallback-user");
        }

        @Override
        public User createUser(User user) {
            return null;
        }
    }

    // ==================== 4. 熔断限流 (Sentinel) ====================

    /*
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
    */

    @Service
    public class ProductService {

        @SentinelResource(value = "getProduct",
                         blockHandler = "getProductBlockHandler",
                         fallback = "getProductFallback")
        public Product getProduct(Long id) {
            return productRepository.findById(id).orElseThrow();
        }

        // 限流/熔断处理
        public Product getProductBlockHandler(Long id, BlockException ex) {
            return new Product(id, "限流商品");
        }

        // 异常降级
        public Product getProductFallback(Long id, Throwable ex) {
            return new Product(id, "降级商品");
        }
    }

    // ==================== 5. 网关 (Spring Cloud Gateway) ====================

    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    */

    // application.yml
    /*
    spring:
      cloud:
        gateway:
          routes:
            - id: user-service
              uri: lb://user-service
              predicates:
                - Path=/api/users/**
              filters:
                - StripPrefix=1
                - AddRequestHeader=X-Request-Source, gateway
    */

    // ==================== 6. 配置中心 (Nacos Config) ====================

    /*
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    */

    // bootstrap.yml
    /*
    spring:
      application:
        name: order-service
      cloud:
        nacos:
          config:
            server-addr: localhost:8848
            file-extension: yaml
            namespace: dev
    */

    @RefreshScope  // 支持配置动态刷新
    @RestController
    public class ConfigController {

        @Value("${app.config.value}")
        private String configValue;

        @GetMapping("/config")
        public String getConfig() {
            return configValue;
        }
    }

    // ==================== 7. 链路追踪 (Sleuth + Zipkin) ====================

    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    </dependency>
    */

    // application.yml
    /*
    spring:
      sleuth:
        sampler:
          probability: 1.0  # 采样率
      zipkin:
        base-url: http://localhost:9411
    */
}
```

---

### 115. Nacos和Eureka的区别？

```java
/**
 * Nacos vs Eureka 对比
 */
public class NacosVsEurekaDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      Nacos vs Eureka 对比                               │
    ├──────────────────┬─────────────────────────┬─────────────────────────────┤
    │      特性         │       Nacos             │        Eureka              │
    ├──────────────────┼─────────────────────────┼─────────────────────────────┤
    │ 定位             │ 服务发现 + 配置管理      │ 仅服务发现                   │
    │ CAP模型          │ AP + CP（可切换）        │ AP                          │
    │ 一致性协议       │ Raft（CP模式）           │ 无（最终一致）               │
    │ 健康检查         │ TCP/HTTP/MySQL等         │ 心跳检测                     │
    │ 负载均衡         │ 权重/metadata            │ Ribbon                      │
    │ 雪崩保护         │ 支持                     │ 支持                        │
    │ 配置管理         │ ✅ 支持                  │ ❌ 不支持                    │
    │ 监听查询         │ 支持                     │ 不支持                       │
    │ 界面             │ 功能丰富                 │ 简单                        │
    │ 维护状态         │ 活跃维护                 │ 停止更新                    │
    └──────────────────┴─────────────────────────┴─────────────────────────────┘
    */

    // ==================== 1. Eureka 使用示例 ====================

    // Eureka Server
    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    */

    @SpringBootApplication
    @EnableEurekaServer  // 启用Eureka Server
    public class EurekaServerApplication {
        public static void main(String[] args) {
            SpringApplication.run(EurekaServerApplication.class, args);
        }
    }

    // Eureka Server配置
    /*
    server:
      port: 8761

    eureka:
      instance:
        hostname: localhost
      client:
        register-with-eureka: false  # 不注册自己
        fetch-registry: false
      server:
        enable-self-preservation: true  # 开启自我保护
    */

    // Eureka Client
    @SpringBootApplication
    @EnableDiscoveryClient
    public class EurekaClientApplication {
        public static void main(String[] args) {
            SpringApplication.run(EurekaClientApplication.class, args);
        }
    }

    // Eureka Client配置
    /*
    spring:
      application:
        name: user-service

    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8761/eureka/
      instance:
        prefer-ip-address: true
        lease-renewal-interval-in-seconds: 30    # 心跳间隔
        lease-expiration-duration-in-seconds: 90  # 过期时间
    */

    // ==================== 2. Nacos 使用示例 ====================

    /*
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    */

    @SpringBootApplication
    @EnableDiscoveryClient
    public class NacosClientApplication {
        public static void main(String[] args) {
            SpringApplication.run(NacosClientApplication.class, args);
        }
    }

    // Nacos配置
    /*
    spring:
      application:
        name: user-service
      cloud:
        nacos:
          discovery:
            server-addr: localhost:8848
            namespace: dev          # 命名空间
            group: DEFAULT_GROUP    # 分组
            weight: 1               # 权重
            cluster-name: BJ        # 集群名
            ephemeral: true         # 临时实例（AP）/ 持久实例（CP）
    */

    // ==================== 3. 核心区别详解 ====================

    /**
     * 1. CAP模型区别
     */
    /*
     * Eureka: AP模式
     * - 高可用，分区容错
     * - 服务之间可能数据不一致（最终一致）
     * - 适合服务数量大、对一致性要求不高的场景
     *
     * Nacos: AP + CP可切换
     * - ephemeral=true: AP模式（临时实例）
     * - ephemeral=false: CP模式（持久实例，使用Raft协议）
     * - 更灵活，可根据场景选择
     */

    /**
     * 2. 健康检查区别
     */
    /*
     * Eureka:
     * - 只支持心跳检测
     * - 客户端定期发送心跳
     *
     * Nacos:
     * - 临时实例：客户端心跳
     * - 持久实例：服务端主动检测（TCP/HTTP/MySQL）
     * - 更丰富的健康检查策略
     */

    /**
     * 3. 服务发现模式
     */
    // Eureka: 定期拉取
    @Service
    public class EurekaDiscovery {
        @Autowired
        private DiscoveryClient discoveryClient;

        public List<ServiceInstance> getInstances(String serviceId) {
            return discoveryClient.getInstances(serviceId);
        }
    }

    // Nacos: 支持订阅推送
    @Service
    public class NacosDiscovery {
        @Autowired
        private NamingService namingService;

        public void subscribe(String serviceName) throws NacosException {
            // 订阅服务变化
            namingService.subscribe(serviceName, event -> {
                if (event instanceof NamingEvent) {
                    List<Instance> instances = ((NamingEvent) event).getInstances();
                    // 处理实例变化
                }
            });
        }
    }

    // ==================== 4. 选型建议 ====================

    /*
     * 选择 Nacos 的场景：
     * 1. 需要同时管理服务发现和配置
     * 2. 需要灵活的健康检查策略
     * 3. 需要支持权重负载均衡
     * 4. 需要服务变化的实时推送
     * 5. 使用阿里巴巴技术栈
     *
     * 选择 Eureka 的场景：
     * 1. 简单的服务注册发现需求
     * 2. 已有成熟的 Netflix 技术栈
     * 3. 不需要配置管理功能
     *
     * 实际建议：新项目优先选择 Nacos
     */
}
```

---

### 116. Ribbon的负载均衡策略？

```java
/**
 * Ribbon/LoadBalancer 负载均衡策略
 */
public class RibbonLoadBalancerDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                    Ribbon 负载均衡策略                                   │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  ┌──────────────────────────────────────────────────────────────────┐  │
    │  │                      IRule 接口                                   │  │
    │  └─────────────────────────────┬────────────────────────────────────┘  │
    │                                │                                        │
    │    ┌───────────────────────────┼───────────────────────────────────┐   │
    │    │           │               │               │               │    │   │
    │    ▼           ▼               ▼               ▼               ▼    │   │
    │ ┌──────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌─────────┐  │   │
    │ │Round │  │ Random   │  │ Weighted  │  │ BestAvail│  │ Retry   │  │   │
    │ │Robin │  │          │  │ Response  │  │ able     │  │         │  │   │
    │ └──────┘  └──────────┘  └───────────┘  └──────────┘  └─────────┘  │   │
    │                                                                    │   │
    └────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== Ribbon策略详解 ====================

    /*
    ┌─────────────────────────┬────────────────────────────────────────────────┐
    │        策略              │                   说明                         │
    ├─────────────────────────┼────────────────────────────────────────────────┤
    │ RoundRobinRule          │ 轮询（默认）                                    │
    │ RandomRule              │ 随机选择                                        │
    │ WeightedResponseTimeRule│ 根据响应时间加权，响应越快权重越大               │
    │ BestAvailableRule       │ 选择并发请求最少的服务器                         │
    │ AvailabilityFilteringRule│ 过滤故障和并发过高的服务器                      │
    │ RetryRule               │ 带重试的轮询                                    │
    │ ZoneAvoidanceRule       │ 复合判断（zone+可用性），默认策略                 │
    └─────────────────────────┴────────────────────────────────────────────────┘
    */

    // ==================== 1. 配置文件方式 ====================

    /*
    # 全局配置
    ribbon:
      NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule

    # 针对特定服务配置
    user-service:
      ribbon:
        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
        ConnectTimeout: 3000
        ReadTimeout: 5000
        MaxAutoRetries: 1
        MaxAutoRetriesNextServer: 2
    */

    // ==================== 2. Java配置方式 ====================

    // 全局配置
    @Configuration
    public class GlobalRibbonConfig {

        @Bean
        public IRule ribbonRule() {
            return new RandomRule();  // 随机策略
        }
    }

    // 针对特定服务配置
    @Configuration
    @RibbonClient(name = "user-service", configuration = UserServiceRibbonConfig.class)
    public class RibbonClientConfig {
    }

    // 注意：此配置类不能被@ComponentScan扫描到
    public class UserServiceRibbonConfig {

        @Bean
        public IRule userServiceRule() {
            return new WeightedResponseTimeRule();
        }
    }

    // ==================== 3. 自定义负载均衡策略 ====================

    /**
     * 自定义策略：基于IP Hash
     */
    public class IpHashRule extends AbstractLoadBalancerRule {

        @Override
        public Server choose(Object key) {
            List<Server> servers = getLoadBalancer().getReachableServers();
            if (servers.isEmpty()) {
                return null;
            }

            // 获取客户端IP
            String clientIp = getClientIp();
            int hash = Math.abs(clientIp.hashCode());
            int index = hash % servers.size();

            return servers.get(index);
        }

        private String getClientIp() {
            RequestAttributes attributes = RequestContextHolder.getRequestAttributes();
            if (attributes instanceof ServletRequestAttributes) {
                HttpServletRequest request = ((ServletRequestAttributes) attributes).getRequest();
                return request.getRemoteAddr();
            }
            return "127.0.0.1";
        }

        @Override
        public void initWithNiwsConfig(IClientConfig clientConfig) {
        }
    }

    /**
     * 自定义策略：基于版本号
     */
    public class VersionRule extends AbstractLoadBalancerRule {

        @Override
        public Server choose(Object key) {
            List<Server> servers = getLoadBalancer().getReachableServers();

            // 获取请求头中的版本号
            String requestVersion = getRequestVersion();

            // 过滤匹配版本的服务器
            List<Server> matchedServers = servers.stream()
                .filter(server -> {
                    if (server instanceof NacosServer) {
                        String serverVersion = ((NacosServer) server)
                            .getMetadata().get("version");
                        return requestVersion.equals(serverVersion);
                    }
                    return false;
                })
                .collect(Collectors.toList());

            if (matchedServers.isEmpty()) {
                return servers.get(ThreadLocalRandom.current().nextInt(servers.size()));
            }

            // 在匹配的服务器中轮询
            return matchedServers.get(
                ThreadLocalRandom.current().nextInt(matchedServers.size())
            );
        }
    }

    // ==================== 4. Spring Cloud LoadBalancer (新版) ====================

    /*
    Spring Cloud 2020开始，Ribbon进入维护模式
    推荐使用 Spring Cloud LoadBalancer
    */

    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    */

    // 自定义ReactorServiceInstanceLoadBalancer
    public class CustomLoadBalancer implements ReactorServiceInstanceLoadBalancer {

        private final String serviceId;
        private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
        private final AtomicInteger position = new AtomicInteger(0);

        public CustomLoadBalancer(String serviceId,
                ObjectProvider<ServiceInstanceListSupplier> provider) {
            this.serviceId = serviceId;
            this.serviceInstanceListSupplierProvider = provider;
        }

        @Override
        public Mono<Response<ServiceInstance>> choose(Request request) {
            ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);

            return supplier.get(request)
                .next()
                .map(instances -> getInstanceResponse(instances));
        }

        private Response<ServiceInstance> getInstanceResponse(
                List<ServiceInstance> instances) {
            if (instances.isEmpty()) {
                return new EmptyResponse();
            }
            // 轮询
            int pos = Math.abs(position.incrementAndGet());
            ServiceInstance instance = instances.get(pos % instances.size());
            return new DefaultResponse(instance);
        }
    }

    // 配置
    @Configuration
    @LoadBalancerClient(name = "user-service",
                       configuration = CustomLoadBalancerConfig.class)
    public class LoadBalancerConfig {
    }

    public class CustomLoadBalancerConfig {
        @Bean
        public ReactorLoadBalancer<ServiceInstance> customLoadBalancer(
                Environment environment,
                LoadBalancerClientFactory loadBalancerClientFactory) {
            String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
            return new CustomLoadBalancer(name,
                loadBalancerClientFactory.getLazyProvider(name,
                    ServiceInstanceListSupplier.class));
        }
    }
}
```

---

### 117. Feign的原理是什么？

```java
/**
 * Feign原理详解
 */
public class FeignPrincipleDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         Feign 工作原理                                   │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  1. 扫描@FeignClient注解                                                 │
    │         │                                                                │
    │         ▼                                                                │
    │  2. 为接口创建动态代理（JDK动态代理）                                      │
    │         │                                                                │
    │         ▼                                                                │
    │  3. 调用代理方法时：                                                      │
    │     ┌─────────────────────────────────────────────────────────────┐    │
    │     │ a. 解析方法上的注解（@GetMapping等）生成RequestTemplate      │    │
    │     │ b. 从服务注册中心获取服务实例列表                             │    │
    │     │ c. 通过LoadBalancer选择服务实例                              │    │
    │     │ d. 编码请求（参数序列化）                                    │    │
    │     │ e. 发送HTTP请求                                             │    │
    │     │ f. 解码响应（反序列化）                                      │    │
    │     │ g. 返回结果                                                  │    │
    │     └─────────────────────────────────────────────────────────────┘    │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 基本使用 ====================

    @FeignClient(
        name = "user-service",           // 服务名
        url = "",                         // 直接指定URL（优先级高于name）
        path = "/api",                   // 路径前缀
        fallback = UserClientFallback.class,      // 降级类
        fallbackFactory = UserClientFallbackFactory.class,  // 降级工厂
        configuration = UserClientConfig.class     // 自定义配置
    )
    public interface UserClient {

        @GetMapping("/users/{id}")
        User getUser(@PathVariable("id") Long id);

        @GetMapping("/users")
        List<User> listUsers(@RequestParam("name") String name);

        @PostMapping("/users")
        User createUser(@RequestBody User user);

        @PutMapping("/users/{id}")
        User updateUser(@PathVariable("id") Long id, @RequestBody User user);

        @DeleteMapping("/users/{id}")
        void deleteUser(@PathVariable("id") Long id);
    }

    // ==================== 2. Feign核心组件 ====================

    /**
     * 核心组件说明
     */
    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │ Contract    │ 解析接口方法上的注解，生成方法元数据                        │
    ├─────────────┼───────────────────────────────────────────────────────────┤
    │ Encoder     │ 请求体编码器（对象 -> 请求体）                             │
    ├─────────────┼───────────────────────────────────────────────────────────┤
    │ Decoder     │ 响应体解码器（响应体 -> 对象）                             │
    ├─────────────┼───────────────────────────────────────────────────────────┤
    │ Client      │ HTTP客户端（默认使用HttpURLConnection）                    │
    ├─────────────┼───────────────────────────────────────────────────────────┤
    │ Logger      │ 日志记录器                                                 │
    ├─────────────┼───────────────────────────────────────────────────────────┤
    │ Retryer     │ 重试器                                                    │
    ├─────────────┼───────────────────────────────────────────────────────────┤
    │ ErrorDecoder│ 错误解码器                                                │
    └─────────────┴───────────────────────────────────────────────────────────┘
    */

    // ==================== 3. 自定义配置 ====================

    public class UserClientConfig {

        // 自定义日志级别
        @Bean
        public Logger.Level feignLoggerLevel() {
            return Logger.Level.FULL;
            /*
             * NONE: 不记录日志
             * BASIC: 只记录请求方法、URL、响应状态码、执行时间
             * HEADERS: 在BASIC基础上，额外记录请求和响应头
             * FULL: 记录所有请求和响应的内容、头信息、元数据
             */
        }

        // 自定义编码器
        @Bean
        public Encoder feignEncoder() {
            return new JacksonEncoder();
        }

        // 自定义解码器
        @Bean
        public Decoder feignDecoder() {
            return new JacksonDecoder();
        }

        // 自定义HTTP客户端
        @Bean
        public Client feignClient() {
            return new OkHttpClient();  // 使用OkHttp
        }

        // 自定义重试器
        @Bean
        public Retryer feignRetryer() {
            // 间隔100ms，最大间隔1s，最多重试5次
            return new Retryer.Default(100, 1000, 5);
        }

        // 自定义错误解码器
        @Bean
        public ErrorDecoder errorDecoder() {
            return new CustomErrorDecoder();
        }

        // 请求拦截器
        @Bean
        public RequestInterceptor requestInterceptor() {
            return template -> {
                // 添加请求头
                template.header("Authorization", getToken());
                template.header("X-Request-Id", UUID.randomUUID().toString());
            };
        }
    }

    // ==================== 4. 降级处理 ====================

    // 方式1：Fallback类
    @Component
    public class UserClientFallback implements UserClient {
        @Override
        public User getUser(Long id) {
            return new User(id, "fallback-user");
        }

        @Override
        public List<User> listUsers(String name) {
            return Collections.emptyList();
        }

        @Override
        public User createUser(User user) {
            return null;
        }

        @Override
        public User updateUser(Long id, User user) {
            return null;
        }

        @Override
        public void deleteUser(Long id) {
        }
    }

    // 方式2：FallbackFactory（可以获取异常信息）
    @Component
    public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

        @Override
        public UserClient create(Throwable cause) {
            return new UserClient() {
                @Override
                public User getUser(Long id) {
                    log.error("获取用户失败, id={}, 原因: {}", id, cause.getMessage());
                    return new User(id, "error-user");
                }

                // ... 其他方法
            };
        }
    }

    // ==================== 5. 配置属性 ====================

    /*
    feign:
      client:
        config:
          default:  # 全局配置
            connectTimeout: 5000
            readTimeout: 10000
            loggerLevel: FULL
          user-service:  # 针对特定服务
            connectTimeout: 3000
            readTimeout: 5000

      httpclient:
        enabled: true  # 使用Apache HttpClient
        max-connections: 200
        max-connections-per-route: 50

      okhttp:
        enabled: false  # 或使用OkHttp

      compression:
        request:
          enabled: true
          mime-types: application/json
          min-request-size: 2048
        response:
          enabled: true

      sentinel:
        enabled: true  # 整合Sentinel
    */

    // ==================== 6. 原理源码分析 ====================

    /**
     * Feign代理创建流程（简化版）
     */
    public class FeignClientFactoryBean implements FactoryBean<Object> {

        private Class<?> type;  // 接口类型
        private String name;    // 服务名

        @Override
        public Object getObject() throws Exception {
            // 1. 获取FeignContext
            FeignContext context = applicationContext.getBean(FeignContext.class);

            // 2. 构建Feign.Builder
            Feign.Builder builder = feign(context);

            // 3. 配置编码器、解码器、Client等
            configureFeign(context, builder);

            // 4. 获取服务URL
            String url = getUrl();

            // 5. 创建代理
            return builder.target(type, url);
        }
    }

    /**
     * 方法调用时的处理（简化版）
     */
    public class ReflectiveFeign extends Feign {

        @Override
        public <T> T newInstance(Target<T> target) {
            // 1. 解析方法元数据
            Map<String, MethodHandler> methodHandlers = new HashMap<>();
            for (Method method : target.type().getMethods()) {
                MethodMetadata metadata = parseMethodMetadata(method);
                MethodHandler handler = new SynchronousMethodHandler(
                    target, client, retryer, requestInterceptors,
                    logger, logLevel, metadata, encoder, decoder
                );
                methodHandlers.put(method.getName(), handler);
            }

            // 2. 创建InvocationHandler
            InvocationHandler invocationHandler = new FeignInvocationHandler(
                target, methodHandlers
            );

            // 3. 创建JDK动态代理
            return (T) Proxy.newProxyInstance(
                target.type().getClassLoader(),
                new Class<?>[] { target.type() },
                invocationHandler
            );
        }
    }
}
```

---

### 118. Gateway和Zuul的区别？

```java
/**
 * Spring Cloud Gateway vs Zuul 对比
 */
public class GatewayVsZuulDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                   Gateway vs Zuul 对比                                   │
    ├──────────────────┬─────────────────────────┬─────────────────────────────┤
    │      特性         │   Spring Cloud Gateway  │       Zuul 1.x             │
    ├──────────────────┼─────────────────────────┼─────────────────────────────┤
    │ 编程模型         │ 响应式（WebFlux）        │ 阻塞式（Servlet）           │
    │ 性能             │ 高（非阻塞）             │ 中等（阻塞）                │
    │ 长连接           │ 支持WebSocket           │ 不支持                      │
    │ 限流             │ 内置（RequestRateLimiter）│ 需要整合                   │
    │ 动态路由         │ 支持                     │ 支持                       │
    │ 过滤器           │ GatewayFilter            │ ZuulFilter                 │
    │ 底层框架         │ Spring WebFlux + Netty   │ Spring MVC + Tomcat        │
    │ 维护状态         │ 活跃                     │ 停止更新                    │
    └──────────────────┴─────────────────────────┴─────────────────────────────┘
    */

    // ==================== 1. Gateway 配置 ====================

    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    */

    // application.yml
    /*
    spring:
      cloud:
        gateway:
          routes:
            - id: user-service-route
              uri: lb://user-service    # 负载均衡到user-service
              predicates:               # 断言（路由条件）
                - Path=/api/users/**
                - Method=GET,POST
                - Header=X-Request-Id, \d+
                - Query=name
                - After=2023-01-01T00:00:00+08:00[Asia/Shanghai]
              filters:                  # 过滤器
                - StripPrefix=1         # 去掉第一个路径段
                - AddRequestHeader=X-Gateway, true
                - AddResponseHeader=X-Response-Time, %{responseTime}ms
                - RewritePath=/api/(?<segment>.*), /$\{segment}
                - Retry=3

            - id: order-service-route
              uri: lb://order-service
              predicates:
                - Path=/api/orders/**
              filters:
                - name: RequestRateLimiter  # 限流
                  args:
                    redis-rate-limiter.replenishRate: 10
                    redis-rate-limiter.burstCapacity: 20
                    key-resolver: "#{@ipKeyResolver}"

          default-filters:  # 全局过滤器
            - AddResponseHeader=X-Gateway-Version, 1.0
    */

    // ==================== 2. Gateway 自定义过滤器 ====================

    /**
     * 全局过滤器 - 对所有路由生效
     */
    @Component
    public class AuthGlobalFilter implements GlobalFilter, Ordered {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            ServerHttpRequest request = exchange.getRequest();

            // 获取Token
            String token = request.getHeaders().getFirst("Authorization");

            if (StringUtils.isEmpty(token)) {
                // 未认证，返回401
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            // 验证Token
            if (!validateToken(token)) {
                exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                return exchange.getResponse().setComplete();
            }

            // 传递用户信息到下游服务
            ServerHttpRequest newRequest = request.mutate()
                .header("X-User-Id", getUserIdFromToken(token))
                .build();

            return chain.filter(exchange.mutate().request(newRequest).build());
        }

        @Override
        public int getOrder() {
            return -100;  // 优先级高
        }
    }

    /**
     * 局部过滤器工厂 - 针对特定路由
     */
    @Component
    public class LogGatewayFilterFactory extends
            AbstractGatewayFilterFactory<LogGatewayFilterFactory.Config> {

        public LogGatewayFilterFactory() {
            super(Config.class);
        }

        @Override
        public GatewayFilter apply(Config config) {
            return (exchange, chain) -> {
                long startTime = System.currentTimeMillis();

                return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                    long endTime = System.currentTimeMillis();
                    if (config.isEnabled()) {
                        log.info("请求路径: {}, 耗时: {}ms",
                            exchange.getRequest().getPath(),
                            endTime - startTime);
                    }
                }));
            };
        }

        @Data
        public static class Config {
            private boolean enabled = true;
        }
    }

    // 使用: - Log=true

    // ==================== 3. Gateway 断言工厂 ====================

    /**
     * 自定义断言工厂
     */
    @Component
    public class VipRoutePredicateFactory extends
            AbstractRoutePredicateFactory<VipRoutePredicateFactory.Config> {

        public VipRoutePredicateFactory() {
            super(Config.class);
        }

        @Override
        public Predicate<ServerWebExchange> apply(Config config) {
            return exchange -> {
                String userId = exchange.getRequest().getHeaders()
                    .getFirst("X-User-Id");
                // 检查是否是VIP用户
                return config.getVipUserIds().contains(userId);
            };
        }

        @Data
        public static class Config {
            private List<String> vipUserIds = new ArrayList<>();
        }
    }

    // 使用: - Vip=user1,user2,user3

    // ==================== 4. Gateway 限流配置 ====================

    @Configuration
    public class RateLimiterConfig {

        // 基于IP的限流Key
        @Bean
        public KeyResolver ipKeyResolver() {
            return exchange -> Mono.just(
                exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
            );
        }

        // 基于用户的限流Key
        @Bean
        public KeyResolver userKeyResolver() {
            return exchange -> Mono.just(
                exchange.getRequest().getHeaders().getFirst("X-User-Id")
            );
        }

        // 基于接口的限流Key
        @Bean
        public KeyResolver apiKeyResolver() {
            return exchange -> Mono.just(
                exchange.getRequest().getPath().value()
            );
        }
    }

    // ==================== 5. Zuul 配置（了解） ====================

    // Zuul配置示例
    /*
    zuul:
      routes:
        user-service:
          path: /api/users/**
          serviceId: user-service
          strip-prefix: true
        order-service:
          path: /api/orders/**
          serviceId: order-service

      # 敏感头设置
      sensitive-headers: Cookie,Set-Cookie,Authorization

      # 超时配置
      host:
        connect-timeout-millis: 5000
        socket-timeout-millis: 10000
    */

    // Zuul过滤器
    public class AuthZuulFilter extends ZuulFilter {

        @Override
        public String filterType() {
            return "pre";  // pre, route, post, error
        }

        @Override
        public int filterOrder() {
            return 0;
        }

        @Override
        public boolean shouldFilter() {
            return true;
        }

        @Override
        public Object run() {
            RequestContext ctx = RequestContext.getCurrentContext();
            HttpServletRequest request = ctx.getRequest();

            String token = request.getHeader("Authorization");
            if (StringUtils.isEmpty(token)) {
                ctx.setSendZuulResponse(false);
                ctx.setResponseStatusCode(401);
            }

            return null;
        }
    }

    // ==================== 6. 选型建议 ====================

    /*
     * 选择 Spring Cloud Gateway:
     * 1. 新项目，推荐使用
     * 2. 需要高性能（非阻塞IO）
     * 3. 需要WebSocket支持
     * 4. 使用Spring WebFlux技术栈
     *
     * 选择 Zuul 1.x（不推荐新项目）:
     * 1. 已有Zuul的存量项目
     * 2. 团队对Servlet更熟悉
     *
     * 注意: Zuul 2.x 使用Netty，但Spring Cloud没有集成
     */
}
```

---

### 119. Sentinel的原理是什么？

```java
/**
 * Sentinel 原理详解
 */
public class SentinelPrincipleDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      Sentinel 核心概念                                   │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  资源(Resource)    ──>  定义需要保护的代码块或方法                        │
    │                                                                          │
    │  规则(Rule)        ──>  流控规则、熔断规则、热点规则、系统规则            │
    │                                                                          │
    │  入口(Entry)       ──>  资源访问的入口，统计通过/拒绝数据                 │
    │                                                                          │
    │  槽(Slot)          ──>  责任链中的处理节点                                │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      Sentinel 处理流程（责任链）                          │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  Request ──> NodeSelectorSlot ──> ClusterBuilderSlot                   │
    │                    │                     │                               │
    │                    ▼                     ▼                               │
    │              LogSlot ──> StatisticSlot ──> AuthoritySlot               │
    │                              │                   │                       │
    │                              ▼                   ▼                       │
    │                        SystemSlot ──> FlowSlot ──> DegradeSlot         │
    │                                           │            │                 │
    │                                           ▼            ▼                 │
    │                                        通过/拒绝    熔断降级              │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 基本使用 ====================

    /*
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
    */

    // application.yml
    /*
    spring:
      cloud:
        sentinel:
          transport:
            dashboard: localhost:8080  # 控制台地址
            port: 8719                 # 与控制台通信端口
          eager: true                  # 立即加载
    */

    // ==================== 2. 注解方式使用 ====================

    @Service
    public class OrderService {

        @SentinelResource(
            value = "createOrder",           // 资源名
            blockHandler = "createOrderBlockHandler",    // 限流降级处理
            fallback = "createOrderFallback",            // 异常降级处理
            exceptionsToIgnore = {IllegalArgumentException.class}  // 忽略的异常
        )
        public Order createOrder(OrderRequest request) {
            // 业务逻辑
            return orderRepository.save(new Order(request));
        }

        // 限流/熔断时调用（参数需与原方法一致，多一个BlockException）
        public Order createOrderBlockHandler(OrderRequest request, BlockException ex) {
            log.warn("创建订单被限流: {}", ex.getClass().getSimpleName());
            return new Order("BLOCKED");
        }

        // 业务异常时调用（参数需与原方法一致，多一个Throwable）
        public Order createOrderFallback(OrderRequest request, Throwable ex) {
            log.error("创建订单异常: {}", ex.getMessage());
            return new Order("FALLBACK");
        }
    }

    // 处理方法也可以放在单独的类中
    public class OrderServiceBlockHandler {
        public static Order createOrderBlockHandler(OrderRequest request, BlockException ex) {
            return new Order("BLOCKED");
        }
    }

    // 使用：blockHandlerClass = OrderServiceBlockHandler.class

    // ==================== 3. 编程方式使用 ====================

    @Service
    public class ProductService {

        public Product getProduct(Long id) {
            Entry entry = null;
            try {
                // 获取资源入口
                entry = SphU.entry("getProduct");

                // 业务逻辑
                return productRepository.findById(id).orElse(null);

            } catch (BlockException ex) {
                // 被限流或熔断
                return new Product(id, "限流商品");
            } finally {
                if (entry != null) {
                    entry.exit();
                }
            }
        }
    }

    // ==================== 4. 规则配置 ====================

    @Configuration
    public class SentinelRuleConfig {

        @PostConstruct
        public void init() {
            // 流控规则
            initFlowRules();
            // 熔断规则
            initDegradeRules();
            // 热点参数规则
            initParamFlowRules();
            // 系统保护规则
            initSystemRules();
        }

        /**
         * 流控规则
         */
        private void initFlowRules() {
            List<FlowRule> rules = new ArrayList<>();

            FlowRule rule = new FlowRule();
            rule.setResource("createOrder");
            rule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS模式
            rule.setCount(100);  // 阈值100
            rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);  // 直接拒绝
            // 其他控制行为：WARM_UP（预热）、RATE_LIMITER（排队等待）

            rules.add(rule);
            FlowRuleManager.loadRules(rules);
        }

        /**
         * 熔断规则
         */
        private void initDegradeRules() {
            List<DegradeRule> rules = new ArrayList<>();

            DegradeRule rule = new DegradeRule();
            rule.setResource("getProduct");
            rule.setGrade(CircuitBreakerStrategy.ERROR_RATIO.getType());  // 异常比例
            rule.setCount(0.5);  // 50%异常率
            rule.setTimeWindow(10);  // 熔断时长10秒
            rule.setMinRequestAmount(5);  // 最小请求数
            rule.setStatIntervalMs(10000);  // 统计时长

            rules.add(rule);
            DegradeRuleManager.loadRules(rules);
        }

        /**
         * 热点参数规则
         */
        private void initParamFlowRules() {
            ParamFlowRule rule = new ParamFlowRule();
            rule.setResource("getProduct");
            rule.setParamIdx(0);  // 第一个参数
            rule.setCount(10);  // 每个参数值QPS上限

            // 特殊值例外
            ParamFlowItem item = new ParamFlowItem();
            item.setObject("hot_product_id");
            item.setClassType(String.class.getName());
            item.setCount(1000);  // 热门商品放宽限制
            rule.setParamFlowItemList(Collections.singletonList(item));

            ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
        }

        /**
         * 系统保护规则
         */
        private void initSystemRules() {
            SystemRule rule = new SystemRule();
            rule.setHighestSystemLoad(10);      // 系统负载阈值
            rule.setAvgRt(100);                 // 平均RT阈值
            rule.setMaxThread(500);             // 最大线程数
            rule.setQps(1000);                  // 入口QPS阈值
            rule.setHighestCpuUsage(0.8);       // CPU使用率阈值

            SystemRuleManager.loadRules(Collections.singletonList(rule));
        }
    }

    // ==================== 5. 核心源码分析 ====================

    /**
     * 责任链模式 - Slot处理器
     */
    public interface ProcessorSlot<T> {

        // 入口处理
        void entry(Context context, ResourceWrapper resourceWrapper, T param,
                   int count, boolean prioritized, Object... args) throws Throwable;

        // 出口处理
        void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);
    }

    /**
     * FlowSlot - 流控核心
     */
    public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

        @Override
        public void entry(Context context, ResourceWrapper resourceWrapper,
                         DefaultNode node, int count, boolean prioritized, Object... args)
                throws Throwable {

            // 检查流控规则
            checkFlow(resourceWrapper, context, node, count, prioritized);

            // 传递给下一个Slot
            fireEntry(context, resourceWrapper, node, count, prioritized, args);
        }

        void checkFlow(ResourceWrapper resource, Context context,
                      DefaultNode node, int count, boolean prioritized)
                throws BlockException {

            // 获取资源对应的流控规则
            Collection<FlowRule> rules = FlowRuleManager.getRules(resource.getName());

            for (FlowRule rule : rules) {
                if (!canPassCheck(rule, context, node, count, prioritized)) {
                    throw new FlowException(rule.getLimitApp(), rule);
                }
            }
        }
    }

    /**
     * StatisticSlot - 统计核心（滑动窗口）
     */
    public class StatisticSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

        @Override
        public void entry(Context context, ResourceWrapper resourceWrapper,
                         DefaultNode node, int count, boolean prioritized, Object... args)
                throws Throwable {

            try {
                // 先执行后续Slot
                fireEntry(context, resourceWrapper, node, count, prioritized, args);

                // 通过，增加通过数
                node.increaseThreadNum();
                node.addPassRequest(count);

            } catch (BlockException e) {
                // 被拒绝，增加拒绝数
                node.increaseBlockQps(count);
                throw e;
            }
        }

        @Override
        public void exit(Context context, ResourceWrapper resourceWrapper,
                        int count, Object... args) {
            DefaultNode node = (DefaultNode) context.getCurNode();

            // 记录RT
            if (context.getCurEntry().getError() == null) {
                long rt = TimeUtil.currentTimeMillis() - context.getCurEntry().getCreateTime();
                node.addRtAndSuccess(rt, count);
            } else {
                // 记录异常
                node.increaseExceptionQps(count);
            }

            // 减少线程数
            node.decreaseThreadNum();
        }
    }

    // ==================== 6. 滑动窗口统计 ====================

    /*
     * Sentinel使用滑动窗口统计QPS
     *
     * 默认：1秒分成2个窗口，每个窗口500ms
     *
     * |------ 1秒 ------|
     * | 窗口1  | 窗口2  |
     * | 500ms | 500ms |
     *
     * 窗口滑动时，丢弃最老的窗口，创建新窗口
     */
}
```

---

### 120. Config配置中心的原理？

```java
/**
 * Spring Cloud Config / Nacos Config 原理
 */
public class ConfigCenterPrincipleDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      配置中心架构                                        │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │   ┌──────────────────────────────────────────────────────────────┐     │
    │   │                    Config Server                             │     │
    │   │   ┌─────────┐   ┌─────────┐   ┌─────────┐                   │     │
    │   │   │   Git   │   │  本地    │   │  数据库  │   <-- 配置存储   │     │
    │   │   └─────────┘   └─────────┘   └─────────┘                   │     │
    │   └────────────────────────┬─────────────────────────────────────┘     │
    │                            │ HTTP/长连接                                │
    │          ┌─────────────────┼─────────────────┐                         │
    │          │                 │                 │                         │
    │          ▼                 ▼                 ▼                         │
    │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
    │   │  Service A   │  │  Service B   │  │  Service C   │   <-- 客户端  │
    │   └──────────────┘  └──────────────┘  └──────────────┘                │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. Spring Cloud Config ====================

    // Config Server
    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    */

    @SpringBootApplication
    @EnableConfigServer
    public class ConfigServerApplication {
        public static void main(String[] args) {
            SpringApplication.run(ConfigServerApplication.class, args);
        }
    }

    // Config Server配置
    /*
    server:
      port: 8888

    spring:
      cloud:
        config:
          server:
            git:
              uri: https://github.com/example/config-repo
              search-paths: '{application}'
              username: xxx
              password: xxx
              clone-on-start: true
              default-label: main
    */

    // Config Client
    /*
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    */

    // bootstrap.yml（Config Client）
    /*
    spring:
      application:
        name: user-service
      profiles:
        active: dev
      cloud:
        config:
          uri: http://localhost:8888
          label: main
          fail-fast: true
    */

    // ==================== 2. Nacos Config ====================

    /*
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    */

    // bootstrap.yml
    /*
    spring:
      application:
        name: user-service
      profiles:
        active: dev
      cloud:
        nacos:
          config:
            server-addr: localhost:8848
            file-extension: yaml
            namespace: dev
            group: DEFAULT_GROUP
            refresh-enabled: true
            # 扩展配置
            extension-configs:
              - data-id: common.yaml
                group: DEFAULT_GROUP
                refresh: true
            # 共享配置
            shared-configs:
              - data-id: shared.yaml
                group: DEFAULT_GROUP
                refresh: true
    */

    // 配置文件命名规则：${spring.application.name}-${spring.profiles.active}.${file-extension}
    // 例如：user-service-dev.yaml

    // ==================== 3. 配置动态刷新 ====================

    /**
     * @RefreshScope 实现配置热更新
     */
    @RefreshScope  // 配置变化时重新创建Bean
    @RestController
    public class ConfigController {

        @Value("${app.config.name}")
        private String configName;

        @Value("${app.config.timeout:5000}")
        private int timeout;

        @GetMapping("/config")
        public Map<String, Object> getConfig() {
            Map<String, Object> config = new HashMap<>();
            config.put("name", configName);
            config.put("timeout", timeout);
            return config;
        }
    }

    /**
     * 使用@ConfigurationProperties自动刷新
     */
    @Data
    @ConfigurationProperties(prefix = "app.config")
    @RefreshScope
    public class AppConfig {
        private String name;
        private int timeout;
        private List<String> servers;
    }

    // ==================== 4. Nacos配置监听 ====================

    @Component
    public class NacosConfigListener {

        @NacosConfigListener(dataId = "user-service-dev.yaml", groupId = "DEFAULT_GROUP")
        public void onConfigChange(String newConfig) {
            log.info("配置发生变化: {}", newConfig);
            // 处理配置变化
        }
    }

    // 编程方式监听
    @Component
    public class ConfigChangeListener implements ApplicationRunner {

        @Autowired
        private NacosConfigManager nacosConfigManager;

        @Override
        public void run(ApplicationArguments args) throws Exception {
            ConfigService configService = nacosConfigManager.getConfigService();

            configService.addListener("user-service-dev.yaml", "DEFAULT_GROUP",
                new Listener() {
                    @Override
                    public Executor getExecutor() {
                        return null;
                    }

                    @Override
                    public void receiveConfigInfo(String configInfo) {
                        log.info("收到配置变更: {}", configInfo);
                    }
                });
        }
    }

    // ==================== 5. 配置加载原理 ====================

    /**
     * 配置加载流程
     */
    /*
     * 1. 应用启动
     * 2. Bootstrap阶段
     *    - 加载bootstrap.yml/properties
     *    - 创建Bootstrap ApplicationContext
     *    - PropertySourceLocator加载远程配置
     * 3. 合并配置
     *    - 远程配置 + 本地配置
     *    - 优先级：远程 > bootstrap > application
     * 4. 创建主ApplicationContext
     * 5. 刷新配置（@RefreshScope）
     */

    /**
     * PropertySourceLocator接口
     */
    public interface PropertySourceLocator {
        PropertySource<?> locate(Environment environment);
    }

    // Nacos实现
    public class NacosPropertySourceLocator implements PropertySourceLocator {

        @Override
        public PropertySource<?> locate(Environment environment) {
            // 1. 获取Nacos配置
            ConfigService configService = nacosConfigManager.getConfigService();

            // 2. 拉取配置
            String config = configService.getConfig(dataId, group, timeout);

            // 3. 解析配置
            Map<String, Object> properties = parseYaml(config);

            // 4. 创建PropertySource
            return new MapPropertySource("nacos", properties);
        }
    }

    // ==================== 6. 配置加密 ====================

    // 使用jasypt加密
    /*
    <dependency>
        <groupId>com.github.ulisesbocchio</groupId>
        <artifactId>jasypt-spring-boot-starter</artifactId>
    </dependency>
    */

    // 加密后的配置
    /*
    spring:
      datasource:
        password: ENC(encrypted_password)

    jasypt:
      encryptor:
        password: ${JASYPT_ENCRYPTOR_PASSWORD}  # 解密密钥
        algorithm: PBEWITHHMACSHA512ANDAES_256
    */

    // 加密工具
    public class JasyptEncryptUtil {
        public static void main(String[] args) {
            StandardPBEStringEncryptor encryptor = new StandardPBEStringEncryptor();
            encryptor.setPassword("your-secret-key");
            encryptor.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");

            String encrypted = encryptor.encrypt("your-password");
            System.out.println("Encrypted: " + encrypted);
        }
    }

    // ==================== 7. 多环境配置管理 ====================

    /*
     * Nacos命名空间隔离：
     *
     * ┌─────────────────────────────────────────────┐
     * │  Nacos                                      │
     * │  ┌─────────┐ ┌─────────┐ ┌─────────┐      │
     * │  │   dev   │ │  test   │ │  prod   │  <-- namespace
     * │  │ ─────── │ │ ─────── │ │ ─────── │      │
     * │  │ group1  │ │ group1  │ │ group1  │  <-- group
     * │  │ group2  │ │ group2  │ │ group2  │      │
     * │  └─────────┘ └─────────┘ └─────────┘      │
     * └─────────────────────────────────────────────┘
     */
}
```

---

### 121. Sleuth链路追踪的原理？

```java
/**
 * Spring Cloud Sleuth 链路追踪原理
 */
public class SleuthPrincipleDemo {

    /*
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      链路追踪核心概念                                    │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  Trace (追踪)                                                            │
    │  └── 一次完整请求的追踪，包含多个Span                                     │
    │                                                                          │
    │  Span (跨度)                                                             │
    │  └── 一次RPC调用或本地操作，是追踪的基本单元                               │
    │                                                                          │
    │  TraceId                                                                 │
    │  └── 追踪ID，全局唯一，贯穿整个调用链                                     │
    │                                                                          │
    │  SpanId                                                                  │
    │  └── 跨度ID，每个Span唯一                                                │
    │                                                                          │
    │  ParentSpanId                                                            │
    │  └── 父SpanId，用于构建调用关系                                          │
    │                                                                          │
    │  Annotation                                                              │
    │  └── 时间戳记录：cs(Client Send), sr(Server Receive),                   │
    │       ss(Server Send), cr(Client Receive)                               │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      调用链示例                                          │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                          │
    │  Browser ──> Gateway ──> Service A ──> Service B ──> Service C         │
    │                                                                          │
    │  TraceId: abc123 (全链路相同)                                            │
    │                                                                          │
    │  Gateway:    SpanId=1, ParentId=null                                    │
    │  Service A:  SpanId=2, ParentId=1                                       │
    │  Service B:  SpanId=3, ParentId=2                                       │
    │  Service C:  SpanId=4, ParentId=3                                       │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
    */

    // ==================== 1. 基本使用 ====================

    /*
    <!-- Sleuth -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>

    <!-- Zipkin Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    </dependency>
```

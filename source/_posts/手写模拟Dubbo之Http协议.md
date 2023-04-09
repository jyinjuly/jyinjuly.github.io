---
title: 手写模拟Dubbo之Http协议
tags: ["Dubbo", "RPC"]
categories: ["Dubbo"]
---
# 前言
我们知道Dubbo的通信协议包含http和dubbo，两者的简单区别如下：
| Dubbo通信协议 | 数据传输协议 | 请求处理方案 |
| ---- | ---- | ---- |
| http | http | tomcat |
| dubbo | tcp | netty |

本篇手写模拟Dubbo的http通信协议（基于JDK 11）。

# 整体项目布局
创建maven项目，引入依赖：
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-core</artifactId>
        <version>9.0.12</version>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.51</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.4</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-io</artifactId>
        <version>1.3.2</version>
    </dependency>
</dependencies>
```
创建如下目录结构及文件:
![dubbo目录.png](dubbo目录.png)

```java
/**
 * 服务消费者
 */
public class Consumer {

    public static void main(String[] args) {

    }

}
```
```java
/**
 * Dubbo调用数据传输
 */
@Data
@AllArgsConstructor
public class Invocation {

    private String interfaceName;
    private String methodName;
    private Class[] paramTypes;
    private Object[] params;

}
```
```java
/**
 * 负载均衡
 */
public class LoadBalance {

    /**
     * 随机策略
     *
     * @param urlList 服务地址列表
     * @return
     */
    public static URL random(List<URL> urlList) {
        return urlList.get(new Random().nextInt(urlList.size()));
    }

}
```
```java
/**
 * 服务地址
 */
@Data
@AllArgsConstructor
public class URL implements Serializable {

    private String hostname;
    private Integer port;

}
```
```java
/**
 * 服务提供者：
 * 1、提供接口
 * 2、提供实现类
 * 3、暴露服务（启动Tomcat，从而接收并处理请求）
 * 4、注册服务（本地注册、注册中心注册）
 */
public class Provider {

    public static void main(String[] args) {

    }

}
```
# 服务提供者
## 提供接口及实现类
![接口和实现.png](接口和实现.png)
```java
/**
 * 服务接口
 */
public interface HelloService {

    String sayHello(String name);

}
```
```java
/**
 * 服务实现类
 */
public class HelloServiceImpl implements HelloService {

    public String sayHello(String name) {
        return "你好，" + name;
    }

}
```
## 注册服务（本地注册）
![LocalRegister.png](LocalRegister.png)
```java
/**
 * 本地注册中心
 */
public class LocalRegister {

    private static Map<String, Class> localRegister = new HashMap<>();

    /**
     * 服务注册
     *
     * @param interfaceName 接口名
     * @param implClass 实现类
     */
    public static void register(String interfaceName, Class implClass) {
        localRegister.put(interfaceName, implClass);
    }

    public static Class get(String interfaceName) {
        return localRegister.get(interfaceName);
    }

}
```
```java
public class Provider {

    public static void main(String[] args) {

        // 本地注册
        LocalRegister.register(HelloService.class.getName(), HelloServiceImpl.class);

    }

}
```
## 暴露服务
### 启动Tomcat
![HttpServer.png](HttpServer.png)
```java
public class HttpServer {

    /**
     * 启动Tomcat
     *
     * @param hostname ip地址
     * @param port 端口号
     */
    public void start(String hostname, Integer port) {
        Tomcat tomcat = new Tomcat();
        Server server = tomcat.getServer();
        Service service = server.findService("Tomcat");

        Connector connector = new Connector();
        connector.setPort(port);

        Engine engine = new StandardEngine();
        engine.setDefaultHost(hostname);

        Host host = new StandardHost();
        host.setName(hostname);

        String contextPath = "";
        Context context = new StandardContext();
        context.setPath(contextPath);
        context.addLifecycleListener(new Tomcat.FixContextListener());

        host.addChild(context);
        engine.addChild(host);

        service.setContainer(engine);
        service.addConnector(connector);

        // 指定请求分发器
        tomcat.addServlet(contextPath, "dispatcher", new DispatcherServlet());
        // 将全部请求路由到上述指定的请求分发器
        context.addServletMappingDecoded("/*", "dispatcher");

        try {
            tomcat.start();
            tomcat.getServer().await();
        } catch (LifecycleException e) {
            e.printStackTrace();
        }
    }

}
```
```java
public class Provider {

    public static void main(String[] args) {

        // 本地注册
        LocalRegister.register(HelloService.class.getName(), HelloServiceImpl.class);

        // 启动Tomcat，暴露服务
        URL url = new URL("localhost", 8080);
        HttpServer httpServer = new HttpServer();
        httpServer.start(url.getHostname(), url.getPort());
        
    }

}
```
### 请求分发器
![DispatcherServlet.png](DispatcherServlet.png)
```java
/**
 * 请求分发器
 */
public class DispatcherServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        new HttpServerHandler().handle(req, resp);
    }

}
```
### 请求处理器
![HttpServerHandler.png](HttpServerHandler.png)
```java
/**
 * 请求处理器
 */
public class HttpServerHandler {

    /**
     * 1、从请求实体中获取{@link com.liteng.framework.Invocation}
     * 2、根据接口名在本地注册中心获取实现类
     * 3、通过反射进行方法调用
     * 4、将结果写进响应实体
     *
     * @param req 请求实体
     * @param resp 响应实体
     */
    public void handle(HttpServletRequest req, HttpServletResponse resp) {
        try {
            Invocation invocation = JSONObject.parseObject(req.getInputStream(), Invocation.class);

            Class implClass = LocalRegister.get(invocation.getInterfaceName());
            Method method = implClass.getMethod(invocation.getMethodName(), invocation.getParamTypes());
            String result = (String) method.invoke(implClass.getDeclaredConstructor().newInstance(), invocation.getParams());

            IOUtils.write(result, resp.getOutputStream());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```
# 服务消费者
## http请求客户端
![HttpClient.png](HttpClient.png)
```java
public class HttpClient {

    /**
     * 发送http请求
     *
     * @param hostname ip地址
     * @param port 端口号
     * @param invocation 请求体
     * @return
     */
    public String send(String hostname, Integer port, Invocation invocation) {

        try {
            HttpRequest httpRequest = HttpRequest.newBuilder()
                    .uri(new URI("http", null, hostname, port, "/", null, null))
                    .POST(HttpRequest.BodyPublishers.ofString(JSONObject.toJSONString(invocation)))
                    .build();
            java.net.http.HttpClient httpClient = java.net.http.HttpClient.newHttpClient();
            return httpClient.send(httpRequest, HttpResponse.BodyHandlers.ofString()).body();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;

    }

}
```
## 服务消费者发起请求
```java
/**
 * 服务消费者
 */
public class Consumer {

    public static void main(String[] args) {

        Invocation invocation = new Invocation(
                HelloService.class.getName(), "sayHello",
                new Class[]{String.class}, new Object[]{"张三"});
        
        HttpClient httpClient = new HttpClient();
        String result = httpClient.send("localhost", 8080, invocation);

        System.out.println(result); // 你好，张三
    }

}
```
# Break and Think
好的，到这里服务消费者已经可以成功调通服务提供者，但是有什么问题吗？不难发现，有以下问题：
1. 服务消费者调用方式不太友好，貌似和直接发送http请求没太大差别
2. ip地址和端口号是写死的，况且服务消费者也不需要知道这俩玩意具体是什么
3. 注册中心怎么也没有用到啊

OK，问题有了，一个一个解决。
## 服务消费者通过代理对象发起调用
我们知道，RPC调用远程方法就好像调用本地方法一样方便。因此，服务消费者期望的调用方式如下：
```java
/**
 * 服务消费者
 */
public class Consumer {

    public static void main(String[] args) {
        
        HelloService helloService = xxx; // 通过某种特殊方式获取一个对象
        String result = helloService.sayHello("张三");
        System.out.println(result);

    }
}
```
这种特殊的方式就是JDK动态代理（Dubbo也是这样做的）：
![ProxyFactory.png](ProxyFactory.png)
```java
/**
 * 代理工厂
 */
public class ProxyFactory {

    /**
     * 获取代理对象
     *
     * @param interfaceClass 接口类
     * @param <T>
     * @return
     */
    public static <T> T getProxy(final Class<?> interfaceClass) {
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class[]{interfaceClass},
                (proxy, method, args) -> {
            Invocation invocation = new Invocation(
                    interfaceClass.getName(), method.getName(),
                    method.getParameterTypes(), args);

            HttpClient httpClient = new HttpClient();
            return httpClient.send("localhost", 8080, invocation);
        });
    }

}
```
此时的服务消费者调用如下：
```java
/**
 * 服务消费者
 */
public class Consumer {

    public static void main(String[] args) {

        HelloService helloService = ProxyFactory.getProxy(HelloService.class);
        String result = helloService.sayHello("张三");
        System.out.println(result);
        
    }

}
```
## 注册中心实现服务地址共享
上述问题2,3均可以通过注册中心解决。对照着通过代理对象发起调用的方式，我们思考一下：服务消费者在发起调用时关注点是什么？关注服务是由谁提供的吗？不，它只需要知道自己要调用哪个服务，并且正确传参就OK了。那么服务由谁提供是谁负责管理呢？服务消费者怎么就能准确地将请求发送到服务提供者对应的服务地址呢？这些都依托于注册中心！

### 注册中心
常用的注册中心有Zookeeper、Redis及Nacos等，这里我们为了方便演示，使用本地文件的方式来实现：
![RemoteRegister.png](RemoteRegister.png)
```java
/**
 * 注册中心
 */
public class RemoteRegister {

    private static Map<String, List<URL>> remoteRegister = new HashMap<>();

    /**
     * 服务注册
     * 
     * @param interfaceName 服务名
     * @param url 服务地址
     */
    public static void register(String interfaceName, URL url) {
        List<URL> urlList = remoteRegister.get(interfaceName);
        if (urlList == null) {
            urlList = new ArrayList<>();
        }
        urlList.add(url);
        remoteRegister.put(interfaceName, urlList);

        writeFile();
    }

    public static List<URL> get(String interfaceName) {
        remoteRegister = readFile();
        return remoteRegister.get(interfaceName);
    }

    private static void writeFile() {
        try {
            String file = new File("").getCanonicalPath() + "\\temp.txt";
            FileOutputStream fileOutputStream = new FileOutputStream(file);
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
            objectOutputStream.writeObject(remoteRegister);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static Map<String, List<URL>> readFile() {
        try {
            String file = new File("").getCanonicalPath() + "\\temp.txt";
            FileInputStream fileInputStream = new FileInputStream(file);
            ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
            return (Map<String, List<URL>>) objectInputStream.readObject();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;

    }
    
}
```
### 注册服务（注册中心注册）
服务提供者在启动的时候，向注册中心注册服务
```java
public class Provider {

    public static void main(String[] args) {

        // 本地注册
        LocalRegister.register(HelloService.class.getName(), HelloServiceImpl.class);

        URL url = new URL("localhost", 8080);
        // 注册中心注册
        RemoteRegister.register(HelloService.class.getName(), url);

        // 启动Tomcat，暴露服务
        HttpServer httpServer = new HttpServer();
        httpServer.start(url.getHostname(), url.getPort());

    }

}
```
### 代理对象通过注册中心获取服务地址
```java
public class ProxyFactory {

    /**
     * 获取代理对象
     *
     * @param interfaceClass 接口类
     * @param <T>
     * @return
     */
    public static <T> T getProxy(final Class<?> interfaceClass) {
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class[]{interfaceClass},
                (proxy, method, args) -> {
            Invocation invocation = new Invocation(
                    interfaceClass.getName(), method.getName(),
                    method.getParameterTypes(), args);

            URL url = LoadBalance.random(RemoteRegister.get(interfaceClass.getName()));

            HttpClient httpClient = new HttpClient();
            return httpClient.send(url.getHostname(), url.getPort(), invocation);
        });
    }

}
```











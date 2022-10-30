---
title: Zookeeper分布式锁应用——秒杀活动
tags: ["Zookeeper", "分布式锁"]
categories: ["Zookeeper"]
---
# 场景描述
淘宝双十一我们都知道，数以亿计的买家在同一时间去争抢有限数量的商品，也就是所谓的商品秒杀活动。面对如此高的并发，如果我们不采取一些措施，将会引发一系列问题，比如：商品超卖、单机锁的失效等。
## 商品超卖
假设有100个用户同时去抢一个商品，并且全部下单成功，但是由于并发问题没有得到有效处理，导致实际库存只扣减了99个，也就是发生了1个商品的超卖。
## 单机锁的失效
如果我们的服务是单实例的话，可以通过加单机锁来实现对并发问题的处理。这里的单机锁指的是JVM进程级别的锁，比如关键字`synchronized`、juc并发包下的工具类等等。但如果是分布式系统的话，服务通常是以多实例的姿态出现的，这时候单机锁将会失效。
<!-- more -->
# 场景模拟
## 单机环境搭建
这里我们就简单一点，直接用IDE启动一个SpringBoot项目，提供一个外部调用的REST接口代表商品下单即可，具体代码请参见`https://github.com/jyinjuly/zookeeper/tree/master/zookeeper-distributed-lock`。这里我们重点看一下Controller层接口，如下：
```java
@Slf4j
@RestController
@RequestMapping("/product")
public class ProductController {

    @Autowired
    private ProductService productService;

    @GetMapping("/deduct")
    public void deduct() throws Exception {
        Product product = productService.selectProduct(1L);
        log.info("当前库存：{}", product.getNumber());
        if (product.getNumber() > 0) {
            product.setNumber(product.getNumber()-1);
            productService.updateProduct(product);
            log.info("出库成功，剩余库存：{}", product.getNumber());
        }
    }

}
```
接口逻辑很简单，首先查询商品，如果商品还有库存，则去做扣减库存操作。
## 复现商品超卖
### jmeter模拟并发请求
jmeter的安装这里就不做说明了，请各位读者小伙伴自行百度。我们只介绍本次模拟的相关参数，如下：
![线程组1.png](线程组1.png)
![HTTP请求1.png](HTTP请求1.png)
### 请求结果分析
首先，初始库存如下：
![初始库存.png](初始库存.png)
并发请求之后的库存如下：
![超卖后库存.png](超卖后库存.png)
什么意思？客户下单成功了100个商品，实际库存却只扣减了3个！什么原因导致的呢？分析如下：
![超卖原因分析.png](超卖原因分析.png)
究其原因是代码没有针对并发场景做处理导致的。
## 单机锁解决商品超卖
针对以上单机环境的并发问题，可以通过加JVM锁处理。如下：
```java
@GetMapping("/deduct")
public void deduct() throws Exception {
    // 单机版加锁
    synchronized (ProductController.class) {
        Product product = productService.selectProduct(1L);
        log.info("当前库存：{}", product.getNumber());
        if (product.getNumber() > 0) {
            product.setNumber(product.getNumber()-1);
            productService.updateProduct(product);
            log.info("出库成功，剩余库存：{}", product.getNumber());
        }
    }
}
```
首先，初始库存如下：
![初始库存.png](初始库存.png)
并发请求之后的库存如下：
![剩余库存.png](剩余库存.png)
## 分布式环境搭建
这里我们通过IDE并行启动两个服务实例，端口号分别为8080和8081。然后通过（本地）Nginx将请求反向代理到这两个实例，实现简单的分布式系统。
### Nginx配置
![Nginx反向代理.png](Nginx反向代理.png)
### 服务并行启动
首先配置服务允许并行启动，如下：
![服务并行启动.png](服务并行启动.png)
然后手动修改端口号，分别启动8080服务实例和8081服务实例。
## 单机锁失效
有了分布式环境，我们再去测试一下单机锁还能否解决商品超卖的问题，首先修改一下jmeter的HTTP请求，如下：
![HTTP请求2.png](HTTP请求2.png)
然后模拟并发请求，结果如下，首先，初始库存：
![初始库存.png](初始库存.png)
并发请求之后的库存：
![分布式超卖后库存.png](分布式超卖后库存.png)
结果显而易见，单机锁在分布式环境下失效了。
# Zookeeper分布式锁
分布式锁有很多种实现，比如Redis分布式锁、Zookeeper分布式锁等等，这里我们使用的是Zookeeper版本，而且还是Curator框架封装好的工具类。
```java
@GetMapping("/deduct")
public void deduct() throws Exception {
    // zookeeper分布式锁
    InterProcessMutex processMutex = new InterProcessMutex(curatorFramework, "/product_1");

    try {
        processMutex.acquire();
        Product product = productService.selectProduct(1L);
        log.info("当前库存：{}", product.getNumber());
        if (product.getNumber() > 0) {
            product.setNumber(product.getNumber()-1);
            productService.updateProduct(product);
            log.info("出库成功，剩余库存：{}", product.getNumber());
        }
    } finally {
        processMutex.release();
    }

}
```
然后模拟并发请求，结果如下，首先，初始库存：
![初始库存.png](初始库存.png)
并发请求之后的库存：
![剩余库存.png](剩余库存.png)
由此可见，使用分布式锁，分布式环境下的并发问题得以解决。
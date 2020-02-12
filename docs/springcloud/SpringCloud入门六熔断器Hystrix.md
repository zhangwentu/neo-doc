1.简介

Hystrix也是Netflix套件的一部分。他的功能是，当对某个服务的调用在一定的时间内（默认10s，由metrics.rollingStats.timeInMilliseconds配置），有超过一定次数（默认20次，由circuitBreaker.requestVolumeThreshold参数配置）并且失败率超过一定值（默认50%，由circuitBreaker.errorThresholdPercentage配置），该服务的断路器会打开。返回一个由开发者设定的fallback

这个fallback可以是固定的返回值也可以是一个替代服务,保证上级服务可以得到一个反馈，而不是单纯的抛出异常

2.配置文件追加配置，开启熔断

```
feign:
  hystrix:
    enabled: true #开启熔断
```

有的版本默认就是开启的，但是Greenwich.RELEASE版本默认关闭，这里必须加上这个配置

3.给之前feign client增加fallback

```java
package com.neo.customer.service;

import com.neo.customer.service.hystrix.ProducerServiceHystric;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/2/11 17:11
 */
@FeignClient(name = "producer-server", fallback = ProducerServiceHystric.class)
public interface IFeignService {
    @RequestMapping("/hello")
    String hello(@RequestParam(name = "name") String name);
}
```

```java
package com.neo.customer.service.hystrix;

import com.neo.customer.service.IFeignService;
import org.springframework.stereotype.Component;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/2/12 16:43
 */
@Component
public class ProducerServiceHystric implements IFeignService {
    @Override
    public String hello(String name) {
        return "sorry " + name;
    }
}
```

4.DEMO

之前的单元测试又可以拿出来再跑一次了，跑之前记得把ProducerSerbver关闭

```java
@Test
    public void handle() {
        String res = iFeignService.hello("test");
        log.info(res);
    }
```

实现效果：

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbtq9bqhe8j30ke02wmxa.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbtq9bqhe8j30ke02wmxa.jpg" width=""/>
    </a>
</p>
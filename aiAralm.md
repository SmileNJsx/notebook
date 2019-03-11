# AI模型管理服务组件性能测试

## 图片告警性能测试

### 前言

图片告警性能测试，不考虑中间件额外的消耗时间以及非组件产生的性能问题。

### 环境

+ window10 企业版本 i5处理器 8G内存 

### 限制

+ 硬件条件 _(见环境)_
+ 软件架构体系 _(公司架构体系)_

### QPS 

无

### TPS

#### 中间环节退出

服务器端响应请求不需要处理完整的事务链路，在事务处理过程中条件退出。

+ 客户端发送信息消耗时间
```
<!-- 500 record -->
Start Time: Mon Mar 11 11:18:07 CST 2019
End Time: Mon Mar 11 11:18:08 CST 2019
Cost Time: 1111ms
Total data record: 500

<!-- 5000 record -->
Start Time: Mon Mar 11 13:55:30 CST 2019
End Time: Mon Mar 11 13:55:37 CST 2019
Cost Time: 6673ms
Total data record: 5000

```

+ 服务端响应信息消耗时间
```
<!-- 500 record -->
Start Time: Mon Mar 11 11:18:07 CST 2019
End Time: Mon Mar 11 11:18:12 CST 2019

<!-- 5000 record -->
Start Time: Mon Mar 11 13:55:30 CST 2019
Start Time: Mon Mar 11 13:56:15 CST 2019
```
+ 分析对比
<!-- 500 record -->
客户端第07秒开始发送信息，服务端第07秒开始接受信息，发送和接受接近相同时间 _(说明消息中间件此时处理性能良好，服务端并发不高)_;

客户端第08秒500条消息发送完毕，大约耗时1111ms _(其实平均耗时只有800ms左右)_;

服务端第12秒500条非全链路信息处理完毕，大约耗时5s，TPS:100tps;

<!-- 5000 record -->

客户端第30秒开始发送信息，服务端第37秒开始接受信息，耗时7秒单线程可优化 _(说明消息中间件此时处理性能良好，服务端并发不高)_;

客户端第37秒5000条消息发送完毕，大约耗时6673ms;

服务端第15秒5000条非全链路信息处理完毕，大约耗时45s，TPS:111.11tps;

#### 全链路业务

+ 服务端响应
```
<!-- 500 record -->
Start Time: Mon Mar 11 15:16:58 CST 2019
End Time: Mon Mar 11 15:17:09 CST 2019

<!-- 5000 record -->
Start Time: Mon Mar 11 15:17:37 CST 2019
End Time: Mon Mar 11 15:19:18 CST 2019
```

500条 11秒 45.45 tps
5000条 101秒 50 tps

全链路业务 大约 TPS: 47.5tps;


### 源码 测试客户端

```java
package com.bobac.activemqclient;

import org.apache.activemq.ActiveMQConnectionFactory;

import javax.jms.*;
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class Client {

//    Start Time: Mon Mar 11 11:07:35 CST 2019
//    Start Time: Mon Mar 11 11:07:37 CST 2019
    public static void main(String[] args) throws JMSException {
        ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory("admin", "MZgtBafZ", "tcp://10.33.31.160:7018");
        Connection connection = activeMQConnectionFactory.createConnection();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Destination destination = session.createTopic("aimms.aimmsmq.topic.aimms.iac.detection.data");
//        MessageConsumer messageConsumer = session.createConsumer(destination);
//
//
//        messageConsumer.setMessageListener(new MessageListener() {
//            public void onMessage(Message message) {
//                try {
//                    System.out.println(((TextMessage) message).getText());
//                } catch (JMSException e) {
//                    e.printStackTrace();
//                }
//            }
//        });

        MessageProducer producer = session.createProducer(destination);
        connection.start();
        long start = System.currentTimeMillis();
        System.out.println("Start Time: " + new Date(System.currentTimeMillis()));
        int i = 0;
        for (; i < 500; i++) {
            producer.send(session.createTextMessage(“”));
//            System.out.println(i);
        }
        System.out.println("End Time: " + new Date(System.currentTimeMillis()));
        long end = System.currentTimeMillis();
        System.out.println("Cost Time: " + TimeUnit.MILLISECONDS.toMillis(end - start) + "ms");
        System.out.println("Total data record: " + i);
        connection.close();
    }
}

```

### 结论

限制环境下达 75.5tps

测试显示报警处理本环境下单机可达 100tps，多机集群 _(假设十台)_，理想情况下也只达 1000tps，优化空间很大;

期望预期目标优化目标单机可达 100000 tps _(此处目标需要定义标准的硬件能力和软件中间件支持后续研究)_;

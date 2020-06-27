---
title: 基于Flink设计统一报警平台
date: 2020-05-16 17:24:48
tags: 
    - Java
    - Flink
    - 监控
categories :
    - Technology
---

最近在整理后端已有的错误报警，主要分两大类：
1. 异常报警
   - 基于监控系统。如CAT、pingpong、skywalking等APM系统的错误异常监控。开发人员针对异常类型，制定相应的阈值。对代码运行时异常、sql异常、网络异常等进行指标监控。当异常超过指定阈值时推送异常信息。我们的后端系统是基于CAT进行监控，监控的指标和配置都要人为进行配置，异常信息的推送需要开发并与CAT对接，同时微信推送这块异常信息有限，往往要通过邮件告警和登录CAT平台查看才能大概了解错误的类型。这点长期被开发人员诟病，虽然我们能够通过修改CAT的源码来改善上诉问题。但对于程序员来说，更加直观的堆错误信息进行处理是再好不过的事情。
   - 基于系统指标。有别于服务异常，系统指标有更灵活的采集方式。我们采用了Spring Merics，基于Prometheus搭建了系统指标的监控体系，结合Grafana将近实时的系统指标展现至屏幕。同样遇到系统指标异常的时候，如Heap空间使用量异常，CPU飙高，内存告急等情况，我们通过Grafana的自定义告警将错误推送给相关开发。
  <!-- more -->
2. 业务告警
   - 基于实时数据。通过对业务流程的埋点，将采集的实时数据进行计算比较。并将异常的业务数据进行推送告警。
   - 基于离线数据。定时分析离线数据，推送异常业务数据告警。

以上大概介绍了监控的几个方向及其落地方式，通过分析我们可以将整个监控行为抽象成一个数据流：监控数据源--> 监控数据清洗 --> 监控数据计算 --> 推送报警

基于这个结论，正式我们会开始研究并探索Flink的契机。面对目前后端形形色色的监控方式及处理流程，每一个部分都是割裂的，无法统一思想的。面对不同的监控场景时，我们要考虑不同接处理手段，这变相的增加了开发人员的人力资源。

为此我们考虑怎样可以最大限度的统一各个报警的流程，并提供一个统一的处理平台。这个时候Flink来到了我们面前。

Flink是什么相比不用介绍了，官网已经介绍的相当详细了：
<https://flink.apache.org/>

上面抽象的监控流程：监控数据源--> 监控数据清洗 --> 监控数据计算 --> 推送报警

用Flink可以表示为：Source--> ETL--> Operator--> Sink

结合上面的分析，我打算先从推送报警功能入手。利用Flink的流式处理能力，开发一个统一的报警出口任务。
#### 数据源
数据源我们打算采用RocketMQ。创建一个统一的报警推送Topic，定义统一的报警消息格式。

|Topic|AlertWechatTopic|
|---|---|
|Body|{"alertMsg":"",<br>"alertTo":"",<br>"bizType":""<br> }

#### 算子
1. 应用RocketMQ-Extend提供的Flink支持，官方继承了RichParallelSourceFunction类。实现了注册Consumer，拉取Topic下的消息。

2. 官方提供的序列化方法仅有针对key和value的String序列化SimpleKeyValueDeserializationSchema。在运用上是满足的，但是后期编程上可能自定义会更加方便。后期我们会考虑扩展序列化方式或是官方提供的拉取消息源码，做到更灵活的消息格式传输。

3. 将从RocketMQ中拉取到的消息做Map，格式化成我们希望的结构。

```java
public static void main(String[] args) {
        ParameterTool parameterTool = ParameterTool.fromArgs(args);
        String nameserverAddress = parameterTool.get("nameserverAddress");
        String consumerGroup = parameterTool.get("consumerGroup");
        String consumerTopic = parameterTool.get("consumerTopic");

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        SendWechatService sendWechatService = new SendWechatService();
        // enable checkpoint
        env.enableCheckpointing(3000);

        Properties consumerProps = new Properties();
        consumerProps.setProperty(RocketMQConfig.NAME_SERVER_ADDR, nameserverAddress);
        consumerProps.setProperty(RocketMQConfig.CONSUMER_GROUP, consumerGroup);
        consumerProps.setProperty(RocketMQConfig.CONSUMER_TOPIC, consumerTopic);

        env.addSource(new RocketMQSource<Map>(new SimpleKeyValueDeserializationSchema(null,"body"),consumerProps))
                .name("rocketmq-source")
                .map(new MapFunction<Map, WechatAlertInfo>() {
                    @Override
                    public WechatAlertInfo map(Map value) throws Exception {
                        String body = (String)value.get("body");
                        WechatAlertInfo wechatAlertInfo = JSONObject.parseObject(body, WechatAlertInfo.class);
                        return wechatAlertInfo;
                    }
                }).addSink(new WechatSink())
                .name("wechat sink");
        try {
            env.execute("send to wechat");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

4. 将格式化完成的数据源输出到我们自定义的Sink中，该sink的功能为发送wechat信息。

```java
public class WechatSink extends RichSinkFunction<WechatAlertInfo> {
    private transient SendWechatService sendWechatService;

    @Override
    public void open(Configuration parameters) throws Exception {
        sendWechatService = new SendWechatService();
    }

    @Override
    public void invoke(WechatAlertInfo value, Context context) throws Exception {
        sendWechatService.sendMessage(value.getAlertMsg(),value.getAlertTo());
    }
}
```

以上我们就简单的实现了将RocketMQ中的报警消息推送至wechat的功能，下次我们把代码跑起来看看。
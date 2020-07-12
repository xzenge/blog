---
title: 基于Flink实现简单的日志异常推送
date: 2020-05-30 16:13:42
tags: 
    - Java
    - Flink
    - 监控
categories :
    - Technology
---

## 监控系统
业务系统经常会接入各种监控系统或APM系统，从而实现生产问题的快速报警和定位。比如CAT、PingPong、skywalking等，虽然实现原理和架构略有不同，但都为业务系统提供了调用链跟踪、异常日志、DB异常、连接超时、JVM信息等报警。甚至我们可以在监控平台上定制自己的报警规则，监控系统会根据规则向关系人提供报警信息。

以上都是基于监控系统提供的功能实现各个维度的报警。最近在捯饬Flink，所以想能否用Flink实现一个简单的日志异常报警。话不多说，开始撸代码。
<!-- more -->
## 引入日志服务依赖
由于系统已经全面接入阿里云日志平台，并且阿里云也非常有好的为我们提供了Flink阿里云Source的实现，所以我们只要引入阿里云提供的jar包即可，省去了自己实现的功夫。

```yml
    compile "com.aliyun.openservices:flink-log-connector:0.1.3"
    compile "com.google.protobuf:protobuf-java:2.5.0"
    compile "com.aliyun.openservices:aliyun-log:0.6.10"
    // 由于当前只是用Source，所以不引入Sink模块
    // compile "com.aliyun.openservices:log-loghub-producer:0.1.8" 
```
可以看到，阿里云日志通过protobuf协议进行序列化传输。

## 日志分析启动类
参考官网文档，我们配置了阿里云日志服务的链接属性，指定了日志文件。
此外我们还定义了interval参数为日志间隔；threshold为错误数；nameserverAddress为将错误信息发送至RockerMQ地址。结合上一篇文件，异常推送任务会将错误信息通过wechat推送给干系人。

```java
 ParameterTool parameterTool = ParameterTool.fromArgs(args);
        String interval = parameterTool.get("interval");
        String threshold = parameterTool.get("threshold");
        String nameserverAddress = parameterTool.get("nameserverAddress");

        Properties configProps = new Properties();
        // 设置访问日志服务的域名
        configProps.put(ConfigConstants.LOG_ENDPOINT, "xxxxxxxxxxxx");
        // 设置访问ak
        configProps.put(ConfigConstants.LOG_ACCESSSKEYID, "xxxxxxxx");
        configProps.put(ConfigConstants.LOG_ACCESSKEY, "xxxxxx");
        // 设置日志服务的project
        configProps.put(ConfigConstants.LOG_PROJECT, "xxxxxxx");
        // 设置日志服务的LogStore
        configProps.put(ConfigConstants.LOG_LOGSTORE, "xxxxxxxx");
        // 设置消费日志服务起始位置
        configProps.put(ConfigConstants.LOG_CONSUMER_BEGIN_POSITION, Consts.LOG_END_CURSOR);
        // 设置日志服务的消息反序列化方法
        RawLogGroupListDeserializer deserializer = new RawLogGroupListDeserializer();
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        Properties producerProps = new Properties();
        producerProps.setProperty(RocketMQConfig.NAME_SERVER_ADDR, nameserverAddress);

        DataStream<RawLogGroupList> logTestStream = env.addSource(
                new FlinkLogConsumer<RawLogGroupList>(deserializer, configProps));
```

### 解析阿里云日志并转换为日志流
```java
SingleOutputStreamOperator<LogInfo> process = logTestStream
        .flatMap(new TransLogInfo())
        .keyBy("podName")
        .keyBy("level")
        .process(new SplitLeveProcess());
```

#### 我们首先实现了FlatMapFunction接口，为的是将阿里云的日志流结构解析成我们自己想要的日志结构
```java
public class TransLogInfo implements FlatMapFunction<RawLogGroupList, LogInfo> {


    @Override
    public void flatMap(RawLogGroupList value, Collector<LogInfo> out) throws Exception {
        List<RawLogGroup> rawLogGroups = value.getRawLogGroups();
        if(rawLogGroups != null){
            for(RawLogGroup rawLogGroup : rawLogGroups){
                String nodeIp = "";
                Map<String, String> tags = rawLogGroup.getTags();
                if(tags != null){
                    nodeIp = tags.get("_node_ip_");
                }
                if(rawLogGroup != null){
                    List<RawLog> logs = rawLogGroup.getLogs();
                    if(!CollectionUtils.isEmpty(logs)){
                        for(RawLog log : logs){
                            if(log != null){
                                LogInfo logInfo = new LogInfo();
                                Map<String, String> contents = log.getContents();
                                if(contents != null){
                                    logInfo.setNodeIp(nodeIp);
                                    logInfo.setReceiveTime(contents.get("@timestamp"));
                                    logInfo.setLevel(contents.get("_level"));
                                    logInfo.setAccountId(contents.get("accountId"));
                                    logInfo.setPodName(contents.get("_pod_name_"));
                                    logInfo.setTraceId(contents.get("traceId"));
                                    logInfo.setMessage(contents.get("nte_message"));
                                    logInfo.setAppName(contents.get("appName"));
                                }
                                out.collect(logInfo);
                            }
                        }
                    }
                }
            }
        }
    }
```
#### 将解析后的日志流按照容器名和日志等级分组(如果只是想处理ERROR级别的错误，也可以用Filter对level进行过滤)

#### 我们实现了KeyedProcessFunction接口，将ERROR日志分流进行处理
```java
public class SplitLeveProcess extends KeyedProcessFunction<Tuple, LogInfo, LogInfo> {
    private static final String LEVEL_INOF = "INFO";
    private static final String LEVEL_ERROR = "ERROR";

    final OutputTag<LogInfo> outputTag = new OutputTag<LogInfo>("error-log"){};

    @Override
    public void processElement(LogInfo value, Context ctx, Collector<LogInfo> out) throws Exception {
        if(LEVEL_INOF.equals(value.getLevel())){
            out.collect(value);
        }else if(LEVEL_ERROR.equals(value.getLevel())){
            ctx.output(outputTag,value);
        }
    }
}
```

#### 得到ERROR日志侧输出流，处理错误日志流
```java
        DataStream<LogInfo> errorStream = process.getSideOutput(new OutputTag<LogInfo>("error-log"){});
        errorStream.keyBy("podName").process(new ErrorProcess(Long.valueOf(interval) * 1000,Integer.valueOf(threshold)))
//                .setParallelism(4)
                .map(new ToMessage())
                .addSink(new RocketMQSink(new SimpleKeyValueSerializationSchema(null, "body"),
                        new DefaultTopicSelector(topic), producerProps).withBatchFlushOnCheckpoint(true));
```

#### 针对错误日志处理，同样实现了KeyedProcessFunction接口，通过注册定时器判断interval周期内，错误数量是否超过threshold。如果超过则输出错误信息流，交由后面的算子处理

```java
public class ErrorProcess extends KeyedProcessFunction<Tuple, LogInfo, AlertInfo> {
    private long interval;
    private int threshold;
//    private transient SendWechatService sendWechatService;
    ValueState<Integer> cnt ;
    ValueState<Long> timer ;
    ValueState<String> pod ;
    ValueState<List> errorMsg ;

    public ErrorProcess(long interval,int threshold){
        this.interval = interval;
        this.threshold = threshold;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        cnt = getRuntimeContext().getState(new ValueStateDescriptor<Integer>("cnt",Integer.class,0));
        timer = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer",Long.class));
        pod = getRuntimeContext().getState(new ValueStateDescriptor<String>("pod",String.class));
        errorMsg = getRuntimeContext().getState(new ValueStateDescriptor<>("errorMsg",List.class));
    }

    @Override
    public void processElement(LogInfo value, Context ctx, Collector<AlertInfo> out) throws Exception {
        pod.update(value.getPodName());
        if("ERROR".equals(value.getLevel())){
            if(timer.value() == null){
                ctx.timerService().registerProcessingTimeTimer(ctx.timerService().currentProcessingTime() + interval);
            }
            cnt.update(cnt.value() + 1);
            List em = errorMsg.value();
            if(em == null){
                em = new ArrayList();
            }
            em.add(value.getMessage());
            errorMsg.update(em);
        }else {
            if(timer.value() != null){
                ctx.timerService().deleteEventTimeTimer(timer.value());
                cnt.clear();
                timer.clear();
            }
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<AlertInfo> out) throws Exception {
        if(cnt.value() >=threshold){
            AlertInfo alertInfo = new AlertInfo();
            alertInfo.setErrorCnt(cnt.value());
            alertInfo.setLevel("ERROR");
            alertInfo.setPodName(pod.value());
            alertInfo.setMg(errorMsg.value());
            String content = "In " + interval/1000 + " seconds, pod[" + pod.value()  + "] over " + threshold + " error happened ,error count[" + cnt.value() + "]    detail:" + "\n"  + "\n";
            for(int i = 1 ;i<=errorMsg.value().size();i++){
                content = content + "[" + i + "]" + errorMsg.value().get(i-1) + "  " + "\n";
            }
            alertInfo.setContent(content);
            out.collect(alertInfo);

        }
        cnt.clear();
        timer.clear();
        errorMsg.clear();
    }
```

#### 后续算子将错误日志流转换成指定结构，并发给RocketMQ的Sink，交由错误推送任务进行wechat推送。
```java
    public static class ToMessage implements MapFunction<AlertInfo, String> {
        @Override
        public String map(AlertInfo value) throws Exception {
            JSONObject messageBody = new JSONObject();
            messageBody.put("alertMsg",value.getContent());
            messageBody.put("alertTo","xxxxx");
            messageBody.put("bizType","");
            return messageBody.toJSONString();
        }
    }
```

## 总结
以上就是一个简单的错误日志分析及报警的流程，下次我们针对这个流程再写一篇测试文。同时这个流程中，我们对错误日志流采用的是侧流输出。所以在我们的主流中，我们还有很多正常日志值得我们分析和挖掘，后面我们也会围绕这些日志来做些有意思的东西，那这一次我们就先到这了。
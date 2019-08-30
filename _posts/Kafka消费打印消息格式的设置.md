---
layout: post
title:  "Kafka消费打印消息格式的设置"
date:   2019-02-27 12:19:12 +0800
tags:
      - Kafka
---

通常在测试kafka时，会用kafka-console-consumer来查看消息是否能被消费。改名了为kafka官方提供的控制台消费消息工具，使用方法可通过直接抵用该命令（sh kafka-console-consumer.sh）仅用于测试场景。

默认情况下该命令在消费消息打印时，只会打印消息的值。

如果想要查看更多消息的信息，如createTime，key，或者想要更友好显示消息，则可以指定相关配置。

    使用方法 sh kafka-console-consumer.sh ... --property print.key=true来打印消息的key值
    
当然还有其他的设置可以使用。 如下：

    if (props.containsKey("print.timestamp"))
      printTimestamp = props.getProperty("print.timestamp").trim.equalsIgnoreCase("true")
    if (props.containsKey("print.key"))
      printKey = props.getProperty("print.key").trim.equalsIgnoreCase("true")
    if (props.containsKey("key.separator"))
      keySeparator = props.getProperty("key.separator").getBytes
    if (props.containsKey("line.separator"))
      lineSeparator = props.getProperty("line.separator").getBytes
    // Note that `toString` will be called on the instance returned by `Deserializer.deserialize`
    if (props.containsKey("key.deserializer"))
      keyDeserializer = Some(Class.forName(props.getProperty("key.deserializer")).newInstance().asInstanceOf[Deserializer[_]])
    // Note that `toString` will be called on the instance returned by `Deserializer.deserialize`
    if (props.containsKey("value.deserializer"))
      valueDeserializer = Some(Class.forName(props.getProperty("value.deserializer")).newInstance().asInstanceOf[Deserializer[_]])

当然如果以上还不能完全满足诉求，就可以开发一个自定义的消息展示格式了

    1.  实现继承MessageFormatter的类，并实现相关init,writeTo,close方法
    2.  使用--format指定自定义的消息展示类名
    
    
    

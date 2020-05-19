# rabbitmq

## 基本结构
rabbitmq分为exchange交换机，routing key路由键，binding绑定，queue队列基本模块；生产者将mq发送到exchange中，然后根据情况进行binding，由routing key路由到对应的一个或多个queue中，消费者从对应的queue中取消息；

## 工作模式

### fanout 广播模式 <span hidden>/fænaut/</span>
这个模式会把消息发送到该exchange绑定的所有queue中，不需要指定路由，指了也没用；

### direct直接模式 <span hidden>/dəˈrekt/</span>
需要routing key，该路由不支持通配符匹配，只会发送到完全匹配的queue中；

### topic主题模式
他需要binding key与路由key，但是他允许通配符进行匹配，routing、binding key均使用”.“来分隔，binding key使用*匹配一个单词 #用于匹配零或多个单词，binding是具体的匹配规则，routing则是需要匹配的字符串；
### headers首部交换
他不依赖r与b，而是根据消息内容中的headers属性来匹配，在绑定queue与exchange时需要指定一组k、v，发送消息时，会匹配消息中的headers，必须完全匹配；
### amqp消息保证之事务模式
通过将当前channel设为transaction模式，有start、commit、rollback三个方法；
### confirm消息保证之确认模式
确认模式在消息投递到对队列后，会回复一个消息id，如果开启了持久化，则落盘后才会返回；ack确认，nack拒绝，可以一次拒绝N条消息，客户端可以设置basicNack方法的multiple参数为true，服务器会拒绝指定了delivery_tag的所有未确认的消息；<br>
他拥有confirmSelect开启、waitForConfirm等待确认，有个超时时间、setConfirmCallback设置回调，他允许批量推送消息，一次confirm；<br>
对于消费者，recover路由不成功的消息可以使用recovery重新发送到队列中，reject接收端告诉服务器这个消息我拒绝接收,不处理,可以设置是否放回到队列中还是丢掉，而且只能一次拒绝一个消息,官网中有明确说明不能批量拒绝消息，为解决批量拒绝消息才有了basicNack
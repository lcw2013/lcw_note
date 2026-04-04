## 5.顺序消息机制
**应用场景：**
每一个订单有从下单、锁库存、支付、下物流等几个业务步骤。每个业务步骤都由一个消息生产者通知给下游服务。如何保证对每个订单的业务处理顺序不乱？
**示例代码：**
生产者核心代码：通过MessageSelector，将orderId相同的消息，都转发到同一个MessageQueue中。
```
for (int i = 0; i < 10; i++) {
	int orderId = i;
	for(int j = 0 ; j <= 5 ; j ++){
		Message msg =
				new Message("OrderTopicTest", "order_"+orderId, "KEY" + orderId,
						("order_"+orderId+" step " + j).getBytes(RemotingHelper.DEFAULT_CHARSET));
		SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
			@Override
			public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
				Integer id = (Integer) arg;
				int index = id % mqs.size();
				return mqs.get(index);
			}
		}, orderId);
		System.out.printf("%s%n", sendResult);
	}
}
```
消费者核心代码：注入一个MessageListenerOrderly实现。
```
consumer.registerMessageListener(new MessageListenerOrderly() {
	@Override
	public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
		context.setAutoCommit(true);
		for(MessageExt msg:msgs){
			System.out.println("收到消息内容 "+new String(msg.getBody()));
		}
		return ConsumeOrderlyStatus.SUCCESS;
	}
});
```
**实现思路：**
RocketMQ实现消息顺序消费，是需要生产者和消费者配合才能实现的。
![](assets/2、RocketMQ客户端编程模型/file-20260404204034396.png)

1、生产者只有将一批有顺序要求的消息，放到同一个MesasgeQueue上，通过MessageQueue的FIFO特性保证这一批消息的顺序。
​ 如果不指定MessageSelector对象，那么生产者会采用轮询的方式将多条消息依次发送到不同的MessageQueue上。
​ 2、消费者需要实现MessageListenerOrderly接口，实际上在服务端，处理MessageListenerOrderly时，会给一个MessageQueue加锁，拿到MessageQueue上所有的消息，然后再去读取下一个MessageQueue的消息。

**注意点：**
​ 1、理解局部有序与全局有序。大部分业务场景下，我们需要的其实是局部有序。如果要保持全局有序，那就只保留一个MessageQueue。性能显然非常低。
​ 2、生产者端尽可能将有序消息打散到不同的MessageQueue上，避免过于集中导致数据热点竞争。
​ 3、消费者端只进行有限次数的重试。如果一条消息处理失败，RocketMQ会将后续消息阻塞住，让消费者进行重试。但是，如果消费者一直处理失败，超过最大重试次数，那么RocketMQ就会跳过这一条消息，处理后面的消息，这会造成消息乱序。
​ 4、消费者端如果确实处理逻辑中出现问题，不建议抛出异常，可以返回ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT作为替代。
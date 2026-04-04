## 5.顺序消息机制
**应用场景：**
每一个订单有从下单、锁库存、支付、下物流等几个业务步骤。每个业务步骤都由一个消息生产者通知给下游服务。如何保证对每个订单的业务处理顺序不乱？
**示例代码：**
生产者核心代码：
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

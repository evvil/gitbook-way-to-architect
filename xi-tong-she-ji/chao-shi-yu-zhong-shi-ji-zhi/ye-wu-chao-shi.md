# 业务超时

业务超时可分为如下两类：

**任务型**：比如，订单超时未支付取消，超时活动自动关闭等，这属于任务型超时，可以通过Worker定期扫描数据库修改状态即可。还有如有时候需要调用的远程服务超时了（比如，用户注册成功后，需要给用户发放优惠券），可以考虑使用队列或者暂时记录到本地稍后重试。

**服务调用型**：比如，某个服务的全局超时时间为500ms，但我们有多处服务调用，每处的服务调用超时时间可能不一样，此时，可以简单地使用Future来解决问题，通过如Future.get\(3000,TimeUnit.MILLISECONDS\)来设置超时。

内容来源：《亿级流量网站架构核心技术》：超时与重试机制

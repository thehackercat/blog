---
title: 坦率地讲 服务熔断 & 服务降级
date: 2018-02-01 19:03:25
comment: true
weight: 5
draft: false
categories: ["Coding"]
tags: ["Service", "Architecture", "Linux"]
lightgallery: true
thumbnail: https://www.atulhost.com/wp-content/uploads/2016/12/material-design-wallpaper-1.jpg
---

## 坦率地讲 服务熔断 & 服务降级

### 背景

之前遇到个问题，发现一个系统如果拆分了太多业务类服务，或者依赖于大量的第三方服务，就很容易因为某个服务的故障导致整个系统不可用，比如

- 模块中使用了 Elastic Search 进行监控，但是 ES 突然挂了，相关的 api 的调用报错导致级联的服务全部阻塞，那么应该要有规避由 ES 调用 raise 出的异常或者调用超时而导致整个模块或整个系统崩溃的保护措施。
- 使用 AWS 或 阿里云 的 ECS 服务来作为 micro-service 的载体，但是 ECS 服务故障或者过载了导致整个业务链无法正常进行，那么应有对应的降级或者限制调用频度的方案来进行保护。
<!--more-->

### 服务熔断

服务熔断和电路熔断是一个道理，如果一条线路电压过高，保险丝会熔断，防止出现火灾，但是过后重启仍然是可用的。

而服务熔断则是对于目标服务的请求和调用大量超时或失败，这时应该熔断该服务的所有调用，并且对于后续调用应直接返回，从而快速释放资源，确保在目标服务不可用的这段时间内，所有对它的调用都是立即返回，不会阻塞的。再等到目标服务好转后进行接口恢复。

熔断的方式有很多，最出名的奶飞的 [hystrix](https://github.com/Netflix/Hystrix) 项目里有很全面的实践，这里便先列个比较偷懒的案例。

举个栗子，

```python
# Elastic search service decorator
def api_trend(func):
  def wrapper(*args, **kwargs):
    # Call elastic search service to get api trend
    elastic_search_api_call()
    # Custom function
    return func(*args, **kwargs)
  return wrapper

# Custom task to do stuff
@api_trend
def custom_func(foo):
  retrun foo()

```

假设代码中的 `@api_trend` 是个调用 Elastic Search 服务来监控 api 执行情况的装饰器，那么如果 Elastic Search 服务挂了，则后续的 `custom_func(foo)` 也不会成功执行或者被阻塞。所以我们需要做的就是阻止后续的程序继续调用 `@api_trend` 或者 `elastic_search_api_call()` 这两位老哥，把 `custom_func(foo)` 隔离开，这样虽然暂时失去了监控，但是仍能保证业务能正常执行。

所以基于这点，我们可以简单地加个熔断控制器开关来隔离故障接口。

```python
from threading import Timer

# Melt down flag
FUSE = True

# Melt down recover func
def recover():
  FUSE = True
  return

# Melt down decorator
def melt_down(threshold=5, inteval=60, timeout=300, recover_time=3600):
  def wrap_melt(func):
    def wrapper(*args, **kwargs):
      is_fuse = True
      while threshold > 0 and is_fuse:
        try:
          func(timeout, *args, **kwargs)
          is_fuse = False
        exception Exception, e:
          is_fuse = True
          threshold -= 1
          continue
        time.sleep(inteval)
      FUSE = is_fuse
      if not FUSE:
        tr = threading.Timer(recover_time, recover)
    	tr.start()
      return FUSE
    return wrapper
  return wrap_melt

# Elastic search service decorator
def api_trend(func):
  def wrapper(*args, **kwargs):
    # Call elastic search service to get api trend
    if FUSE:
    	elastic_search_api_call()
    # Custom function
    return func(*args, **kwargs)
  return wrapper

# Custom task to do stuff
@melt_down
@api_trend
def custom_func(foo):
  return foo()
```

通过在调用 `@api_trend` 之前加上熔断控制器，进行目标服务的接口调用，如果在规定的重试次数内均未成功，则认为该服务在这一段时间内不可用，对于该 api 的所有调用全都用一个 FUSE_FLAG 进行隔离，并且设置一个定时 Thread, 在一定时间后重新打开 FUSE_FLAG，恢复目标服务的调用。

### 服务降级

当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行。

对于复杂系统而言，会有很多的微服务通过 rpc 调用，从而产生一个业务需要一条很长的调用链，其中任何一环故障了都会导致整个调用链失败或超时而导致业务服务不可用或阻塞。

这种情况下，可以暂时去掉调用链中故障的服务来进行降级，其中降级策略又有很多种，比如限流，接口拒绝等，这里就挑个简单的来举栗。

比如一个电商系统，用户模块，商品模块，订单模块，支付模块，物流模块分别是5个存在相互依赖性的服务，但是如果用户要下单购买个商品则可能需要一条长调用链依次 Call 到这5个模块。

```python
# Call chain
user = UserModule.sender.get_user()
product = ProductModule.sender.get_product(user.selected)
order = OrderModule.sender.post_order(product)
payment = PaymentModule.sender.post_payment(order)
logistics = LogisticsModule.sender.post_logistics(payment)
```

这时候如果物流模块崩了，那么很可能在最终购买商品的流程会被回滚，导致用户购买商品不成功，然而实际上，物流模块即便失效，仍应允许进行商品查看，下单，购买等，所以，坦率地讲，我们应该对这5个模块进行一个上下游依赖的剥离，使之变为纯净的 rpc 调用。

简单地说，

```Python
from xmlrpclib import ServerProxy

MODULE_TO_ENABLE = [
  'UserAgent',
  'ProductAgent',
  'OrderAgent',
  'PaymentAgent',
  'LogisticsAgent'
]

def custom_call():
  return foo()

def call_nothing():
  return

class LogisticsAgent(object):
  self.sender = ServerProxy("http://{host}:{port}".format(host=host, port=port))
  if self.__class__.__name__ in MODULE_TO_ENABLE:
  	self.sender.call = custom_call
  else:
    self.sender.call = call_nothing
  pass

# Call chain
if self.current_agent not in MODULE_TO_ENABLE:
    pass
```

这样通过 diable Call chain 中不重要的一环来确保其他模块可以正常使用。

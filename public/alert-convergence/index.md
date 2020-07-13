# 告警收敛设计


## 背景

最近在豆瓣做面向开发者的 op 报警系统, 最头疼的是下列几种情况

- 脉冲型报警
- 单个原因引起的报警发散
- 多个报警为同个诱因

由于目前现有的一些告警系统都强依赖后面的监控 agent/cmdb , 比如 open-falcon/prometheus 虽然都支持了告警收敛这一套, 但要从豆瓣的 statsd+icinga 迁移到 alertManager 需要折腾一阵。而我实际只想要一个中间轻量的 alert exporter 去做告警聚合/收敛.

所以我期望实现下列 feature:

- 对于告警可配置梯度, 比如 apperr > 50, 则立马报警不做 retry, 并且 notify_interval 为 1 分钟; 对于 50 > apperr > 20 则有3次 retry, retry 周期为 1 分钟, 报警周期为2分钟. 对于 apperr < 1 则 retry 5 次, retry 周期 3 分钟, 报警周期 5 分钟.
- 告警配置中可用 `与|或|非` 的方式进行组合告警,告警收敛. 如

```
[rules]
apperr
- script: xxx
- label: http, service, statsd, codeerr
- warning_threshold: 0.5
- critical_threshold: 1.0

http500:
- script: xxx
- label: http, codeerr, prom

prom-server-down:
- script: xxx
- label: prom, data-plane
...

apperr | http500 表示两者其一报警了, 则 mute 另一个报警. apperr && http500 表示当两种同时发生时才告警. apperr && !http500 表示 apperr 发生但 http500 不发生时告警, 若 apperr, http500 同时发生, 则不触发该条报警规则收敛掉 apperr 报警, 只报出 http500

```

- 告警按标签聚合
- 对于历史数据的分析预测

### 数据源

*目前我们的告警数据源大致可分为 prom, statsd 以及其他(比如 mfs http api, dae-monitor-agent 抓的 cgroups 数据, shuai 数据...), 但大头基本是 prom.*

### 告警流程

目前应该告警的流程如下:

1. dae ossetup -> bridge api 获取告警 meta data
2. dae ossetup 将 meta data render icinga template -> http 请求发给 monitor app -> 刷 icinga-server 配置
3. icinga-server 根据配置中的 cmd 执行命令如 `/usr/bin/dae-app-nagios-monitor dae-apperr $args1 $args2 ...` -> dae-sa-tools [dae-apperr 告警](https://github.intra.douban.com/dae/dae-sa-tools/blob/master/nazg/app_nagios_monitor.py#L164) -> bridge 告警元数据 [api](https://github.intra.douban.com/dae/bridge/blob/master/view/api/app.py#L170) -> prometheus/statsd 的 data-storage 层
4. bridge api 返回对应的指标, 由 dae-sa-tools 判断阈值后返回 icinga 识别的 warning/critical/unknown 等报警状态码
5. icinga-server 根据该状态码进行报警

而这中间是 icinaga 直接拿着 return code 进行告警控制,

故打算实现一个 alert-exporter 来做告警收敛, 预期的第三步骤流程变为

- alert-exporter 按照规则定期抓取数据源指标(如 bridge api, prometheus, statsd 等)放 redis
- icinga 执行 cmd -> `alert-exporter` -> 按配置的收敛规则从 redis 数据进行分析是否告警

### 思路

基于这篇论文[《运维监控系统告警收敛的算法研究与应用》](http://gb.oversea.cnki.net/KCMS/detail/detail.aspx?filename=1018803170.nh&dbcode=CMFD&dbname=CMFDREF)的思路, 有了以下想法

alert-export 需要记什么：

- 告警 name
- 告警 meta data(配置项)
- 告警 value
- 告警 timestamp
- 告警 recover time
1. ~告警的检查周期和提醒周期应该是指数周期降频收敛的算法, 即以 2^n 递增报警周期~
2. 告警分组, 做相同标签收敛
3. 对于频繁自愈的报警打标签降频
4. 对于历史告警数据可根据组合报警预测更大范围的告警, 和预测误报


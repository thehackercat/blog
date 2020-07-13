# 记一次升级 kube-proxy ipvs 引发的线上故障


## 背景
最近在升级集群的 kube-prxoy 并开启 ipvs mode, 引发了一些线上故障

## 替换原因
由于豆瓣的集群使用 calico + kube-proxy iptables mode + puppet iptable 脚本管理

三个组件共同操作同一份 iptables, 容易出现 race condition 问题, 并且还会互相抢占 iptables 锁, 是个 Mutex unsafe 的操作, 不易于维护.

故打算尽量减少操作 iptables 的部分, 替换成 ipvs

## 事故回溯

```
@400000005cbea9a81eaf9164 W0423 13:58:54.514773   14016 server.go:195] WARNING: all flags other than --config, --write-config-to, and --cleanup are deprecated. Please begin using a config file ASAP.
@400000005cbea9a81fe30854 W0423 13:58:54.534952   14016 server_others.go:287] Flag proxy-mode="" unknown, assuming iptables proxy
@400000005cbea9a81ff0bc24 I0423 13:58:54.535856   14016 server_others.go:140] Using iptables Proxier.
@400000005cbea9a820d714e4 I0423 13:58:54.550947   14016 server_others.go:174] Tearing down inactive rules.
@400000005cbea9a828a386bc I0423 13:58:54.681728   14016 server.go:448] Version: v1.11.1
@400000005cbea9a82be9b10c I0423 13:58:54.736441   14016 conntrack.go:98] Set sysctl 'net/netfilter/nf_conntrack_max' to 786432
@400000005cbea9a82be9b8dc I0423 13:58:54.736512   14016 conntrack.go:52] Setting nf_conntrack_max to 786432
@400000005cbea9a82be9b8dc I0423 13:58:54.736566   14016 conntrack.go:98] Set sysctl 'net/netfilter/nf_conntrack_tcp_timeout_established' to 86400
@400000005cbea9a82be9bcc4 I0423 13:58:54.736625   14016 conntrack.go:98] Set sysctl 'net/netfilter/nf_conntrack_tcp_timeout_close_wait' to 3600
@400000005cbea9a82bea457c I0423 13:58:54.736765   14016 config.go:202] Starting service config controller
@400000005cbea9a82bea7074 I0423 13:58:54.736778   14016 controller_utils.go:1025] Waiting for caches to sync for service config controller
@400000005cbea9a82bec31ac I0423 13:58:54.736850   14016 config.go:102] Starting endpoints config controller
@400000005cbea9a82bec3594 I0423 13:58:54.736867   14016 controller_utils.go:1025] Waiting for caches to sync for endpoints config controller
@400000005cbea9a831e37a0c I0423 13:58:54.836917   14016 controller_utils.go:1032] Caches are synced for service config controller
@400000005cbea9a831e3dbb4 I0423 13:58:54.837000   14016 controller_utils.go:1032] Caches are synced for endpoints config controller
@400000005cbea9aa0f11fe7c E0423 13:58:56.252807   14016 proxier.go:1340] Failed to delete stale service IP 172.19.119.127 connections, error: error deleting connection tracking state for UDP service IP: 172.19.119.127, error: error looking for path of conntrack: exec: "conntrack": executable file not found in $PATH
@400000005cbea9aa0f12f494 E0423 13:58:56.252887   14016 proxier.go:1340] Failed to delete stale service IP 172.19.0.53 connections, error: error deleting connection tracking state for UDP service IP: 172.19.0.53, error: error looking for path of conntrack: exec: "conntrack": executable file not found in $PATH
@400000005cbea9aa0f13b014 E0423 13:58:56.252939   14016 proxier.go:1340] Failed to delete stale service IP 172.19.68.99 connections, error: error deleting connection tracking state for UDP service IP: 172.19.68.99, error: error looking for path of conntrack: exec: "conntrack": executable file not found in $PATH

Failed to delete stale service IP 172.19.68.99 connections, error: error deleting connection tracking state for UDP service IP: 172.19.68.99, error: error looking for path of conntrack: exec: "conntrack": executable file not found in $PATH

= = 物理机上没装 conntrack 的包 = =

以及 WARNING: all flags other than --config, --write-config-to, and --cleanup are deprecated. Please begin using a config file ASAP.
```

替换时首先遇到了物理机未安装 conntrack 导致 kube-proxy 拉不起来的问题

接着在升级 kube-proxy 期间, 我通过  另起了 kube-proxy daemonset 执行了 `docker exec -it $(docker ps | grep hyperkube | awk '{print  $1 "  ./hyperkube kube-proxy --cleanup"}' )` 逐步清理了集群节点上的 iptables 规则.

此时便发生了线上故障. 归因为网络不通. podip 互 ping 丢包.

后续排查发现, 发现故障节点因为清理缘故 iptables 中少了这一条规则, docker bridge network 下的 container 向外发包时，没有做 SNAT, 对端收到包但无法正确回包，导致通信失败

```
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

故所有依赖 docker0 访问的非 k8s container 均出现了网络异常.

## 解决办法
临时在节点上批量执行以下脚本刷上 iptables 规则

```
sudo iptables -t nat -I  POSTROUTING 3 -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

## 额外引发的另一个 dns 查询黑洞事故
在升级 kube-proxy ipvs mode 后, 意外发现每当 coredns down 重启后, 必定出现短暂的 dns query 异常并伴随着一波 502.

总结原因如下:

- 当执行 kube-proxy 升级时会 drain k8s 节点
- drain node --> evict 掉节点 coredns pod 时 kubeproxy 来不及删掉旧的规则导致 ipvs rules 里仍有一条不存在后端的 realserver , 导致请求走到这条规则后 dns 解析进黑洞直到 5s timeout
- kube-proxy 会先把要删除的 rs [权重置为0](https://github.com/kubernetes/kubernetes/blob/155688b2f3521ffa913766086ab436eaae81639b/pkg/proxy/ipvs/graceful_termination.go#L149-L150)，然后加入到待删除 rs list 里由另一个 [1mins](https://github.com/kubernetes/kubernetes/blob/155688b2f3521ffa913766086ab436eaae81639b/pkg/proxy/ipvs/graceful_termination.go#L31) 周期的循环来删，但在删除逻辑中如果仍有连接保持着则会出现[删除失败但权重为0](https://github.com/kubernetes/kubernetes/blob/155688b2f3521ffa913766086ab436eaae81639b/pkg/proxy/ipvs/graceful_termination.go#L170)的过渡期，这期间便是有问题的, 会带来短暂的不可用

```
I0514 16:23:54.047540       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:53/UDP/172.18.81.97:53
I0514 16:23:54.047603       1 graceful_termination.go:171] Not deleting, RS 172.19.0.53:53/UDP/172.18.81.97:53: 0 ActiveConn, 3 InactiveConn
I0514 16:23:54.260987       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:9153/TCP/172.18.81.97:9153
I0514 16:23:54.261050       1 graceful_termination.go:174] Deleting rs: 172.19.0.53:9153/TCP/172.18.81.97:9153
I0514 16:23:54.437893       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:53/TCP/172.18.81.97:53
I0514 16:23:54.437952       1 graceful_termination.go:174] Deleting rs: 172.19.0.53:53/TCP/172.18.81.97:53
I0514 16:24:09.602189       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:53/UDP/172.18.81.97:53
I0514 16:24:09.602330       1 graceful_termination.go:171] Not deleting, RS 172.19.0.53:53/UDP/172.18.81.97:53: 0 ActiveConn, 2 InactiveConn
I0514 16:25:09.602466       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:53/UDP/172.18.81.97:53
I0514 16:25:09.602601       1 graceful_termination.go:171] Not deleting, RS 172.19.0.53:53/UDP/172.18.81.97:53: 0 ActiveConn, 2 InactiveConn
I0514 16:26:09.602770       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:53/UDP/172.18.81.97:53
I0514 16:26:09.602917       1 graceful_termination.go:171] Not deleting, RS 172.19.0.53:53/UDP/172.18.81.97:53: 0 ActiveConn, 2 InactiveConn
I0514 16:27:09.603060       1 graceful_termination.go:160] Trying to delete rs: 172.19.0.53:53/UDP/172.18.81.97:53
I0514 16:27:09.603182       1 graceful_termination.go:171] Not deleting, RS 172.19.0.53:53/UDP/172.18.81.97:53: 0 ActiveConn, 1 InactiveConn

直到所有连接都释放后才能正常删除
I0514 16:37:09.605442       1 graceful_termination.go:174] Deleting rs: 172.19.0.53:53/UDP/172.18.79.142:53
I0514 16:37:09.605468       1 graceful_termination.go:93] lw: remote out of the list: 172.19.0.53:53/UDP/172.18.79.142:53
```

## 解决办法
有 coredns 加上 fallback dns 服务以及 lvs 探活

确保 health check 过了才会正常打入后端


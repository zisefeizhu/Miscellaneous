> 最近在开发容器平台项目时需要调tke对集群进行CRUD操作，总是偶现调接口删除集群后，出现在tke界面显示集群处于Terminating。基于需求，写下本篇文章。
## 获取集群中的etcd pod 列表
```markdown
# ctl get po -n kube-system | grep etcd
etcd-192.168.xxx.45                      1/1     Running   45         25d
etcd-192.168.xxx.46                      1/1     Running   2          23d
etcd-192.168.xxx.47                      1/1     Running   2          397d
```
## 进入etcd 容器并配置环境
```markdown
# ctl exec -it etcd-192.168.xxx.46 -n kube-system -- /bin/sh
# etcd --version
etcd Version: 3.4.7
Git SHA: e694b7bb0
Go Version: go1.12.17
Go OS/Arch: linux/amd64
# alias etcdctl='etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key'
```
## 检索etcd中的所有目录和子目录
```markdown
# etcdctl get /  --prefix --keys-only
```
## 检索单个资源的详细信息
> 输出格式有四种,但是protobuf和simple会乱码,这里使用json格式。
```markdown
# etcdctl get /tke/platform/clusters/cls-g5xxxtql -w json
{"header":{"cluster_id":16977795579664915518,"member_id":5537999761451951234,"revision":201808958,"raft_term":3778}}

```
## etcdctl 删除数据
> 以特殊数据为例：tke的集群删除处于Terminating ， 无法通过常规命令等删除
```markdown
# ctl get cls | grep Terminating
cls-vwxxx2cz   Baremetal   1.18.3    Terminating   22d
cls-wc8xxxms   Baremetal   1.18.3    Terminating   5d20h
cls-wflnxxx8   Baremetal   1.18.3    Terminating   5d11h

# etcdctl get /  --prefix --keys-only | grep clusters
/tke/platform/clusters/cls-g5xxxtql
/tke/platform/clusters/cls-vwxxx2cz
/tke/platform/clusters/cls-wc8xxxms
/tke/platform/clusters/cls-wflnxxx8
/tke/platform/clusters/zisefeizhu

# etcdctl del /tke/platform/clusters/cls-vwxxx2cz
1

# ctl get cls | grep Terminating
cls-wc8xxxms   Baremetal   1.18.3    Terminating   5d20h
cls-wflnxxx8   Baremetal   1.18.3    Terminating   5d11h
```
> 更多的etcd操作请查看：https://etcd.io/docs/v3.5/
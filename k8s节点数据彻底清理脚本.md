```markdown
#!/bin/bash
rm -rf /etc/kubernetes
systemctl stop kubelet 2>/dev/null
systemctl stop docker 2>/dev/null
ip link del cni0 2>/etc/null
yum install -y psmisc
for port in 80 2379 6443 8086 {10249..10259} ; do
    fuser -k -9 ${port}/tcp
done
rm -fv /root/.kube/config
rm -rfv /var/lib/kubelet
rm -rfv /var/lib/cni
rm -rfv /etc/cni
rm -rfv /var/lib/etcd
rm -rfv /opt/cni /opt/containerd
docker rm -f $(docker ps -aq) 2>/dev/null
systemctl start docker 2>/dev/null
```
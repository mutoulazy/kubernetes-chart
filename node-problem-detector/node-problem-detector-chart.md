# k8s node-problem-detector组件部署

## 1 部署步骤
### 1.1 获取chart
[chart地址](https://github.com/helm/charts/tree/master/stable/node-problem-detector)
https://github.com/helm/charts/tree/master/stable/node-problem-detector

### 1.2 chart参数说明
<table>
    <tr>
        <td>名称</td>
        <td>默认值</td>
        <td>说明</td>
        <td>是否必填</td>
    </tr>
    <tr>
        <td>settings.log_monitors</td>
        <td>    - /config/kernel-monitor.json    - /config/docker-monitor.json</td>
        <td>日志处理收集模式</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>image.repository</td>
        <td>registry.cn-shenzhen.aliyuncs.com/acs-k8s/node-problem-detector</td>
        <td>镜像名称</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>image.tag</td>
        <td>v0.5.0</td>
        <td>镜像版本</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>resources</td>
        <td>{}</td>
        <td>资源限制</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>serviceAccount.name</td>
        <td>npdserviceaccount</td>
        <td>权限名称</td>
        <td>必填</td>
    </tr>
</table>

**注意** 日志处理收集模式一共有三种每种都对应node-problem-detector镜像文件的config文件夹内不同的配置文件，具体可以参阅https://github.com/kubernetes/node-problem-detector#problem-daemon了解

### 1.3 chart的部署
- 测试运行chart 检测yaml文件
helm install --dry-run --debug /node-problem-detector

- 检查无误后执行install完成搭建
helm install /node-problem-detector

### 1.4 问题
- 在chart的部署中可能出现由于node节点没有文件夹/var/log/journal导致Pod运行错误(观察Pod的日志可以得知)
- 解决方式：在k8s集群的各个node节点上创建/var/log/journal文件夹，并且重启Pod

## 2 部署情况
分别在node1 和了node2 上完成NPD(node-problem-detector)的DaemonSet搭建
```
[root@k8s-m1 mutoulazy]# kubectl get pod -o wide
NAME                                       READY     STATUS    RESTARTS   AGE       IP               NODE
datectortest-node-problem-detector-djkgk   1/1       Running   0          6m        10.244.215.106   k8s-n1
datectortest-node-problem-detector-zk7km   1/1       Running   0          6m        10.244.111.237   k8s-n2
hello-world-646654d956-dt6dv               1/1       Running   10         28d       10.244.215.102   k8s-n1
[root@k8s-m1 mutoulazy]# helm list
NAME        	REVISION	UPDATED                 	STATUS  	CHART                    	APP VERSION	NAMESPACE
datectortest	1       	Thu Nov 15 15:07:07 2018	DEPLOYED	node-problem-detector-1.0	v0.5.0     	default  
```

## 3 测试
- 在主节点启用事件监听
kubectl get events -w

- 模拟node节点发送告警 
sudo sh -c "echo 'kernel: BUG: unable to handle kernel NULL pointer dereference at TESTING' >> /dev/kmsg"

```
# 在node1 节点发送
[root@k8s-n1 f5c779647ffc49cd9fd021f9843d0ad6]# sudo sh -c "echo 'kernel: BUG: unable to handle kernel NULL pointer dereference at TESTING' >> /dev/kmsg"

# 在node2 节点发送
[root@k8s-n2 log]# sudo sh -c "echo 'kernel: BUG: unable to handle kernel NULL pointer dereference at TESTING' >> /dev/kmsg"
```


- 在主节点启用事件监听到node1 和 node2上自定义发送的事件告警（KernelOops）
```
2018-11-15 14:34:41 +0800 CST   2018-11-15 14:34:39 +0800 CST   5         k8s-n2.156738a51b504a43                         Node                                         Normal    NodeHasSufficientPID      kubelet, k8s-n2      Node k8s-n2 status is now: NodeHasSufficientPID
2018-11-15 14:34:41 +0800 CST   2018-11-15 14:34:41 +0800 CST   1         k8s-n2.156738a5a8d0ab81                         Node                                         Normal    NodeAllocatableEnforced   kubelet, k8s-n2      Updated Node Allocatable limit across pods
2018-11-15 14:35:46 +0800 CST   2018-11-15 14:35:46 +0800 CST   1         k8s-n2.156738b4acf314a0                         Node                                         Normal    Starting                  kube-proxy, k8s-n2   Starting kube-proxy.
2018-11-15 15:12:32 +0800 CST   2018-11-15 15:12:32 +0800 CST   1         k8s-n1.15673ab66cbf0d4c   Node                Warning   KernelOops   kernel-monitor, k8s-n1   BUG: unable to handle kernel NULL pointer dereference at TESTING
2018-11-15 15:12:45 +0800 CST   2018-11-15 15:12:45 +0800 CST   1         k8s-n2.15673ab984edc62e   Node                Warning   KernelOops   kernel-monitor, k8s-n2   BUG: unable to handle kernel NULL pointer dereference at TESTING
2018-11-15 15:16:39 +0800 CST   2018-11-15 14:36:39 +0800 CST   161       data-telling-goat-mysqlha-0.156738c106dc0220   PersistentVolumeClaim             Normal    FailedBinding   persistentvolume-controller   no persistent volumes available for this claim and no storage class is set
```

## 4 参考
https://github.com/kubernetes/node-problem-detector
# mysqlHA集群部署说明

## 1 集群环境搭建
此集群搭建由nfs-client-provisioner和mysqlha两部分组成

### 1.1 nfs-client-provisioner服务搭建
- value文件参数说明
<table>
    <tr>
        <td>名称</td>
        <td>默认值</td>
        <td>说明</td>
        <td>是否必填</td>
    </tr>
    <tr>
        <td>global.namespace</td>
        <td>default</td>
        <td>租户</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>nfs.provisionerimage</td>
        <td>registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner:latest</td>
        <td>nfs provisioner镜像地址</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>nfs.ip</td>
        <td>172.17.81.66</td>
        <td>nfs 服务IP地址</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>nfs.backuppath</td>
        <td>/data/provisioner</td>
        <td>nfs 备份文件地址</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>nfs.storage</td>
        <td>1Gi</td>
        <td>nfs 备份文件分配空间大小</td>
        <td>必填</td>
    </tr>
</table>

- 使用chart部署服务

```
在mysqlHAchart文件夹内执行查看yaml脚本是否正常
helm install --dry-run --debug nfs-client-provisioner/
检查yaml文件无误后直接执行完成服务搭建
helm install nfs-client-provisioner/
```

### 1.2 mysqlHA集群搭建
- value文件参数说明
<table>
    <tr>
        <td>名称</td>
        <td>默认值</td>
        <td>说明</td>
        <td>是否必填</td>
    </tr>
    <tr>
        <td>mysqlImage</td>
        <td>mysql:5.7.13</td>
        <td>mysql镜像</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>xtraBackupImage</td>
        <td>registry.cn-hangzhou.aliyuncs.com/test_k8s/xtrabackup:1.0</td>
        <td>xtrabackup镜像</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>mysqlha.replicaCount</td>
        <td>3</td>
        <td>集群副本数</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlRootPassword</td>
        <td>admin</td>
        <td>mysql root用户密码</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlReplicationUser</td>
        <td>repl</td>
        <td>nfs 副本用户名称</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlReplicationPassword</td>
        <td>"123456"</td>
        <td>nfs 副本用户密码</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlUser</td>
        <td>user</td>
        <td>数据库普通用户</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlPassword</td>
        <td>"123456"</td>
        <td>数据库普通用户密码</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlAllowEmptyPassword</td>
        <td>false</td>
        <td>是否允许远程匿名连接</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>mysqlha.mysqlDatabase</td>
        <td>test</td>
        <td>新建数据库</td>
        <td>选填</td>
    </tr>
    <tr>
        <td>persistence.enabled</td>
        <td>true</td>
        <td>是否备份数据持久化</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>persistence.size</td>
        <td>1Gi</td>
        <td>备份空间大小</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>resources.requests.cpu</td>
        <td>100m</td>
        <td>pod需要cpu资源大小</td>
        <td>必填</td>
    </tr>
    <tr>
        <td>resources.requests.memory</td>
        <td>128Mi</td>
        <td>pod需要内存资源大小</td>
        <td>必填</td>
    </tr>
</table>

- 使用chart部署服务

```
在mysqlHAchart文件夹内执行查看yaml脚本是否正常
helm install --dry-run --debug --name mysqlhatest --namespace default mysqlha/
检查yaml文件无误后直接执行完成服务搭建
helm install --name mysqlhatest --namespace default mysqlha/
```

## 2 集群连接
### 2.1 采用新建一个临时mysqlClient Pod 进行连接

```
1 获取连接数据库密码
    kubectl get secret --namespace default mysqlhatest-mysqlha -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo
2 创建一个临时Pod
    kubectl run mysql-client --namespace default --image=mysql:5.7.13 -it --rm --restart=Never -- /bin/sh
3 连接对应的mysql数据库Pod
    mysql -h mysqlhatest-mysqlha-0.mysqlhatest-mysqlha -p
    （注意 0号库为主库可读可写；1、2号库为备库只可进行读取）
```

## 3 集群备份
### 3.1 mysqlHA集群的组成

```
[root@172-17-81-37 mysqlHAchart]# kubectl get pod -n default
NAME                               READY     STATUS    RESTARTS   AGE
mysqlhatest-mysqlha-0              2/2       Running   0          2h
mysqlhatest-mysqlha-1              2/2       Running   0          2h
mysqlhatest-mysqlha-2              2/2       Running   0          2h
nfs-provisioner-65987f5487-nvnbq   1/1       Running   0          1d
```

该集群由1主2副 三个mysql数据库组成，0号库为主库可读可写；1、2号库为备库只可进行读取。从而实现主备读写分离

### 3.2 mysqlHA集群数据备份文件位置

由于使用了nfs做动态PV的原因，所以该集群的备份文件都存在于nfs服务器对应的目录下

```
[root@172-17-81-66 provisioner]# pwd
/data/provisioner
You have new mail in /var/spool/mail/root
[root@172-17-81-66 provisioner]# ll
total 0
drwxrwxrwx 2 root root  6 Nov 13 13:52 archived-default-test-claim1-pvc-a9f9037c-e704-11e8-bd73-fa163eba06fa
drwxrwxrwx 3 root root 19 Nov 13 14:10 default-data-mysqlhatest-mysqlha-0-pvc-99aba44f-e705-11e8-bd73-fa163eba06fa
drwxrwxrwx 3 root root 19 Nov 13 14:11 default-data-mysqlhatest-mysqlha-1-pvc-f22c62b8-e70a-11e8-bd73-fa163eba06fa
drwxrwxrwx 3 root root 19 Nov 13 14:12 default-data-mysqlhatest-mysqlha-2-pvc-1c3b4b5f-e70b-11e8-bd73-fa163eba06fa
drwxrwxrwx 2 root root  6 Nov 13 14:01 default-test-claim1-pvc-88ac7263-e709-11e8-bd73-fa163eba06fa
[root@172-17-81-66 provisioner]# 
```

### 3.3 备份与还原
- 备份 
1. 由于集群设置了主备模式，该集群会自动把主库的数据同步到2个副库内
2. 由集群内部xtrabackup组件自动进行全量备份保存在nfs服务对应目录内

- 还原
1. 只需要保持mysqlHA chart脚本部署的名称相同且nfs对应服务下备份文件目录没有被删除的情况下，新建集群的过程会自动还原已备份保存的数据
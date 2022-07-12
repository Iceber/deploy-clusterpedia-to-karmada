# 将 [Clusterpedia](https://github.com/clusterpedia-io/clusterpedia) 部署在 Karmada APIServer 空间中
> 由于 karmada apiserver 的特殊性，我们需要使用 yaml 来部署。
>
> **未来如果有小伙伴感兴趣可以参与修改 [charts](https://github.com/clusterpedia-io/clusterpedia/tree/main/charts) 来支持设置外部 kubeconfig，**
> **这样 Clusterpedia 根据 kubeconfig 连接到其他 APIServer（包括 Karmada APIServer） 中**

主要分为三步：
* 在 karmada 空间中部署 CRDs， APIService 和 Service
* 在 karmada 空间中创建用于自动发现纳管集群的 ClusterImportPolicy
* 在 karmada host 空间中部署 clusterpedia 组件

部署完成后，将 kubectl 切换到 Karmada APIServer 空间中，后续的使用可以参考 https://clusterpedia.io/zh-cn/docs/usage/
> 使用 ClusterImportPolicy 后，不需要手动创建 PediaCluster, clusterpedia 会自动发现 karmada 纳管的集群

## 在 Karmada APIService 空间内的操作
karmada 空间中需要部署的 yaml 都在 *karmada* 目录下
```bash
$ cd karmada && ls -al
total 80
drwxr-xr-x  9 icebergu  staff   288  7 12 11:49 .
drwxr-xr-x  9 icebergu  staff   288  7 12 11:46 ..
-rw-r--r--  1 icebergu  staff  2091  7 12 11:36 cluster.clusterpedia.io_clustersyncresources.yaml
-rw-r--r--  1 icebergu  staff  9576  7 12 11:36 cluster.clusterpedia.io_pediaclusters.yaml
drwxr-xr-x  3 icebergu  staff    96  7 12 11:49 clusterimportpolicy
-rw-r--r--  1 icebergu  staff   303  7 12 11:36 clusterpedia_apiserver_apiservice.yaml
-rw-r--r--  1 icebergu  staff   254  7 12 11:36 clusterpedia_apiserver_service.yaml
-rw-r--r--  1 icebergu  staff  7157  7 12 11:36 policy.clusterpedia.io_clusterimportpolicies.yaml
-rw-r--r--  1 icebergu  staff  7741  7 12 11:36 policy.clusterpedia.io_pediaclusterlifecycles.yaml
```

**kubectl 切换到 karmada apiserver**
### 部署 CRDs，APIService 和 Service
```bash
kubectl apply -f .
```

### 自动发现和同步 karmada 纳管的集群
自动发现多云平台集群的功能详情可以查看 https://github.com/clusterpedia-io/clusterpedia/issues/185

@calvin0327 贡献了用于 karmada 的 ClusterImportPolicy https://github.com/clusterpedia-io/clusterpedia/pull/263

当前并不支持同步 Pull 模式的集群，未来 clusterpedia 会增加 agent 模式来应对这种场景

karmada clusterimportpolicy 放在 *./clusterimportpolicy* 下
```bash
kubectl apply ./clusterimportpolicy
```

clusterpedia 组件部署完成后，可以直接 `kubectl get pediacluster` 查看已经被 karmada 纳管的集群
```bash
$ kubectl get pediacluster
$ kubectl get pediaclusterlifecycle
```

如果发现未创建的 pediacluster，可以通过 `kubectl describe pediaclusterlifecycle <name>` 查看详情

## 在 karmada host 中部署组件
我们将 clusterpedia 的组件部署在 karmada host 中，为了复用 karmada apiserver 的 kubeconfig，我们将 clusterpedia 部署在 karmada-system 命名空间。

**将 kubectl 切换到 karmada host**

进入 host 目录下，可以看到组件的 yaml
```bash
$ cd host && ls -al
total 32
drwxr-xr-x  7 icebergu  staff   224  7 12 11:36 .
drwxr-xr-x  9 icebergu  staff   288  7 12 12:05 ..
-rw-r--r--  1 icebergu  staff  1393  7 12 11:36 clusterpedia_apiserver_deployment.yaml
-rw-r--r--  1 icebergu  staff   209  7 12 11:36 clusterpedia_apiserver_service.yaml
-rw-r--r--  1 icebergu  staff  1463  7 12 11:36 clusterpedia_clustersynchro_manager_deployment.yaml
-rw-r--r--  1 icebergu  staff   900  7 12 11:36 clusterpedia_controller_manager_deployment.yaml
drwxr-xr-x  4 icebergu  staff   128  7 12 11:36 internalstorage
```

### 部署存储组件
首先需要部署存储组件，默认存储层可以选择使用 MySQL 或者 PostgreSQL，可以选择想要使用的存储组件
**可以选择使用外部的存储组件**，详见 https://clusterpedia.io/zh-cn/docs/installation/configurate/configurate-internalstorage/
```bash
$ cd internalstorage && ls -al
total 0
drwxr-xr-x  4 icebergu  staff  128  7 12 11:36 .
drwxr-xr-x  7 icebergu  staff  224  7 12 11:36 ..
drwxr-xr-x  6 icebergu  staff  192  7 12 11:36 mysql
drwxr-xr-x  6 icebergu  staff  192  7 12 11:36 postgres
$ cd postgres
```

存储组件使用 Local PV 的方式存储数据，部署时需要指定 Local PV 所在节点
```bash
export STORAGE_NODE_NAME=<节点名称>
sed "s|__NODE_NAME__|$STORAGE_NODE_NAME|g" `grep __NODE_NAME__ -rl ./templates` > clusterpedia_internalstorage_pv.yaml
```

将存储组件部署到 karmada-system 命名空间
```bash
kubectl -n karmada-system apply -f .
```

### 部署 Clusterpedia
我们回到 host 目录中
```bash
$ cd host && ls -al
```
部署组件到 karmada-system 命名空间中
```bash
kubectl -n karmada-system apply -f .
```

# 使用
对于 clusterpedia 的使用我们需要在 karmada apiserver 中，**切换 kubectl 到 karmada apiserver 空间**

创建 kubectl 集群快捷方式
```bash
$ # 进入项目目录
$ ./gen-clusterconfigs.sh
```
查看接入的集群
```bash
$ kubectl get pediacluster
```

访问集群资源
```bash
$ kubectl --cluster clusterpedia api-resources
$ kubectl --cluster <clustername> api-resources
```
具体使用，可以参考 https://clusterpedia.io/zh-cn/docs/usage/



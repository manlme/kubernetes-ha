# 目标
* 在集群中安装helm

helm是k8s的包（charts）管理软件，功能超级强大也很方便。

下载helm放到master节点，将helm放到path里。

在master上运行
```
kubectl create serviceaccount tiller --namespace kube-system

# be careful to this role. you can create a smaller privillege role. and bind it with sa.

kubectl create clusterrolebinding test --serviceaccount=tiller --clusterrole=cluster-admin

helm init --tiller-image=<your docker image repos>/kubernetes-helm/tiller:v2.8.2 --service-account=tiller
```
如果无法访问外网，遇到加google的repo的错误，运行如下命令，删除所有tiller，再次安装
```
touch ~/.helm/repository/repositories.yaml
```
拥有了helm也就拥有了kubernetes非常多共享的应用。比如dashboard，nginx-ingress，elk， gitlab等等。手到拈来。

# install dashboard to kubernetes cluster

## certs
* dashboard.crt
* dashboard.key
stored under secret 

data:
  dashboard.crt: base64
  dashboard.key: base64


## 自签证书

* 产生私钥（key）和证书签发请求（csr）
```
$ openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
...
$ openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
# Writing RSA key
$ rm dashboard.pass.key
$ openssl req -new -key dashboard.key -out dashboard.csr
...
Country Name (2 letter code) [AU]: US
...
A challenge password []:
...
```
* 生成ssl证书

`openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt`



* 生成k8s的secret
```
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kube-system
```

# install dashboard to kubernetes cluster

## 需要的证书 certs
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
生成csr，产生服务器证书
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn
    
[ dn ]
C = <country>
ST = <state>
L = <city>
O = <organization>
OU = <organization unit>
CN = <MASTER_IP>
    
[ req_ext ]
subjectAltName = @alt_names
    
[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = <MASTER_IP>
IP.2 = <MASTER_CLUSTER_IP>
    
[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

```
openssl req -new -key server.key -out server.csr -config csr.conf
```
* 生成自签tls证书

`openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt`

* 生成k8s签发的证书
```
$ cat <<EOF | kubectl create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  groups:
  - system:authenticated
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

* 查看k8s的csr
```
kubectl describe csr my-svc.my-namespace
```
* approve it
```
kubectl certificate approve <csr>
```

* 下载签名证书
```
kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 -d > server.crt
```
* 生成dashboard k8s的secret
```
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kube-system
```

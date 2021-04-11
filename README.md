# k8s-cert-manager-aliyundns
自己操作了一遍，基于阿里云k8s的1.16版本，最终成功，网上的教程有很多，这个教程是最靠谱的。 https://github.com/PowerDos/k8s-cret-manager-aliyun-webhook-demo
我记录的内容是我操作的过程，大部分参考了上面那个连接，感谢原作者。

---
# 简介

> 这里介绍如何在K8s中通过cret-manager自动创建HTTPS证书，提供两种方式，一种是单域名证书，一种是通过阿里云DNS验证实现通配符域名证书申请

> 我通过官网给的配置文件yaml直接安装cret-manager,**请注意查看k8s版本正确安装对应版本的应用**

## 1.安装cert-manager

### 前期准备

#### 添加命名空间

> kubectl create namespace cert-manager

==注：这里如果没有添加，其实也不影响，安装cert-manager的时候，那个yaml里面有自动创建namespace，不过我是做了这一步。==

### 安装cert-manager

```
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.0/cert-manager.yaml

#为了下载yaml顺利，我用两步来操作
$ wget https://github.com/jetstack/cert-manager/releases/download/v1.3.0/cert-manager.yaml
$ kubectl apply -f cer-manager.yaml
```

官网地址：https://cert-manager.io/docs/installation/kubernetes/

==注：yaml里面有docker的image文件，我看网上有教程说国内服务器因为docker pull官网image的速度不稳定，可能需要把image上传到阿里云的容器空间，再进行部署，不过这个问题我没有遇到。==

#### 安装CRDs

> 注意安装对应的版本

```
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.crds.yaml

#为了下载yaml顺利，我用两步来操作，我用了最新版1.3.0版本
$ wget https://github.com/jetstack/cert-manager/releases/download/v1.3.0/cert-manager.crds.yaml
kubectl apply -f cer-manager.crds.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager-legacy.crds.yaml
```

==注：因为我的k8s版本高于1.15，所以，我用了第一个，这里我第一次部署的时候就遇到坑了，我用的是1.15+的部署，后面安装webhook的时候，网上很多教程放出的是低版本的那个链接，导致webhook一直不能正常运行，为此浪费了半天的时间，我一步一步查，才找到这个原因。==

==后来我又看了一下，这个CRDs的版本应该是客户自定义内容的版本，装这个和上面安装cert-manager好像是重复了，不过我当时也是一步步趟出来的，就两个都装了。==

### 验证是否安装成功

> 以下结果为成功，你也可以看看镜像日志，是否正常启动，是否正常

```
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6344597-zw8kh               1/1     Running   0          2m
cert-manager-cainjector-348f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-893u48fcdb-nlzsq      1/1     Running   0          2m
```

## 2.安装证书

> 官方介绍这中 Issuer 与 ClusterIssuer 的概念：

```
Issuers, and ClusterIssuers, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests. All cert-manager certificates require a referenced issuer that is in a ready condition to attempt to honor the request.
```

> Issuer 与 ClusterIssuer 的区别是 ClusterIssuer 可跨命名空间使用，而 Issuer 需在每个命名空间下配置后才可使用。这里我们使用 ClusterIssuer，其类型选择 Let‘s Encrypt

> 测试证书

```
# cluster-issuer-letsencrypt-staging.yaml

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # 务必将此处替换为你自己的邮箱, 否则会配置失败。当证书快过期时 Let's Encrypt 会与你联系
    email: gavin.tech@qq.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # 将用来存储 Private Key 的 Secret 资源
      name: letsencrypt-staging
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

> 正式证书

```
# cluster-issuer-letsencrypt-prod.yaml

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email:  gavin.tech@qq.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

### 这里安装两个环境的证书的作用

> 这里分别配置了测试环境与生产环境两个 ClusterIssuer， 原因是 Let’s Encrypt 的生产环境有着非常严格的接口调用限制，最好是在测试环境测试通过后，再切换为生产环境。生产环境和测试环境的区别：https://letsencrypt.org/zh-cn/docs/staging-environment/

### 在Ingress中使用证书

> 在ingress配置后，会自动生成证书

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    # 务必添加以下两个注解, 指定 ingress 类型及使用哪个 cluster-issuer
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer："letsencrypt-staging" # 这里先用测试环境的证书测通后，就可以替换成正式服证书

    # 如果你使用 issuer, 使用以下注解 
    # cert-manager.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - example.example.com                # TLS 域名 - 这里仅支持单域名，下面会讲通配符的域名配置
    secretName: quickstart-example-tls   # 用于存储证书的 Secret 对象名字，可以是任意名称，cert-manager会自动生成对应名称的证书名称
  rules:
  - host: example.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```

==注：整个第二步是http验证域名，因为我们的域名在阿里云，我就直接操作了第三步。==

## 3.通过DNS配置通配符域名域名

> 这里样式的是阿里云DNS操作的流程，如果需要其他平台的方法，可以自行开发，或者找已开源webhook，这是官方的例子：https://github.com/jetstack/cert-manager-webhook-example

> 这里用的是这个包：https://github.com/pragkent/alidns-webhook

### 安装WebHook

> 不同cret-manager的安装办法不同

#### cret-manager 版本大于等于v0.11

##### 安装

```
# Install alidns-webhook to cert-manager namespace. 
kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml
#这里也可以先wget下来yaml，再kubectl apply执行
```

##### 添加阿里云RAM账号

> 子账号需要开通HTTPS管理权限(AliyunDNSFullAccess,管理云解析（DNS）的权限)

```
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: cert-manager
data:
  access-key: YOUR_ACCESS_KEY # 需要先base64加密 echo -n "Mac" | base64
  secret-key: YOUR_SECRET_KEY # 需要先base64加密
```

==注：这里是一个坑，阿里云的key和secret都需要先base64加密之后，才能被使用，否则执行就报错。我一开始找的教程里都没有说明，这一步就一直报错。==

##### 创建ClusterIssuer

> 测试证书申请

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging-dns
spec:
  acme:
    email: gavin.tech@qq.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-dns
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com # 注意这里要改动，在https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml中也要改动对应的groupName
          solverName: alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```

> 正式证书申请

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-dns
spec:
  acme:
    email: gavin.tech@qq.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-dns
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com
          solverName: alidns
          config:
            region: "" # 这里可以不填 或者填对应的区域:cn-shenzhen
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```

==注：我没有用测试证书，直接选择了正式证书==

##### 创建Certificate

> 测试证书

```
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: ditigram-com-staging-tls
spec:
  secretName: ditigram-com-staging-tls
  commonName: ditigram.com
  dnsNames:
  - ditigram.com
  - "*.ditigram.com"
  issuerRef:
    name: letsencrypt-staging-dns
    kind: ClusterIssuer
```

> 正式证书

```
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: ditigram-com-prod-tls
spec:
  secretName: ditigram-com-prod-tls
  commonName: ditigram.com
  dnsNames:
  - ditigram.com
  - "*.ditigram.com"
  issuerRef:
    name: letsencrypt-prod-dns
    kind: ClusterIssuer
```

==注：上面两块我暂时没有使用，直接在ingress里面生成证书了==

##### 在Ingress中应用

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    # 务必添加以下两个注解, 指定 ingress 类型及使用哪个 cluster-issuer
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-staging-dns" # 这里先用测试环境的证书测通后，就可以替换成正式服证书
spec:
  tls:
  - hosts:
    - "*.ditigram.com"                     # 如果填写单域名就只会生产单域名的证书，如果是通配符请填写"*.example.com", 注意：如果填写example.com只会生成www.example.com一个域名。
    secretName: ditigram-com-staging-tls   # 测试的证书，填写刚刚创建Certificate的名称，注意更换环境时证书也要一起更换，这里并不会像单域名一样自动生成
  rules:
  - host: example.ditigram.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```

#### cret-manager 版本小于v0.11 ==这个版本我没有试，我就不删了，也许有朋友能用到==

##### 安装

```
# Install alidns-webhook to cert-manager namespace. 
kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/legacy.yaml
```

##### 添加阿里云RAM账号

> 子账号需要开通HTTPS管理权限(AliyunDNSFullAccess,管理云解析（DNS）的权限)

```
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: cert-manager
data:
  access-key: YOUR_ACCESS_KEY # 需要先base64加密
  secret-key: YOUR_SECRET_KEY # 需要先base64加密
```

##### 创建ClusterIssuer

> 测试证书申请

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging-dns
spec:
  acme:
    email: gavin.tech@qq.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-dns
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com # 注意这里要改动，在https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml中也要改动对应的groupName
          solverName: alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```

> 正式证书申请

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-dns
spec:
  acme:
    email: gavin.tech@qq.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-dns
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com
          solverName: alidns
          config:
            region: "" # 这里可以不填 或者填对应的区域:cn-shenzhen
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```

##### 创建Certificate

> 测试证书

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: ditigram-com-staging-tls
spec:
  secretName: ditigram-com-staging-tls
  commonName: ditigram.com
  dnsNames:
  - ditigram.com
  - "*.ditigram.com"
  issuerRef:
    name: letsencrypt-staging-dns
    kind: ClusterIssuer
```

> 正式证书

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: ditigram-com-prod-tls
spec:
  secretName: ditigram-com-prod-tls
  commonName: ditigram.com
  dnsNames:
  - ditigram.com
  - "*.ditigram.com"
  issuerRef:
    name: letsencrypt-prod-dns
    kind: ClusterIssuer
```

##### 在Ingress中应用

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    # 务必添加以下两个注解, 指定 ingress 类型及使用哪个 cluster-issuer
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer："letsencrypt-staging-dns" # 这里先用测试环境的证书测通后，就可以替换成正式服证书
spec:
  tls:
  - hosts:
    - "*.ditigram.com"                     # 如果填写单域名就只会生产单域名的证书，如果是通配符请填写"*.example.com", 注意：如果填写example.com只会生成www.example.com一个域名。
    secretName: ditigram-com-staging-tls   # 测试的证书，填写刚刚创建Certificate的名称，注意更换环境时证书也要一起更换，这里并不会像单域名一样自动生成
  rules:
  - host: example.ditigram.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```

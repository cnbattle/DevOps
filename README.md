# CloudNativeArchitect

> 致力于设计快速/灵活的云原生架构设计及搭建。

## 搭建

### Debian10 安装kubernetes

```
# On master 自动初始化
apt update -y && apt upgrade -y && apt install curl -y && apt autoremove -y
curl -fsSL https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/install-kubernetes-on-buster.sh | bash - 

# On node 需手动jion
apt update -y && apt upgrade -y && apt install curl -y && apt autoremove -y
curl -fsSL https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/install-kubeadm-on-buster.sh | bash - 
```

### 手动初始化

```
# initialize kubernetes with a Flannel compatible pod network CIDR
kubeadm init --apiserver-advertise-address=0.0.0.0 \
--apiserver-cert-extra-sans=127.0.0.1 \
--image-repository=registry.aliyuncs.com/google_containers \
--ignore-preflight-errors=all \
--service-cidr=10.18.0.0/16 \
--pod-network-cidr=10.244.0.0/16

# setup kubectl
mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# del master taint
kubectl taint nodes --all node-role.kubernetes.io/master-

# install Flannel
kubectl apply -f https://raw.fastgit.org/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

## 安装 traefik

https://github.com/traefik/traefik

```shell

kubectl create ns traefik
kubectl apply -n traefik -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/traefik/1.traefik-crd.yaml
kubectl apply -n traefik -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/traefik/2.traefik-rbac.yaml
kubectl apply -n traefik -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/traefik/3.0.traefik-ingress-controller.yml
sleep 1
kubectl delete -n traefik -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/traefik/3.0.traefik-ingress-controller.yml
sleep 1
kubectl apply -n traefik -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/traefik/3.1.traefik-ingress-controller.yml
```

## 安装 metallb

https://github.com/metallb/metallb

```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/metallb/namespace.yaml
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/metallb/metallb.yaml
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/metallb/config.yaml
```

## 安装 metrics server

https://github.com/kubernetes-sigs/metrics-server

```
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/kubernetes-dashboard/metrics-server.yaml
```

## 安装 kubernetes dashboard

https://github.com/kubernetes/dashboard

```
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/kubernetes-dashboard/kubernetes-dashboard.yaml
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/kubernetes/kubernetes-dashboard/dashboard-adminuser.yaml
kubectl describe secret admin-user --namespace=kube-system
```

## 安装 mysql

https://github.com/bitpoke/mysql-operator

```shell
kubectl create namespace mysql
helm repo add bitpoke https://helm-charts.bitpoke.io
helm install mysql bitpoke/mysql-operator -n=mysql
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/mysql/dev-secret.yaml
kubectl apply -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/mysql/dev-cluster.yaml
```

## 安装 redis

https://github.com/OT-CONTAINER-KIT/redis-operator

```shell
kubectl create namespace redis
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm upgrade redis-operator ot-helm/redis-operator --install --namespace redis
kubectl create secret generic redis-secret --from-literal=password=password -n redis
helm upgrade dev-redis ot-helm/redis --install --namespace redis
```

## 安装 argo-workflows

https://github.com/argoproj/argo-cd
https://github.com/argoproj/argo-workflows

```shell
kubectl create ns argo
kubectl apply -n argo -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/argo/cd.yaml
kubectl apply -n argo -f https://raw.fastgit.org/cnbattle/CloudNativeArchitect/main/argo/workflow.yaml
# Visit cd: https://<your server ip>:30810/
# Visit workflow: https://<your server ip>:30846/
```

## 安装 kube-prometheus

https://github.com/prometheus-operator/kube-prometheus

```
# 创建命名空间和CRD
kubectl create -f kube-prometheus/manifests/setup
# 等待命令空间和CRD资源可用后再执行
kubectl create -f kube-prometheus/manifests/
```

## 安装 jaeger-operator

https://github.com/jaegertracing/jaeger-operator

```
kubectl create namespace observability
kubectl create -n observability -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing.io_jaegers_crd.yaml
kubectl create -n observability -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
kubectl create -n observability -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/role.yaml
kubectl create -n observability -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
kubectl create -n observability -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/operator.yaml

kubectl create -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/cluster_role.yaml
kubectl create -f https://raw.fastgit.org/jaegertracing/jaeger-operator/master/deploy/cluster_role_binding.yaml

kubectl apply -n observability -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simplest
EOF
```
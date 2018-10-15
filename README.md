# kubernetes-for-china baremetal on ubuntu bionic

 With Kubernetes v1.12.1

## Kubernetes Install

1. Install Docker, turn off swipe and commment out swap entry from your fstab

- [Get Docker CE for Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

```bash
sudo su
swapoff -a 
vi /etc/fstab
#/swap 
apt install docker.io
```

2. Installing kubeadm, kubelet and kubectl since the bionic is not yet available we will be using xenial repos

- [install kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
- [Kubernetes mirror](https://opsx.alibaba.com/mirror)

```bash
apt-get update && apt-get install -y apt-transport-https
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

3. Preload Kubernetes images form Alibaba Cloud Registry Service

- [gcr.io images mirror on aliyun](https://dev.aliyun.com/list.html?namePrefix=google-containers)

```bash
# list images kubeadm will use
$ kubeadm config images list

# pre load image
$ ./load_images.sh
```

4. Create a Cluster
```bash
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
$ sudo KUBECONFIG=/etc/kubernetes/admin.conf kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml

# Master Isolation (if single-machine Kubernetes cluster )
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

## [Helm](https://github.com/kubernetes/helm)


```bash
# install
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz | tar xzv
$ sudo cp linux-amd64/helm /usr/local/bin
$ rm -rf linux-amd64

# initialize the local CLI and also install Tiller into your Kubernetes cluster
$ helm init

# fix https://github.com/kubernetes/helm/issues/3130
$ kubectl create serviceaccount --namespace kube-system tiller
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

# update charts repo
$ helm repo update

# install mysql chart
$ helm install --name my-mysql stable/mysql

# delete
$ helm delete my-mysql

# remove the release from the store and make its name free for later use
$ helm delete --purge my-mysql
```

## [Rook](https://github.com/rook/rook)

```bash
$ docker pull rook/ceph:master

# install Rook Operator: https://rook.io/docs/rook/master/helm-operator.html
$ kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

# create the Rook cluster
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml

# list pods in the rook-ceph namespace.
$ kubectl -n rook-ceph get pod

#  creating storage pools.
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/pool.yaml
# create block storage to be consumed by a pod
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/storageclass.yaml

# set rook-block as default storageclass 
$ kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```

## [OpenEBS](https://github.com/openebs/openebs)

```bash
# OpenEBS can be setup in few easy steps. 
# You can get going on your choice of Kubernetes cluster by having open-iscsi installed on the Kubernetes nodes and running the openebs-operator using kubectl.

# Start the OpenEBS Services using Operator:
$ kubectl apply -f https://raw.githubusercontent.com/openebs/openebs/master/k8s/openebs-operator.yaml

# As an alternative to the above comment you can also use Helm package manager to install OpenEBS. 
# More info on OpenEBS Chart: https://github.com/kubernetes/charts/tree/master/stable/openebs
$ helm install stable/openebs

# Customize or use the Default storageclasses
$ kubectl apply -f https://raw.githubusercontent.com/openebs/openebs/master/k8s/openebs-storageclasses.yaml

# List predefined OpenEBS Storage Classes
$ kubectl get sc

# set openEBS Standard Storage Class as default storageclass 
$ kubectl patch storageclass openebs-standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Optional: Enable monitoring using Prometheus and Grafana
$ kubectl apply -f https://raw.githubusercontent.com/openebs/openebs/master/k8s/openebs-monitoring-pg.yaml

```

# Prepare the environment
## Deploy a local k8s cluster with k3d

```bash

curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | TAG=v1.7.0 bash
k3d create --workers 2
```

Wait a couple of minutes and then you should see the deployment succeeding:

```bash
export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
kubectl get node
```

## Install go and kubebuilder

```bash
wget https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.14.4.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/bin:/usr/local/go/bin
```

```bash
os=$(go env GOOS)
arch=$(go env GOARCH)
curl -L https://go.kubebuilder.io/dl/2.3.1/${os}/${arch} | tar -xz -C /tmp/

sudo mv /tmp/kubebuilder_2.3.1_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
```


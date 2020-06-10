## Deploy a local k8s cluster with k3d

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | TAG=v1.7.0 bash
k3d create --workers 2
```{{execute}}

## Install kubebuilder

```bash
os=$(go env GOOS)
arch=$(go env GOARCH)
curl -L https://go.kubebuilder.io/dl/2.3.1/${os}/${arch} | tar -xz -C /tmp/

sudo mv /tmp/kubebuilder_2.3.1_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin

curl -s https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh  | bash
sudo mv kustomize /usr/local/bin
```{{execute}}

## Finally check the installation

Wait a couple of minutes and then you should see the deployment succeeding:

```bash
export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
kubectl get node
```{{execute}}
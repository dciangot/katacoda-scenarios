## Initialize a kubebuilder project

```bash
export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
kubectl get node
mkdir k8s-operator-tutorial
cd k8s-operator-tutorial
go mod init tutorial.kubebuilder.io/tutorial
kubebuilder init --domain tutorial.kubebuilder.io
```

```bash
mkdir k8s-operator-tutorial
cd k8s-operator-tutorial
go mod init tutorial.kubebuilder.io/tutorial
kubebuilder init --domain tutorial.kubebuilder.io
```
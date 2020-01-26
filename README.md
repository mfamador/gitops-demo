# Gitops workflow demo

## Create a k3d cluster with 2 workers
```
k3d create --publish 8080:80 --workers 2
```

## Install Helm

```
kubectl create serviceaccount -n kube-system tiller
kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

helm init --service-account tiller --tiller-namespace kube-system

helm list
```


## Install Flux

Add the fluxcd repo:

```
helm repo add fluxcd https://charts.fluxcd.io
```

### Install the HelmRelease CRD:

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/flux-helm-release-crd.yaml
```

### Install Flux:
```
export TILLER_NAMESPACE=kube-system
export FLUX_FORWARD_NAMESPACE=flux

helm install --name flux \
--set rbac.create=true \
--set git.url=git@github.com:mfamador/gitops-demo.git \
--set git.branch=master \
--set git.path="releases/development\,releases/common" \
--set git.pollInterval=120s \
--namespace flux fluxcd/flux 

fluxctl identity --k8s-fwd-ns flux
```

### Create a deploy key with write permissions on Githib

    Settings -> Deploy keys -> Add deploy key

### Install Flux Helm Operator
```
helm install --name helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--set workers=2 \
--namespace flux fluxcd/helm-operator 
```

### Test

```
export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"

curl -H "host:echo.domain.com" http://localhost:8080/
```

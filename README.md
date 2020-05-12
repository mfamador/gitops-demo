# Gitops workflow demo

## Create a k3d cluster with 2 workers
```
k3d create --publish 8080:80 --workers 2

export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
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

### Install the HelmRelease CRD

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/flux-helm-release-crd.yaml
```

### Install Flux:
```
kubectl create ns flux

helm install flux \
--set rbac.create=true \
--set git.url=git@github.com:mfamador/gitops-demo.git \
--set git.branch=add_kustomize \
--set git.path="releases/development\,releases/common" \
--set git.pollInterval=120s \
--namespace flux fluxcd/flux 

fluxctl identity --k8s-fwd-ns flux
```

### Create a deploy key on the github gitops repository with write permissions

    Settings -> Deploy keys -> Add deploy key -> check write permission checkbox

### Install Flux Helm Operator
```
helm install helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--set workers=2 \
--namespace flux fluxcd/helm-operator 
```


Configure ServiceMonitor for Flux and Helm Operator

```
helm upgrade --reuse-values flux --set prometheus.enabled=true,prometheus.serviceMonitor.create=true,"prometheus.serviceMonitor.additionalLabels.release=prometheus-operator" fluxcd/flux

helm upgrade --reuse-values helm-operator --set prometheus.enabled=true,prometheus.serviceMonitor.create=true,"prometheus.serviceMonitor.additionalLabels.release=prometheus-operator" fluxcd/helm-operator 
```

### Test

```
curl -H "host:echo.domain.com" http://localhost:8080/
```

# Gitops workflow demo

## Install Helm

```
kubectl create serviceaccount -n kube-system tiller

helm init --service-account tiller --tiller-namespace kube-system

helm list

```


## Install Flux

Add the fluxcd repo:

```
helm repo add fluxcd https://charts.fluxcd.io
```

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

create a deploy key

helm install --name helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--set workers=2 \
--namespace flux fluxcd/helm-operator 

```




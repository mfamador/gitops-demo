# Gitops workflow demo

## Create a local Kubernetes cluster, e.g. [k3d](https://k3d.io/#installation) with 3 worker nodes

    k3d cluster create -p 8080:80@loadbalancer --agents 3

If you need another worker
    
    k3d add-node

## Install Helm 3

MacOS
    
    brew install helm@3

[Other OS](https://helm.sh/docs/intro/install/)

## Install Flux


Add the fluxcd repo:

    helm repo add fluxcd https://charts.fluxcd.io

### Install the HelmRelease CRD

    kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/1.0.1/deploy/crds.yaml

### Install Flux on Staging cluster
```
kubectl create ns flux

helm install flux \
--set rbac.create=true \
--set git.url=git@github.com:mfamador/gitops-kustomize-demo.git \
--set git.branch=master \
--set git.path="staging" \
--set git.pollInterval=120s \
--set manifestGeneration=true \
--set syncGarbageCollection.enabled=true \
--namespace flux fluxcd/flux 

fluxctl identity --k8s-fwd-ns flux
```

### Create a deploy key on the github gitops repository with write permissions

    Settings -> Deploy keys -> Add deploy key -> check write permission checkbox

### Install Flux Helm Operator
```
helm upgrade -i helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--set workers=2 \
--set helm.versions=v3 \
--namespace flux fluxcd/helm-operator 
````

## Test

### Echo API
```
curl -H "host:echo.domain.com" http://localhost:8080/
```

### Grafana

Check ingress:
```
curl -H "host:grafana.domain.com" http://localhost:8080/
```

Setup port-forward to Grafana service to use it locally:
```
kubectl port-forward svc/prometheus-operator-grafana 8888:80 -n monitoring
```

Grafana UI:
```
open http://localhost:8888
```

Explore Loki:

![Loki](assets/loki.png "Loki")

```
open http://localhost:8888/explore?orgId=1&left=%5B%22now-1h%22,%22now%22,%22Loki%22,%7B%22expr%22:%22%7Bapp%3D%5C%22echo-api%5C%22%7D%22%7D,%7B%22mode%22:%22Logs%22%7D,%7B%22ui%22:%5Btrue,true,true,%22none%22%5D%7D%5D
```


## Find out outdated helm charts

```
helm plugin install https://github.com/fabmation-gmbh/helm-whatup

helm repo update

helm whatup --all-namespaces --ignore-repo # you'll need `--ignore-repo` because the plugin is not handling custom charts correctly
```

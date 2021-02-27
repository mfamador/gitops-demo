# GitOps workflow demo with Flux 2

[![test](https://github.com/mfamador/gitops-demo/workflows/test/badge.svg)](https://github.com/mfamador/gitops-demo/actions)
[![e2e](https://github.com/mfamador/gitops-demo/workflows/e2e/badge.svg)](https://github.com/mfamador/gitops-demo/actions)
[![license](https://img.shields.io/github/license/mfamador/gitops-demo.svg)](https://github.com/mfamador/gitops-demo/blob/main/LICENSE)

_For the original example see [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example)._

---

# Scenario

For this example we assume a scenario with two clusters: staging and production, both having multiple regions each.
The end goal is to leverage Flux and Kustomize to manage both clusters while minimizing duplicated declarations.

We will configure Flux to install, test and upgrade a demo app using
`HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and it will automatically
upgrade the Helm releases to their latest chart version based on semver ranges.

## Prerequisites

You will need a Kubernetes cluster version 1.16 or newer and kubectl version 1.18.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/) or
[k3d](https://k3d.io/#installation).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS and Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **environments/** dir contains Helm releases with a custom configuration per cluster
- **clusters** dir contains the Flux configuration per cluster

```
├── environments
│   ├── base
│   │     ├── infrastructure
│   │     ├── operations
│   │     └── services
│   ├── production
│   │     ├── eun
│   │     ├── euw
│   │     ├── usea
│   │     └── usew
│   └── staging
│         ├── eun
│         │    ├── infrastructure
│         │    ├── operations
│         │    └── services
│         └── euw
└── clusters
    ├── production
    │     ├── eun
    │     ├── euw
    │     ├── usea
    │     └── usew
    └── staging
          ├── eun
          └── euw
```

The environments' configuration is structured into:

- **environments/base/** dir contains common namespaces and Helm release definitions
- **environments/production/** dir contains the production Helm release values
- **environments/staging/** dir contains the staging values

```
./environments/

├── base
│     ├── infrastructure
│     ├── operations
│     └── services
│           ├── core
│           │     ├── kustomization.yam
│           │     ├── namespace.yam
│           │     └── podinfo.yam
│           └── data
├── production
│     ├── eun
│     ├── euw
│     ├── usea
│     └── usew
└── staging
      ├── eun
      │    ├── infrastructure
      │    ├── operations
      │    └── services
      │           ├── core
      │           │     ├── kustomization.yam
      │           │     └── podinfo-patch.yam
      │           ├── data
      │           └── kustomization.yaml
      └── euw
```

In **environments/base/services/core/podinfo/** dir we have a HelmRelease with common values for all clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: core
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 5m
  values:
    cache: redis-master.redis:6379
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
      path: "/*"
```

In **environments/staging/<region>/** dir we have a Kustomize patch with the staging specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - podinfo.staging,eun
```

Note that with ` version: ">=1.0.0-alpha"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest chart version including alpha, beta and pre-releases.

In **environments/production/<region>/** dir we have a Kustomize patch with the production specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - podinfo.production.eun
```

Note that with ` version: ">=1.0.0"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest stable chart version (alpha, beta and pre-releases will be ignored).

Infrastructure:

```
./environments/base/infrastructure/
├── nginx
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── redis
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
└── sources
    ├── bitnami.yaml
    ├── kustomization.yaml
    └── podinfo.yaml
```

In **infrastructure/sources/** dir we have the Helm repositories definitions:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: podinfo
spec:
  interval: 5m
  url: https://stefanprodan.github.io/podinfo
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bitnami
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
```

Note that with ` interval: 5m` we configure Flux to pull the Helm repository index every five minutes.
If the index contains a new chart version that matches a `HelmRelease` semver range, Flux will upgrade the release.

## Bootstrap staging and production

The clusters dir contains the Flux configuration:

```
./clusters/
├── production
│     ├── eun
│     ├── euw
│     ├── usea
│     └── usew
└── staging
      ├── eun
      │    ├── infrastructure.yaml
      │    ├── operations.yaml
      │    └── services.yaml
      └── euw
```

In **clusters/staging/** dir we have the Kustomization definitions:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: services
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-sytem
  path: ./environments/staging/eun/services
  prune: true
  validation: client
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./environments/staging/eun/infrastructure
```

Note that with `path: ./environments/staging/eun/services` we configure Flux to sync the staging Kustomize overlay and
with `dependsOn` we tell Flux to create the infrastructure items before deploying the services.

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging/eun
```

The bootstrap command commits the environments for the Flux components in `clusters/staging/eun/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being install on staging:

```console
$ watch flux get helmreleases --all-namespaces
NAMESPACE	NAME   	REVISION	SUSPENDED	READY	MESSAGE
nginx    	nginx  	5.6.14  	False    	True 	release reconciliation succeeded
podinfo  	podinfo	5.0.3   	False    	True 	release reconciliation succeeded
redis    	redis  	11.3.4  	False    	True 	release reconciliation succeeded
```

Verify that the demo app can be accessed via ingress:

```console
$ kubectl -n nginx port-forward svc/nginx-ingress-controller 8888:80 &

$ curl -H "Host: podinfo.staging.eun" http://localhost:8888
{
  "hostname": "podinfo-59489db7b5-lmwpn",
  "version": "5.0.3"
}
```

Bootstrap Flux on production by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production/eun
```

Watch the production reconciliation:

```console
$ watch flux get kustomizations
NAME          	REVISION                                        READY
apps          	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
flux-system   	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
infrastructure	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
```

## Encrypt Kubernetes secrets

In order to store secrets safely in a Git repository,
you can use Mozilla's SOPS CLI to encrypt Kubernetes secrets with OpenPGP or KMS.

Install [gnupg](https://www.gnupg.org/) and [sops](https://github.com/mozilla/sops):

```sh
brew install gnupg sops
```

Generate a GPG key for Flux without specifying a passphrase and retrieve the GPG key ID:

```console
$ gpg --full-generate-key
Email address: fluxcdbot@users.noreply.github.com

$ gpg --list-secret-keys fluxcdbot@users.noreply.github.com
sec   rsa3072 2020-09-06 [SC]
      1F3D1CED2F865F5E59CA564553241F147E7C5FA4
```

Create a Kubernetes secret on your clusters with the private key:

```sh
gpg --export-secret-keys \
--armor 1F3D1CED2F865F5E59CA564553241F147E7C5FA4 |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin
```

Generate a Kubernetes secret manifest and encrypt the secret's data field with sops:

```sh
kubectl -n redis create secret generic redis-auth \
--from-literal=password=change-me \
--dry-run=client \
-o yaml > environments/staging/eun/infrastructure/redis/redis-auth.yaml

sops --encrypt \
--pgp=1F3D1CED2F865F5E59CA564553241F147E7C5FA4 \
--encrypted-regex '^(data|stringData)$' \
--in-place environments/staging/eun/infrastructure/redis/redis-auth.yaml
```

Add the secret to `environments/staging/eun/infrastructure/redis/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: redis
resources:
  - namespace.yaml
  - release.yaml
  - redis-auth.yaml
```

Enable decryption on your clusters by editing the `infrastructure.yaml` files:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  # content omitted for brevity
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

Export the public key so anyone with access to the repository can encrypt secrets but not decrypt them:

```sh
gpg --export -a fluxcdbot@users.noreply.github.com > public.key
```

Push the changes to main branch:

```sh
git add -A && git commit -m "add encrypted secret" && git push
```

Verify that the secret has been created in the `redis` namespace on both clusters:

```sh
kubectl --context staging -n redis get secrets
kubectl --context production -n redis get secrets
```

You can use Kubernetes secrets to provide values for your Helm releases:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
spec:
  # content omitted for brevity
  values:
    usePassword: true
  valuesFrom:
  - kind: Secret
    name: redis-auth
    valuesKey: password
    targetPath: password
```

Find out more about Helm releases values overrides in the
[docs](https://toolkit.fluxcd.io/components/helm/helmreleases/#values-overrides).


## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p clusters/dev
```

Copy the sync environments from staging:

```sh
cp clusters/staging/eun/infrastructure.yaml clusters/dev
cp clusters/staging/eun/operations.yaml clusters/dev
cp clusters/staging/eun/services.yaml clusters/dev
```

You could create a dev overlay inside `services` and `operations`, make sure
to change the `spec.path` inside `clusters/dev/eun/services.yaml` to `path: ./environments/dev/eun`.

Push the changes to main branch:

```sh
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev/eun
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.

This repository contains the following GitHub CI workflows:

* the [test](./.github/workflows/test.yaml) workflow validates the Kubernetes manifests and Kustomize overlays with kubeval
* the [e2e](./.github/workflows/e2e.yaml) workflow starts a Kubernetes cluster in CI and tests the staging setup by running Flux in Kubernetes Kind

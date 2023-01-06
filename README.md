# Kubernetes Operator for a simple helm cart based application
A sample operator-sdk created kubernetes operator containing a helm chart


## Install operator-sdk cli (m1 mac)
```
arch -arm64 brew install operator-sdk
```

## Doc reference for operator sdk quickstart
Ref: [quickstart](https://sdk.operatorframework.io/docs/building-operators/helm/quickstart/)

## create directory and initialise
```
mkdir hello-operator
cd hello-operator
```

doc ref: [plugins](https://sdk.operatorframework.io/docs/contribution-guidelines/plugins/)
note: domain used as image prefix
```
operator-sdk init --domain grahamh --plugins="helm.sdk.operatorframework.io/v1"
```

## copy the helm chart into place
```
cp -a <my helm chart source directory> helm-charts/
```

# create watches.yaml entry
```
operator-sdk create api --group hello --kind Hello --helm-chart helm-charts/app-chart

Writing kustomize manifests for you to edit...
Created helm-charts/app-chart

cat watches.yaml
# Use the 'create api' subcommand to add watches to this file.
- group: hello.grahamh
  version: v1alpha1
  kind: Hello
  chart: helm-charts/app-chart
#+kubebuilder:scaffold:watch


## build operator docker image
```
make docker-build docker-push IMG="grahamh/hello-operator:1.0"
```

## investigate image
```
docker run --rm -it --entrypoint=/bin/bash grahamh/hello-operator:1.0

bash-4.4$ id
uid=1001(helm) gid=0(root) groups=0(root)

bash-4.4$ pwd
/opt/helm

bash-4.4$ ls -la
total 20
drwxr-xr-x 1 root root 4096 Jan  5 11:03 .
drwxr-xr-x 1 root root 4096 Dec  9 23:40 ..
drwxr-xr-x 3 root root 4096 Jan  5 11:03 helm-charts
-rw-r--r-- 1 root root  187 Jan  5 11:02 watches.yaml

bash-4.4$ ls -la /usr/local/bin
total 68312
drwxr-xr-x 1 root root     4096 Dec  9 23:48 .
drwxr-xr-x 1 root root     4096 Nov 30 17:53 ..
-rwxr-xr-x 1 root root 69942499 Dec  9 23:48 helm-operator


## Prepare deployment artifacts

### Install Kustomize CLI (m1 mac)
```
arch -arm64 brew install kustomize
```

### Generate crd.yaml
```
mkdir deploy-yaml
kustomize build config/crd > deploy-yaml/hello-crd.yaml
```

### Create and build the operator bundle
```
make bundle IMG="grahamh/hello-operator:1.0"
make bundle-build IMG="grahamh/hello-operator:1.0"
```

WIP


## Deploy the operator on linux host running kubernetes..

note: BYO Kubernetes (tested with kURL)

### Install operator-sdk on Linux host
```
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')

export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.26.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
chmod +x operator-sdk_${OS}_${ARCH} && sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
```

### Install kustomize on Linux host
```
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')

curl -LO https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v4.5.7/kustomize_v4.5.7_${OS}_${ARCH}.tar.gz
tar -xf kustomize_v4.5.7_${OS}_${ARCH}.tar.gz
chmod +x kustomize && sudo mv kustomize /usr/local/bin/
rm kustomize_v4.5.7_${OS}_${ARCH}.tar.gz
```

### Load the Operator framework CRD
```
make install IMG="grahamh/hello-operator:1.0"

/usr/local/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/hellos.hello.grahamh
```

### Deploy using Makefile / kustomize
```
make deploy IMG="grahamh/hello-operator:1.0"

cd config/manager && kustomize edit set image controller=controller:latest

kustomize build config/default | kubectl apply -f -

namespace/hello-operator-system created
customresourcedefinition.apiextensions.k8s.io/hellos.hello.grahamh unchanged
serviceaccount/hello-operator-controller-manager created
role.rbac.authorization.k8s.io/hello-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/hello-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/hello-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/hello-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/hello-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/hello-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/hello-operator-proxy-rolebinding created
service/hello-operator-controller-manager-metrics-service created
deployment.apps/hello-operator-controller-manager created
```



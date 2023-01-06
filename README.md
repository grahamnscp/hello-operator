# Kubernetes Operator for a simple helm cart based application
A sample operator-sdk created kubernetes operator containing a helm chart


## Install operator-sdk cli (m1 mac)
```
arch -arm64 brew install operator-sdk
```

## Doc reference for operator sdk quickstart
Ref: [quickstart](https://sdk.operatorframework.io/docs/building-operators/helm/quickstart/)

## Create working directory and initialise
```
mkdir hello-operator
cd hello-operator
```

doc ref: [plugins](https://sdk.operatorframework.io/docs/contribution-guidelines/plugins/)
note: domain varaiable used as image registry name
```
operator-sdk init --domain grahamh --plugins="helm.sdk.operatorframework.io/v1"
```

## Copy the helm chart into place
```
cp -a <my helm chart source directory> helm-charts/
```

## Create watches.yaml entry
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
```

## build operator docker image
```
###make docker-build docker-push IMG="grahamh/hello-operator:1.0"

#for multi-arch image build directly:
docker buildx build --platform linux/amd64,linux/arm64 --push -t grahamh/hello-operator:1.0 .
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
```


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
make bundle IMG="grahamh/hello-operator-bundle:1.0"

####make bundle-build IMG="grahamh/hello-operator-bundle:1.0"
docker buildx build -f bundle.Dockerfile --platform linux/amd64,linux/arm64 --push -t grahamh/hello-operator-bundle:1.0 .
```


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

### Deploy Operator using Makefile / kustomize
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

### Check the Operator running:
```
kubectl get crd | grep hello
hellos.hello.grahamh                          2023-01-06T14:32:09Z

kubectl get ns | grep hello
hello-operator-system   Active   51m

kubectl -n hello-operator-system get all
NAME                                                     READY   STATUS                       RESTARTS   AGE
pod/hello-operator-controller-manager-565b4ddf56-wknxb   2/2     Running                      0          48m

NAME                                                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/hello-operator-controller-manager-metrics-service   ClusterIP   10.96.0.159   <none>        8443/TCP   52m

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-operator-controller-manager   1/1     1            1           52m

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-operator-controller-manager-565b4ddf56   1         1         1       52m

kubectl -n hello-operator-system logs pod/hello-operator-controller-manager-565b4ddf56-wknxb
{"level":"info","ts":1673015803.18843,"logger":"cmd","msg":"Version","Go Version":"go1.19.4","GOOS":"linux","GOARCH":"amd64","helm-operator":"v1.26.0","commit":"cbeec475e4612e19f1047ff7014342afe93f60d2"}
{"level":"info","ts":1673015803.1893198,"logger":"cmd","msg":"Watch namespaces not configured by environment variable WATCH_NAMESPACE or file. Watching all namespaces.","Namespace":""}
{"level":"info","ts":1673015803.3930433,"logger":"controller-runtime.metrics","msg":"Metrics server is starting to listen","addr":"127.0.0.1:8080"}
{"level":"info","ts":1673015803.3942614,"logger":"helm.controller","msg":"Watching resource","apiVersion":"hello.grahamh/v1alpha1","kind":"Hello","namespace":"","reconcilePeriod":"1m0s"}
I0106 14:36:43.394799       1 leaderelection.go:248] attempting to acquire leader lease hello-operator-system/hello-operator...
{"level":"info","ts":1673015803.3948064,"msg":"Starting server","path":"/metrics","kind":"metrics","addr":"127.0.0.1:8080"}
{"level":"info","ts":1673015803.394807,"msg":"Starting server","kind":"health probe","addr":"[::]:8081"}
I0106 14:36:43.406403       1 leaderelection.go:258] successfully acquired lease hello-operator-system/hello-operator
{"level":"info","ts":1673015803.406685,"msg":"Starting EventSource","controller":"hello-controller","source":"kind source: *unstructured.Unstructured"}
{"level":"info","ts":1673015803.4067383,"msg":"Starting Controller","controller":"hello-controller"}
{"level":"info","ts":1673015803.5078287,"msg":"Starting workers","controller":"hello-controller","worker count":8}
E0106 15:07:17.610823       1 memcache.go:206] couldn't get resource list for custom.metrics.k8s.io/v1beta1: Got empty response for: custom.metrics.k8s.io/v1beta1
{"level":"info","ts":1673017638.297573,"msg":"Starting EventSource","controller":"hello-controller","source":"kind source: *unstructured.Unstructured"}
{"level":"info","ts":1673017638.2976582,"logger":"helm.controller","msg":"Watching dependent resource","ownerApiVersion":"hello.grahamh/v1alpha1","ownerKind":"Hello","apiVersion":"v1","kind":"ServiceAccount"}
{"level":"info","ts":1673017638.2979376,"msg":"Starting EventSource","controller":"hello-controller","source":"kind source: *unstructured.Unstructured"}
{"level":"info","ts":1673017638.2979724,"logger":"helm.controller","msg":"Watching dependent resource","ownerApiVersion":"hello.grahamh/v1alpha1","ownerKind":"Hello","apiVersion":"v1","kind":"Service"}
{"level":"info","ts":1673017638.298403,"msg":"Starting EventSource","controller":"hello-controller","source":"kind source: *unstructured.Unstructured"}
{"level":"info","ts":1673017638.29843,"logger":"helm.controller","msg":"Watching dependent resource","ownerApiVersion":"hello.grahamh/v1alpha1","ownerKind":"Hello","apiVersion":"apps/v1","kind":"Deployment"}
```

## Deploy the App using the Operator
```
kubectl apply -f  config/samples/hello_v1alpha1_hello.yaml
```

### Check:
```
kubectl get all
NAME                                         READY   STATUS    RESTARTS   AGE
pod/hello-sample-app-chart-7cfc6c64d-lh858   1/1     Running   0          10s
pod/hello-sample-app-chart-7cfc6c64d-xh2fn   1/1     Running   0          10s

NAME                             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
service/hello-sample-app-chart   NodePort    10.96.3.167   <none>        8080:8082/TCP   10s
service/kubernetes               ClusterIP   10.96.0.1     <none>        443/TCP         27h

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-sample-app-chart   2/2     2            2           10s

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-sample-app-chart-7cfc6c64d   2         2         2       10s
```

### Operator conroller logs for app deployment:
```
kubectl -n hello-operator-system logs pod/hello-operator-controller-manager-565b4ddf56-wknxb
<snip>
{"level":"info","ts":1673017638.298441,"logger":"helm.controller","msg":"Installed release","namespace":"default","name":"hello-sample","apiVersion":"hello.grahamh/v1alpha1","kind":"Hello","release":"hello-sample"}
E0106 15:07:18.454813       1 memcache.go:206] couldn't get resource list for custom.metrics.k8s.io/v1beta1: Got empty response for: custom.metrics.k8s.io/v1beta1
{"level":"info","ts":1673017639.0278964,"logger":"helm.controller","msg":"Reconciled release","namespace":"default","name":"hello-sample","apiVersion":"hello.grahamh/v1alpha1","kind":"Hello","release":"hello-sample"}
<snip>
```

### Test App:
```
LOCALIP=`ip a | grep inet | grep eth0 | awk '{print $2}' | awk -F"/" '{print $1}'`
curl ${LOCALIP}:8082

 _          _ _                        _ _           _           _
| |__   ___| | | ___    _ __ ___ _ __ | (_) ___ __ _| |_ ___  __| |
| '_ \ / _ \ | |/ _ \  | '__/ _ \ '_ \| | |/ __/ _` | __/ _ \/ _` |
| | | |  __/ | | (_) | | | |  __/ |_) | | | (_| (_| | ||  __/ (_| |
|_| |_|\___|_|_|\___/  |_|  \___| .__/|_|_|\___\__,_|\__\___|\__,_|
                                |_|

[01-06-2023 15:12:24.60] Container hostname: hello-sample-app-chart-7cfc6c64d-xh2fn
```

Although the app was non deployed via Replicated! but packaged as an operator..


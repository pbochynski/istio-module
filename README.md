# istio
Istio module for Kyma created for istio operator. 

## How it was created

First, we need helm chart that installs the operator. It is a temporary solution as Kyma lifecycle-manager is going to support plain kubernetes manifests soon. 

```
mkdir charts
helm create charts/istio
rm -r charts/istio/templates
mkdir charts/istio/templates
istioctl operator dump --operatorNamespace=kcp-system --watchedNamespaces=kcp-system > charts/istio/templates/deployment.yaml
```

Create file `charts/istio/crds/istio-crd.yaml` and move there istiooperators.install.istio.io CRD from `charts/istio/templates/deployment.yaml`


Create config.yaml:
```sh
cat <<EOF > config.yaml
# Samples Config
configs:
EOF
```

Create default istio-cr.yaml:
```sh
cat <<EOF >istio-operator-cr.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istiocontrolplane
spec:
  profile: demo
EOF
```

Download kyma cli:
```
curl -Lo kyma https://storage.googleapis.com/kyma-cli-unstable/kyma-darwin # kyma-linux, kyma-linux-arm, kyma.exe, or kyma-arm.exe
chmod +x kyma
```
Login to your docker registry:
```
docker login ghcr.io
```

Create module template (replace pbochynski with your github handle):
```
kyma alpha create module -n kyma-project.io/istio --version 0.0.2 --registry ghcr.io/pbochynski/istio-module -r istio:helm-chart@./charts/istio -r config:yaml@./config.yaml --default-cr ./istio-operator-cr.yaml -o istio-module-template.yaml
```

Go to your [github packages](https://github.com/pbochynski?tab=packages) and make the istio-module public.

# Install istio module

Install lifcycle-manager:
```
kyma alpha deploy --ci
```

Create module template:
```
kubectl apply -f istio-module-template.yaml
```

Give more permissions to lifecycle-manager (to install any module):
```
cat <<EOF | kubectl apply -f - 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: module-manager-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: module-manager-manager
  namespace: kcp-system
EOF
```

Add istio to the module list in Kyma resource:

```
kubectl patch kyma default-kyma -n kcp-system --type='json' \
-p='[{"op": "add", "path": "/spec/modules", "value": [{"name": "istio","channel":"regular" }] }]'
```

Wait 5 seconds and check if IstioOperator resource was created:
```
kubectl get istiooperator -A
```

Check if istio is installed:
```
kubectl get pods -A
```
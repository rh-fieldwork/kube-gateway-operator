
### Disconnected clusters

If the system is in a disconnected environment, without access to the public image repository, edit the yaml examples to use internally provided container images.

``` bash
# Edit the operator image in operator-controller-manager.yaml
curl https://raw.githubusercontent.com/rh-fieldwork/kube-gateway-operator/main/deploy/kube-gateway-operator.yaml \
    -o kube-gateway-operator.yaml

vim kube-gateway-operator.yaml
kubectl create -f kube-gateway-operator.yaml
```

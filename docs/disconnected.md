
### Disconnected Clusters

If the system is in a disconnected environment, without access to the public image repository, edit the yaml examples to use internally provided container images.

### Build the Image

Create the operator image.

``` bash
make deploy-dir
```

The default image URL in the file is `quay.io/rh-fieldwork/kube-gateway-operator`. Replace all instances of this URL with the image URL you want to use. Below is an example of how to do this in vim, but you can do this using whatever method you like.

```
vim kube-gateway-operator.yaml
# Run the following command within vim
:%s/quay.io\/rh-fieldwork/example-repo-name\/repo-name/g
```

Deploy the image to the cluster. 

Note: You must be logged into the cluster as an admin in order to successfully run the following command.

``` bash
kubectl create -f kube-gateway-operator.yaml
```

Confirm the deployment.

```bash
oc get pods -n kube-gateway-operator-system
```

If the deployment was successfull, there will be a pod named kube-gateway-operator-controller-manager-\<random suffix\> with a status of `running`.

### Deploy the gate-server

Create a new yaml file using the example below and edit the route, image, and webAppImage fields.

```bash
vim gate-server-sample.yaml
```
Example gate-server:
```yaml
apiVersion: ocgate.rh-fieldwork.com/v1beta1
kind: GateServer
metadata:
  name: gateserver-sample
  namespace: kube-gateway-operator-system
spec:
  route: kube-gateway-proxy.apps.ostest.test.metalkube.org
  # serviceAccount fields are used to create a service account for the oc gate proxy.
  # The proxy will run using this service account. It will only be able to proxy
  # requests that are available to this service account. Make sure to allow the 
  # proxy to access all k8s resources that the web application will consume.
  serviceAccountVerbs:
    - "get"
  serviceAccountAPIGroups:
    - "subresources.kubevirt.io"
  serviceAccountResources:
    - "virtualmachineinstances"
    - "virtualmachineinstances/vnc"
  # generateSecret is used to automatically create a secret holding the asymmetrical
  # keys needed to sign and authenticate the JWT tokens.
  generateSecret: true 
  # passThrough is used to pass the request token directly to the k8s API server without
  # authenticating and replaces it with the service account access token of the proxy
  passThrough: false
  # the proxy server container image
  image: 'quay.io/philliprhodes/kube-gateway'
  # webAppImage is used to customize the static files of your web app.
  # This example will install the noVNC web application that consumes
  # websockets streaming VNC data.
  webAppImage: 'quay.io/philliprhodes/kube-gateway-web-app-novnc'
```

Create the server.

```bash
oc create -f gate-server-sample.yaml
```

Confirm the server was created successfully.

```bash
oc get pods -n kube-gateway-operator-system
```

There should be a pod name gateserver-sample with a status of `running`.

### Create a Token


Create a new yaml file using the example below.

```bash
vim gate-token.yaml
```

```yaml
apiVersion: ocgate.rh-fieldwork.com/v1beta1
kind: GateToken
metadata:
  name: gatetoken-sample
  namespace: kube-gateway-operator-system
spec:
  namespace: "yzamir"
  resourceNames:
    - fedora-diverse-eagle
```

Edit the namespace and resourceNames fields in the spec section to match the resource(s) for which you want this token to work and change the metadata name to a unique name.

Create the token.

```bash
oc create -f gate-token.yaml
```

Confirm the token was created successfully.
Note: Make sure to replace 'gatetoken-sample' with the unique name you used in the yaml above.

```bash
oc describe gatetoken.ocgate.rh-fieldwork.com/gatetoken-sample -n kube-gateway-operator-system
```

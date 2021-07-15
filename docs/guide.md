## Build and Push an Image

### Log into Your Image Repository

For example, if you are using quay.io:

`podman login quay.io`

Enter your username and password when prompted. Once you are logged in, you can make and push the image.

### Make and Push an Image

Create your repository.

The following `make` commands will build and push the image to the repository specified by the IMG variable.
Make sure to update the IMG variable with the URL to the repository in which you want the image to be created, for example:

`IMG=quay.io/rh-fieldwork/kube-gateway-operator`

```bash
make
IMG=quay.io/<username>/kube-gateway-operator make podman-build
IMG=quay.io/<username>/kube-gateway-operator make podman-push
```

### Make and Push a noVNC Image

```bash
make novnc
IMG_WEB_APP_NOVNC=quay.io/<username>/<repository name> make image-web-app-novnc
```

## Create the Deployment Manifest

The image URL for the command below must include the image's digest hash, which is retrieved using the podman CLI command
inside the parentheses. Make sure to replace the <username> placeholder.

`IMG=$(podman image inspect quay.io/<username>/kube-gateway-operator | jq '.[0].RepoDigests[0]') make deploy-dir`

## operator-sdk

```bash
operator-sdk init --domain rh-fieldwork.com --repo github.com/rh-fieldwork/kube-gateway-operator
operator-sdk create api --group oc-gate --version v1alpha1 \
  --kind Token --resource --controller

... create the app

... compile

... build bundle and push it

operator-sdk run bundle quay.io/rh-fieldwork/kube-gateway-operator-bundle:v0.0.1 \
  --index-image quay.io/operator-framework/upstream-opm-builder:latest

... run tests

operator-sdk cleanup kube-gateway-operator
```

## Disconnected Clusters

If the system is in a disconnected environment, without access to the public image repository, edit the yaml examples to use internally provided container images.

### Build the Image

Create the operator image.

```bash
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
oc create -f kube-gateway-operator.yaml
```

Confirm the deployment.

```bash
oc get pods -n kube-gateway-operator-system
```

If the pod isn't running or is stuck at '1/2 running' you can debug with the following command.

```bash
oc logs kube-gateway-operator-controller-manager-<random suffix> -n kube-gateway-operator-system -c manager
```

If the deployment was successful, there will be a pod named kube-gateway-operator-controller-manager-\<random suffix\> with a status of `running`.

### Deploy the gate-server

Create a new yaml file using the example below and edit the `route`, `image`, and `webAppImage` fields.

```bash
vim gate-server-sample.yaml
```
Example gate-server:
```yaml
apiVersion: ocgate.rh-fieldwork.com/v1beta1
kind: GateServer
metadata:
  name: gateserver-sample
  namespace: kube-gateway-operator
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

Create the namespace to which we will deploy the server.

```bash
oc create namespace kube-gateway-operator
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

### Obtain the Server Route

```bash
oc get routes -n kube-gateway-operator
```

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
  namespace: "default"
  resourceNames:
    - testvm
```

Replace the `namespace` and `resourceNames` values in the spec section with the resource(s) for which you want this token to work and change the `name` field in the metadata section to a unique name.

Create the token.

```bash
oc create -f gate-token.yaml
```

Confirm the token was created successfully.
Note: Make sure to replace 'gatetoken-sample' with the unique name you used in the yaml above.

```bash
oc describe gatetoken.ocgate.rh-fieldwork.com/gatetoken-sample -n kube-gateway-operator
```

### Create Resource URL

Set the following environment variables, replacing the `vm`, `ns`, and `proxyurl` values.

```bash
vm=testvm
ns=default
apigroup=subresources.kubevirt.io
resource=virtualmachineinstances
resourcepath=k8s/apis/${apigroup}/v1/namespaces/${ns}/${resource}/${vm}/vnc
proxyurl=https://kube-gateway.apps-crc.testing
jwt=$(oc get gatetoken.ocgate.rh-fieldwork.com/gatetoken-sample -n kube-gateway-operator -o jsonpath={.status.token})
```

Now we will create the URL using the environment variables.

```bash
echo "${proxyurl}/auth/token?token=${jwt}\&then=/noVNC/vnc_lite.html?path=${resourcepath}"
```

You can now use this URL in your browser to access the VNC console.



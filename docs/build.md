# Build

## Log in to your image repository

For example, if you are using quay.io:

`podman login quay.io`

Enter your username and password when prompted. Once you are logged in, you can make and push the image.

## Make and push an image

Create your repository.

The following `make` commands will build and push the image to the repository specified by the IMG variable.
Make sure to update the IMG variable with the URL to the repository in which you want the image to be created, for example:

`IMG=quay.io/rh-fieldwork/kube-gateway-operator`

```bash
make 
IMG=quay.io/<username>/kube-gateway-operator make podman-build 
IMG=quay.io/<username>/kube-gateway-operator make podman-push
```

## Create the deployment manifest

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
